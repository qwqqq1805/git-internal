# 实验 2：Pack Encode 局部优化任务书

## 1. 题目意思拆解

这个实验不是要求完整重写 Git 的 pack 编码器，也不是要求全面追求极致性能。它要求在 `git-internal-0.7.3` 项目中，围绕 Pack Encode 流程选择一个“局部问题”完成理解、优化或重构，并能说明修改的原因和正确性。

题目中的 Pack Encode 指 Git 把多个对象写成 pack 文件的过程。当前项目中主要涉及：

- 对象输入：从异步 channel 中接收 `MetaAttached<Entry, EntryMeta>`。
- 对象排序：启用 delta 时，按对象类型分桶，再用 `magic_sort` 排序。
- 对象头编码：`encode_header`、`encode_offset`、`encode_one_object`。
- 对象压缩：`encode_one_object` 中使用 zlib 压缩对象数据。
- delta 选择：`try_as_offset_delta` 使用滑动窗口寻找合适 base。
- pack hash：写出过程中更新 `inner_hash`，最后把 hash 写到 pack trailer。
- idx 生成：pack 完成后使用 `IdxBuilder` 生成 `.idx` 文件。

本实验的核心不是“写出一个新 pack encoder”，而是回答并实践这些问题：

- Encode 的输入是什么？
- 输出数据如何写出？
- 哪些地方会分配 `Vec`？
- 哪些地方存在不必要的 `clone()`？
- `window_size` 在 delta 选择中起什么作用？
- 选择一个局部点做优化，并验证没有破坏 pack 格式。

## 2. 本次选择的局部问题

本次选择的局部优化主题是：

> 减少 Pack Encode 中不必要的内存复制，并修正 `window_size` 参数在 delta 路径中未真正生效的问题。

调研时发现了四个适合局部处理的问题：

1. `PackEncoder::inner_encode` 中调用 `try_as_offset_delta` 时把窗口大小硬编码为 `10`，没有使用用户传入的 `self.window_size`。
2. `PackEncoder::try_as_offset_delta` 中 `res.push((obj_data.clone(), ...))` 会复制完整的已编码对象数据。
3. `PackEncoder::parallel_encode` 和 delta 写出路径中，已有 `Vec<u8>` 仍通过 `&[u8] -> to_vec()` 再发送，产生额外复制。
4. `PackEncoder::generate_idx_file` 中 `self.idx_entries.clone()` 会复制整个索引数组。

这些问题都在 `src/internal/pack/encode.rs` 内部，符合“选择一个局部问题进行优化或重构”的要求。

## 3. 任务拆解

### 任务 1：阅读入口示例

阅读 `examples/encode_pack.rs`，理解：

- 示例如何构造 Blob；
- 如何转换成 `Entry`；
- 如何包装成 `MetaAttached<Entry, EntryMeta>`；
- 如何通过 channel 发送给 encoder；
- `window_size = 10` 的含义。

结论：

- Pack Encode 的输入是 `mpsc::Receiver<MetaAttached<Entry, EntryMeta>>`。
- `window_size == 0` 表示不启用 delta。
- `window_size != 0` 表示启用 delta 窗口。

### 任务 2：阅读 PackEncoder 主流程

阅读 `src/internal/pack/encode.rs`：

- `PackEncoder::encode`
- `PackEncoder::inner_encode`
- `PackEncoder::try_as_offset_delta`
- `PackEncoder::parallel_encode`

结论：

- `encode` 是路由函数。
- `window_size == 0` 走 `parallel_encode`。
- `window_size != 0` 走 `inner_encode`。
- `inner_encode` 会收集全部对象、分桶、排序、并行做 delta 选择，再写出。
- `parallel_encode` 不做 delta，按 batch 并行 zlib 编码。

### 任务 3：找出 Vec 分配点

重点检查这些地方：

- `encode_header` 返回 `Vec<u8>`。
- `encode_offset` 返回 `Vec<u8>`。
- `encode_one_object` 创建 `encoded_data`、`header_data`、zlib 输出 `Vec<u8>`。
- `inner_encode` 中的 `commits`、`trees`、`blobs`、`tags`。
- `try_as_offset_delta` 中的 `window`、`res`、`candidates`、delta 输出。
- `parallel_encode` 中的 `batch_entries`、`batch_result`。
- `generate_idx_file` 中的 `idx_entries`。

### 任务 4：找出不必要 clone

重点检查这些地方：

- `head.clone()`：只有 12 字节，成本很小。
- `entry.clone()`：复制 `Entry.data: Vec<u8>`，成本较高，但目前为了保留窗口 base 的原始数据，语义上仍然需要。
- `obj_data.clone()`：复制完整编码后的对象数据，可以直接删除。
- `idx_entries.clone()`：可以用 `Option::take()` 消费，避免整体复制。
- `write_all_and_update(&data)` 内部 `data.to_vec()`：调用方本来已经有 `Vec<u8>`，可以改为直接 move。

### 任务 5：实施局部优化

在 `src/internal/pack/encode.rs` 中完成以下改动：

1. `inner_encode`：使用 `self.window_size` 作为真实 delta 窗口大小。
2. `inner_encode`：消费 `all_res`，避免 `IndexEntry` clone。
3. `try_as_offset_delta`：删除 `obj_data.clone()`。
4. `parallel_encode`：消费 batch 编码结果，把 `Vec<u8>` 直接写出。
5. `write_all_and_update`：改为 `write_vec_and_update`，接收 `Vec<u8>` 并直接发送。
6. `generate_idx_file`：使用 `self.idx_entries.take()`，避免 clone 整个 `Vec<IndexEntry>`。

### 任务 6：验证正确性

验证分三层：

1. 编译验证：`cargo test --no-run test_pack_encoder`。
2. 格式验证：`cargo fmt --check`。
3. 行为验证：运行 PackEncoder 相关测试，确认 pack 可编码、可解码、`.idx` 可生成。

## 4. 修改了哪些文件和函数

### 修改文件 1：`src/internal/pack/encode.rs`

修改函数：

- `PackEncoder::inner_encode`
- `PackEncoder::try_as_offset_delta`
- `PackEncoder::parallel_encode`
- `PackEncoder::write_vec_and_update`
- `PackEncoder::generate_idx_file`

具体修改如下。

#### 4.1 PackEncoder::inner_encode

修改前，delta 窗口大小被硬编码为 `10`：

```rust
Self::try_as_offset_delta(entries, 10, enable_zstdelta)
```

修改后，使用 encoder 上的真实配置：

```rust
let window_size = self.window_size;
Self::try_as_offset_delta(entries, window_size, enable_zstdelta)
```

同时，写出 delta 编码结果时，原来遍历 `&mut all_res`，因此需要 clone `IndexEntry`：

```rust
idx_entries.push(data.1.clone());
```

修改后改为消费 `all_res`：

```rust
for res in all_res {
    for (data, mut idx_entry) in res {
        idx_entry.offset = self.inner_offset as u64;
        self.write_vec_and_update(data).await;
        idx_entries.push(idx_entry);
    }
}
```

#### 4.2 PackEncoder::try_as_offset_delta

修改前：

```rust
res.push((obj_data.clone(), IndexEntry::new(entry, 0)));
current_offset += obj_data.len();
```

问题：

- `obj_data` 是 `encode_one_object` 刚生成的 `Vec<u8>`。
- 这里 clone 会复制完整的编码对象数据。
- 之后原始 `obj_data` 只用于取长度，因此 clone 不必要。

修改后：

```rust
let obj_len = obj_data.len();
res.push((obj_data, IndexEntry::new(entry, 0)));
current_offset += obj_len;
```

这样先保存长度，再把 `obj_data` move 进结果数组。

#### 4.3 PackEncoder::parallel_encode

修改前：

```rust
let mut obj_data = obj_data?;
obj_data.1.offset = self.inner_offset as u64;
self.write_all_and_update(&obj_data.0).await;
idx_entries.push(obj_data.1);
```

问题：

- `obj_data.0` 本身是 `Vec<u8>`。
- `write_all_and_update(&obj_data.0)` 内部会 `to_vec()` 再复制一份发送。

修改后：

```rust
let (data, mut idx_entry) = obj_data?;
idx_entry.offset = self.inner_offset as u64;
self.write_vec_and_update(data).await;
idx_entries.push(idx_entry);
```

这样直接消费 `Vec<u8>`，避免额外复制。

#### 4.4 PackEncoder::write_vec_and_update

修改前函数是：

```rust
async fn write_all_and_update(&mut self, data: &[u8]) {
    self.inner_hash.update(data);
    self.inner_offset += data.len();
    self.send_data(data.to_vec()).await;
}
```

修改后函数是：

```rust
async fn write_vec_and_update(&mut self, data: Vec<u8>) {
    self.inner_hash.update(&data);
    self.inner_offset += data.len();
    self.send_data(data).await;
}
```

修改原因：

- channel 本来需要发送 `Vec<u8>`。
- 调用方已经拥有 `Vec<u8>`。
- 直接 move 可以减少一次完整复制。

#### 4.5 PackEncoder::generate_idx_file

修改前：

```rust
let idx_entries = self.idx_entries.clone().ok_or(...)?;
```

修改后：

```rust
let idx_entries = self.idx_entries.take().ok_or(...)?;
```

修改原因：

- `encode_idx_file` 是 pack 编码完成后的最终消费步骤。
- 没有必要保留一份 `idx_entries`。
- `take()` 可以把 `Option<Vec<IndexEntry>>` 中的 Vec 移出，避免复制。

### 新增文件 1：`pack_encode_local_optimization_report.md`

用途：

- 记录 Pack Encode 调研结果。
- 回答题目要求的输入、输出、Vec 分配、clone、`window_size` 作用。
- 记录可选优化点。

### 新增文件 2：`pack_encode_experiment_task_book.md`

用途：

- 拆解题目含义。
- 记录任务实施步骤。
- 记录本次实际修改、验证方式、测试结果和简单性能分析。

## 5. 为什么这样修改

### 5.1 修正 window_size 的语义

题目明确要求理解 `window_size` 在 delta 选择中的作用。代码接口也表达了：

- `window_size == 0`：禁用 delta；
- `window_size != 0`：启用 delta，并控制滑动窗口大小。

但是原代码只在 `encode` 中用它决定是否进入 delta 路径，进入 `inner_encode` 后真实 delta 窗口固定为 `10`。

这会导致：

- 用户设置 `window_size = 1`，实际仍搜索最近 10 个 base；
- 用户设置 `window_size = 100`，实际仍只搜索最近 10 个 base；
- 只有 `0` 和非 `0` 的区别生效，具体窗口大小不生效。

修正后，`window_size` 真正控制 `VecDeque` 滑动窗口容量和 base 搜索范围。

### 5.2 删除 obj_data.clone

`obj_data.clone()` 是明确的不必要复制。因为：

- `obj_data` 之后只需要长度；
- 可以先保存 `obj_len`；
- 然后直接把 `obj_data` move 到 `res`。

该修改不会改变编码字节内容，只减少一次内存复制。

### 5.3 避免写出时 to_vec

原 `write_all_and_update(&[u8])` 的设计适合“借用一段数据然后发送副本”。但 pack 编码中最常见的调用场景是：

```rust
encode_one_object(...) -> Vec<u8>
```

调用方已经拥有 `Vec<u8>`。继续借用再 `to_vec()` 会复制完整对象数据。

改为 `write_vec_and_update(Vec<u8>)` 后：

- hash 更新仍然在发送前完成；
- offset 更新仍然使用相同长度；
- 发送给 channel 的数据内容不变；
- 额外复制减少。

### 5.4 用 take 消费 idx_entries

`.idx` 生成需要一份 `Vec<IndexEntry>`。原来使用 `clone()` 是保守做法，但会复制整个数组。

由于 `encode_idx_file` 是最终写 `.idx` 的步骤，消费掉 `self.idx_entries` 是合理的。使用 `take()` 后，重复调用 `encode_idx_file` 会返回缺失错误；这也符合“pack 生成后生成一次 idx”的使用方式。

## 6. 如何验证修改正确

验证思路：

1. 语义不变：pack header、对象编码、hash 更新、offset 更新、idx 写入顺序都不改变。
2. 所有写出的对象字节仍通过同一条发送链路进入 pack writer。
3. `inner_offset` 仍在每次对象写出时增加对象编码长度。
4. `idx_entry.offset` 仍在对象写出前设置为当前 pack offset。
5. pack trailer hash 仍然在所有对象写完后计算并发送。
6. 已有测试会重新 decode 生成的 pack，如果 pack 格式或 offset 错误，测试会失败。

重点验证点：

- 小对象 pack 编码和解码成功；
- SHA-1 和 SHA-256 两种 hash 模式成功；
- delta、zstdelta、无 delta 路径成功；
- 输出 `.pack` 和 `.idx` 文件成功；
- AI 类型对象仍然会被拒绝；
- `cargo fmt --check` 通过。

## 7. 运行了哪些测试，结果如何

### 7.1 编译检查

命令：

```bash
cargo test --no-run test_pack_encoder
```

结果：

- 通过。
- 测试二进制成功构建。

### 7.2 格式检查

命令：

```bash
cargo fmt --check
```

结果：

- 通过。
- 输出了项目现有 rustfmt 配置的 warning：`imports_granularity`、`group_imports`、`unstable_features` 是 nightly-only 配置。
- 这些 warning 不影响格式检查结果。

### 7.3 小范围行为测试

命令：

```bash
cargo test --lib test_pack_encoder_rejects_unencodable_ai_type
```

结果：

```text
2 passed; 0 failed
```

验证内容：

- `test_pack_encoder_rejects_unencodable_ai_type_parallel` 通过；
- `test_pack_encoder_rejects_unencodable_ai_type_delta_window` 通过。

### 7.4 PackEncoder 相关完整测试

第一次在默认 sandbox 中运行：

```bash
cargo test test_pack_encoder
```

结果：

```text
6 passed; 8 failed
```

失败原因：

```text
PermissionDenied: 拒绝访问。
src/internal/pack/cache.rs:178
```

分析：

- 失败发生在测试解码校验清理/访问缓存目录阶段。
- 这是 Windows sandbox 权限导致的临时缓存目录访问问题。
- 不是 Rust 编译错误，也不是本次修改造成的类型错误。

随后在沙箱外重跑同一命令：

```bash
cargo test test_pack_encoder
```

结果：

```text
14 passed; 0 failed
```

通过的关键测试包括：

- `test_pack_encoder`
- `test_pack_encoder_sha256`
- `test_pack_encoder_large_file`
- `test_pack_encoder_large_file_sha256`
- `test_pack_encoder_parallel_large_file`
- `test_pack_encoder_parallel_large_file_sha256`
- `test_pack_encoder_large_file_with_delta`
- `test_pack_encoder_large_file_with_delta_sha256`
- `test_pack_encoder_with_zstdelta`
- `test_pack_encoder_with_zstdelta_sha256`
- `test_pack_encoder_output_to_files`
- `test_pack_encoder_output_to_files_with_delta`
- 两个 AI 类型拒绝测试

## 8. 简单性能分析

本次没有做严格 benchmark，只基于代码路径和测试日志做简单分析。

### 8.1 理论收益

删除的复制包括：

1. `try_as_offset_delta` 中每个对象一次 `obj_data.clone()`。
2. delta 路径写出时的 `Vec<u8> -> &[u8] -> to_vec()`。
3. parallel 路径写出时的 `Vec<u8> -> &[u8] -> to_vec()`。
4. `.idx` 生成时的 `Vec<IndexEntry>` clone。

其中收益最大的通常是对象数据相关的复制，因为 `Vec<u8>` 可能包含完整压缩对象数据；`IndexEntry` 较小，收益相对有限。

### 8.2 测试日志观察

在通过的 `cargo test test_pack_encoder` 日志中可以看到：

- large file parallel encode 路径成功生成 pack，并完成解码校验；
- delta 和 zstdelta 路径也成功生成 pack，并完成解码校验；
- 输出 pack size 和 compression rate 的日志仍正常打印。

部分日志示例：

```text
test result: ok. 14 passed; 0 failed
```

以及测试中出现的简单指标：

```text
new pack file size: 51515670
original total size: 51524886
compression rate: 0.02%
space saved: 9216 bytes
```

这些数字不能作为严格性能结论，因为测试并没有固定单线程环境、重复次数、冷热缓存和基准数据。但它们说明修改后 pack 编码和解码流程可以正常完成。

### 8.3 为什么没有做更深入 benchmark

本实验要求是局部优化，不是全面性能优化。更严格的性能分析需要：

- 固定测试数据；
- 分别测试修改前和修改后；
- 多次运行取均值；
- 区分 CPU 时间、内存峰值、pack size；
- 使用 release 模式。

如果继续扩展，可以增加：

```bash
cargo test --release test_pack_encoder_large_file_with_delta -- --nocapture
```

或者单独写一个 benchmark example，对不同 `window_size` 比较：

- 编码耗时；
- pack 文件大小；
- delta 命中数量；
- 内存峰值。

## 9. 本次修改的边界

本次没有做这些事情：

- 没有引入外部 Git 库。
- 没有改动 delta 算法本身。
- 没有改变 pack 格式。
- 没有改变 zlib 压缩方式。
- 没有重构 `Entry` 的所有权模型。
- 没有删除 `entry.clone()`，因为窗口需要保存当前对象的原始数据作为后续 base，而当前对象可能会被改写为 delta 数据。

## 10. 后续可选优化

如果要继续做更深入的优化，可以考虑：

1. 将窗口中的 `Entry` 改成更轻量的 base 表示，减少 `entry.clone()` 的深拷贝成本。
2. 给 `encode_one_object` 预估容量，减少 `encoded_data` 扩容。
3. 将 `candidates: Vec<_>` 改为并行 reduce，避免每个对象分配候选列表。
4. 为 `window_size` 添加专门测试，验证不同窗口大小会影响 delta 搜索范围。
5. 增加 release benchmark，比较修改前后的复制次数、耗时和内存。

## 11. 最终结论

本次实验完成了 Pack Encode 的局部优化和重构：

- 修正了 `window_size` 在 delta 选择路径中被硬编码的问题。
- 删除了已编码对象数据的无意义 clone。
- 将已有 `Vec<u8>` 的写出路径改为 move，减少复制。
- 使用 `Option::take()` 消费 idx entries，避免索引数组 clone。
- 通过 PackEncoder 相关测试验证：`14 passed; 0 failed`。

该修改范围集中、风险较低，并且和题目要求的 Pack Encode 局部优化目标一致。

## 12. 按任务书执行记录

本节记录按照任务书一步一步完成实验的实际过程，可作为最终提交说明使用。

### 第一步：阅读题目并确定实验目标

题目要求不是完整实现 Git pack 编码器，而是在现有 `git-internal-0.7.3` 项目中选择一个局部问题进行优化或重构。

我将实验目标确定为：

- 读懂 Pack Encode 输入、输出和 delta 选择流程；
- 找出 `Vec` 分配和 `clone()` 较集中的位置；
- 选择一个改动范围小、收益明确的局部问题；
- 修改 Rust 代码；
- 通过测试证明修改没有破坏 pack 编码和解码。

最终选择的局部问题是：

> 修正 delta 路径中 `window_size` 未真正生效的问题，并减少 pack 编码写出路径中的不必要复制。

### 第二步：阅读 `examples/encode_pack.rs`

阅读内容：

- 示例构造字符串数组；
- 将字符串转为 `Blob`；
- 将 `Blob` 转为 `Entry`；
- 将 `Entry` 包装成 `MetaAttached<Entry, EntryMeta>`；
- 通过 Tokio `mpsc` channel 发送给 encoder；
- 调用 `encode_and_output_to_files` 写出 pack 和 idx 文件。

得到结论：

- Pack Encode 的输入是 `mpsc::Receiver<MetaAttached<Entry, EntryMeta>>`。
- `object_number` 用于 pack header 中记录对象数量。
- `window_size` 用于控制是否启用 delta 以及 delta 搜索窗口大小。
- `window_size == 0` 表示不做 delta 压缩。

### 第三步：阅读 `PackEncoder::encode`

阅读 `src/internal/pack/encode.rs` 中的 `PackEncoder::encode`：

```rust
if self.window_size == 0 {
    self.parallel_encode(entry_rx).await
} else {
    self.inner_encode(entry_rx, false).await
}
```

得到结论：

- `encode` 本身不直接编码对象，只负责选择编码路径。
- `window_size == 0` 走 `parallel_encode`。
- `window_size != 0` 走 `inner_encode`。

### 第四步：阅读 `PackEncoder::inner_encode`

阅读重点：

- pack header 如何写出；
- 输入对象如何从 channel 中全部读出；
- 对象如何按 `Commit`、`Tree`、`Blob`、`Tag` 分桶；
- 每个桶如何排序；
- 每个桶如何调用 `try_as_offset_delta`；
- 编码结果如何写出；
- `idx_entries` 如何收集；
- pack trailer hash 如何计算。

发现问题：

```rust
Self::try_as_offset_delta(..., 10, enable_zstdelta)
```

这里四个分桶都传入了硬编码 `10`。这意味着用户创建 `PackEncoder` 时传入的 `window_size` 只有 `0` 和非 `0` 的区别；只要进入 delta 路径，真实窗口大小始终是 `10`。

修改：

```rust
let window_size = self.window_size;
Self::try_as_offset_delta(..., window_size, enable_zstdelta)
```

同时将写出结果的循环从“借用遍历”改成“消费遍历”，避免 `IndexEntry` clone。

### 第五步：阅读 `PackEncoder::try_as_offset_delta`

阅读重点：

- `VecDeque<(Entry, usize)>` 如何作为滑动窗口；
- 当前对象如何在窗口内寻找 base；
- 如何过滤候选 base；
- 如何计算 delta rate；
- 如何选择 `best_base`；
- 如何将当前对象改写为 `OffsetDelta` 或 `OffsetZstdelta`；
- 如何写入窗口；
- 如何生成 `(Vec<u8>, IndexEntry)` 结果。

发现问题：

```rust
res.push((obj_data.clone(), IndexEntry::new(entry, 0)));
current_offset += obj_data.len();
```

`obj_data` 是刚生成的完整编码对象数据，clone 会复制整个 `Vec<u8>`。但是后面只需要它的长度，因此可以先保存长度，再 move。

修改：

```rust
let obj_len = obj_data.len();
res.push((obj_data, IndexEntry::new(entry, 0)));
current_offset += obj_len;
```

### 第六步：阅读 `PackEncoder::parallel_encode`

阅读重点：

- 无 delta 时如何按 batch 接收对象；
- 如何使用 Rayon 并行调用 `encode_one_object`；
- 如何按顺序写出 batch 结果；
- 如何收集 idx entries。

发现问题：

`parallel_encode` 中 batch 编码结果已经拥有 `Vec<u8>`，但写出时传给旧的 `write_all_and_update(&[u8])`，函数内部又执行 `to_vec()` 复制一份。

修改：

```rust
let (data, mut idx_entry) = obj_data?;
idx_entry.offset = self.inner_offset as u64;
self.write_vec_and_update(data).await;
idx_entries.push(idx_entry);
```

### 第七步：重构写出函数

原函数：

```rust
async fn write_all_and_update(&mut self, data: &[u8]) {
    self.inner_hash.update(data);
    self.inner_offset += data.len();
    self.send_data(data.to_vec()).await;
}
```

新函数：

```rust
async fn write_vec_and_update(&mut self, data: Vec<u8>) {
    self.inner_hash.update(&data);
    self.inner_offset += data.len();
    self.send_data(data).await;
}
```

这样修改后：

- hash 更新仍然发生在写出前；
- offset 更新逻辑不变；
- 发送给 channel 的数据内容不变；
- 调用方已有 `Vec<u8>` 时不再复制。

### 第八步：优化 idx entries 消费方式

原代码：

```rust
let idx_entries = self.idx_entries.clone().ok_or(...)?;
```

修改后：

```rust
let idx_entries = self.idx_entries.take().ok_or(...)?;
```

原因：

- `.idx` 文件生成是 pack 编码完成后的最终步骤；
- `idx_entries` 不需要保留第二份；
- `take()` 可以直接移出 `Vec<IndexEntry>`。

### 第九步：创建调研报告

已创建：

```text
pack_encode_local_optimization_report.md
```

该报告记录了：

- Encode 输入是什么；
- 输出如何写出；
- 哪些地方分配 `Vec`；
- 哪些地方发生 clone；
- `window_size` 的作用；
- 可选局部优化点。

### 第十步：创建任务书

已创建：

```text
pack_encode_experiment_task_book.md
```

该任务书记录了：

- 题目意思拆解；
- 任务拆解；
- 修改文件和函数；
- 修改原因；
- 验证方法；
- 测试命令和结果；
- 简单性能分析；
- 后续可选优化。

### 第十一步：运行验证

已运行：

```bash
cargo test --no-run test_pack_encoder
```

结果：

```text
Finished `test` profile
```

说明测试目标可以成功编译。

已运行：

```bash
cargo fmt --check
```

结果：

- 通过；
- 仅出现项目现有 nightly-only rustfmt 配置 warning。

已运行：

```bash
cargo test --lib test_pack_encoder_rejects_unencodable_ai_type
```

结果：

```text
2 passed; 0 failed
```

已运行 PackEncoder 相关完整测试：

```bash
cargo test test_pack_encoder
```

默认 sandbox 中出现 Windows 缓存目录权限错误：

```text
PermissionDenied: 拒绝访问。
src/internal/pack/cache.rs:178
```

随后在沙箱外重跑同一命令，结果：

```text
14 passed; 0 failed
```

说明本次修改没有破坏 PackEncoder 的主要编码、delta、zstdelta、idx 输出和解码校验流程。

### 第十二步：最终交付物

本次实验的最终交付物包括：

- 修改后的 `src/internal/pack/encode.rs`；
- `pack_encode_local_optimization_report.md`；
- `pack_encode_experiment_task_book.md`；
- `pack_encode_experiment_work_record.md`。

### 第十三步：本次实验结论

实验完成后，Pack Encode 局部优化达到以下效果：

- `window_size` 参数在 delta 路径中真正生效；
- 删除了已编码对象数据的无意义 clone；
- 写出已有 `Vec<u8>` 时避免额外 `to_vec()`；
- `.idx` 生成时避免复制整个 `idx_entries`；
- 相关测试通过，pack 编码结果仍可成功解码。

## 13. 追加扩展任务完成情况

在基础任务完成后，继续按照要求完成了以下扩展：

1. 给 `window_size` 补了专门单元测试：`test_try_as_offset_delta_respects_window_size`。
2. 给 `try_as_offset_delta` 增加了 `DeltaEncodeStats` 统计信息，包括对象数、delta 命中数、候选 base 数、平均 chain length。
3. 做了 release 模式 benchmark，并用 baseline worktree 对比修改前后结果。
4. 将窗口中的完整 `Entry` 改为轻量 `DeltaBase`，减少大对象深拷贝。
5. 将 `candidates: Vec<_>` 改为 Rayon `fold/reduce`，减少每个对象选择 base 时的临时 Vec 分配。

详细执行过程、原因、代码设计、测试命令和 benchmark 数据已记录在：

```text
pack_encode_experiment_work_record.md
```

追加验证结果：

```text
cargo test --lib test_try_as_offset_delta_respects_window_size
1 passed; 0 failed

cargo test --lib test_pack_encoder_rejects_unencodable_ai_type
2 passed; 0 failed

cargo test test_pack_encoder
14 passed; 0 failed
```

release benchmark 对比：

| 版本 | test executed | pack size | compression rate | space saved | 近似 peak WorkingSet |
| --- | ---: | ---: | ---: | ---: | ---: |
| baseline HEAD | 726.98ms | 51,566,925 | -0.08% | 0 bytes | 155.60 MB |
| 当前优化后 | 720.89ms | 51,515,728 | 0.02% | 9,158 bytes | 128.51 MB |
