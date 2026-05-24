# 实验 2：Pack Encode 局部优化调研报告

## 1. 阅读范围

本报告基于当前工作目录中的 `git-internal-0.7.3` 代码，重点阅读了以下文件和函数：

- `examples/encode_pack.rs`
- `src/internal/pack/encode.rs`
- `PackEncoder::encode`
- `PackEncoder::inner_encode`
- `PackEncoder::try_as_offset_delta`
- `PackEncoder::parallel_encode`
- 辅助阅读：`src/internal/pack/entry.rs`、`src/internal/pack/index_entry.rs`、`src/internal/pack/pack_index.rs`、`src/delta/mod.rs`

本次选择的局部优化方向是：减少 Pack Encode 中不必要的内存分配和深拷贝，并修正 delta 窗口大小参数没有被真正使用的问题。

## 2. Encode 的输入是什么

示例入口在 `examples/encode_pack.rs`。示例先构造字符串内容：

```rust
let contents = vec!["Hello World", "Rust is awesome", "Git internals are fun"];
```

然后每个字符串被转换为：

```rust
Blob -> Entry -> MetaAttached<Entry, EntryMeta>
```

最后通过 Tokio `mpsc` channel 发送给 pack encoder：

```rust
let (entry_tx, entry_rx) = mpsc::channel(100);
entry_tx.send(meta_entry).await?;
```

因此，Pack Encode 的核心输入是：

```rust
mpsc::Receiver<MetaAttached<Entry, EntryMeta>>
```

其中 `Entry` 是 pack 编码真正关心的对象主体：

```rust
pub struct Entry {
    pub obj_type: ObjectType,
    pub data: Vec<u8>,
    pub hash: ObjectHash,
    pub chain_len: usize,
}
```

字段含义：

- `obj_type`：对象类型，例如 `Blob`、`Commit`、`Tree`、`Tag`，或者 delta 后的 `OffsetDelta`、`OffsetZstdelta`。
- `data`：对象内容。对于普通对象是原始对象数据；对于 delta 对象会被替换成 delta 指令数据。
- `hash`：对象 hash，用于 `.idx` 文件和对象身份识别。
- `chain_len`：delta 链长度，用来限制过长的 delta 链。

`EntryMeta` 中包含文件路径、pack offset、crc32 等元信息。当前 `inner_encode` 在分桶时会读取 `meta.file_path` 参与排序，但真正进入 `try_as_offset_delta` 后只保留 `Entry`。

## 3. Pack Encode 工作流程

### 3.1 示例入口

`examples/encode_pack.rs` 的流程是：

1. 准备若干 Blob 内容。
2. 设置 `object_number` 和 `window_size`。
3. 创建 `entry_tx / entry_rx` channel。
4. `tokio::spawn` 启动 `encode_and_output_to_files(...)`。
5. 逐个发送 `MetaAttached<Entry, EntryMeta>`。
6. `drop(entry_tx)` 通知 encoder 输入结束。

示例里的：

```rust
let window_size = 10;
```

注释写明：`0` 表示不启用 delta 压缩，非 `0` 表示启用 delta 窗口。

### 3.2 encode_and_output_to_files

`encode_and_output_to_files` 是文件输出包装层：

1. 创建 `pack_tx / pack_rx` 和 `idx_tx / idx_rx`。
2. 使用 `PackEncoder::new_with_idx(...)` 创建 encoder。
3. 先创建临时 pack 文件 `*.pack.tmp`。
4. 启动 pack writer 任务，从 `pack_rx` 不断收 `Vec<u8>` 并写入文件。
5. 调用 `pack_encoder.encode(raw_entries_rx).await?`。
6. pack 写完后，用 `final_hash` 重命名为 `pack-<hash>.pack`。
7. 再启动 idx writer 任务。
8. 调用 `pack_encoder.encode_idx_file().await?` 写 `.idx`。

所以输出不是一次性返回一个完整 pack，而是通过 channel 流式发送多个 `Vec<u8>` chunk，再由后台 writer 写入文件。

### 3.3 PackEncoder::encode

`PackEncoder::encode` 是路由函数：

```rust
if self.window_size == 0 {
    self.parallel_encode(entry_rx).await
} else {
    self.inner_encode(entry_rx, false).await
}
```

含义：

- `window_size == 0`：不做 delta，走 `parallel_encode`，对象可以批量并行 zlib 压缩。
- `window_size != 0`：启用 delta 选择，走 `inner_encode`。

### 3.4 PackEncoder::inner_encode

`inner_encode` 是启用 delta 时的主流程：

1. 生成 pack header：

   ```rust
   let head = encode_header(self.object_number);
   ```

2. 把 header 发给 pack writer，并更新 pack hash。
3. 从 `entry_rx` 读完所有对象。
4. 按对象类型分桶：

   ```rust
   commits
   trees
   blobs
   tags
   ```

5. 每个桶使用 `magic_sort` 排序。排序主要考虑：

   - 文件路径父目录；
   - 文件名自然排序；
   - 对象大小，大对象优先；
   - 最后用指针地址作为 fallback。

6. 四个桶通过 `tokio::task::spawn_blocking` 并行执行 `try_as_offset_delta`。
7. 将四个桶的编码结果合并。
8. 按顺序写出每个对象的编码数据，并记录 `.idx` 所需的 offset。
9. 计算 pack trailer hash，写到 pack 文件末尾。
10. 保存 `idx_entries`，供后续 `encode_idx_file` 生成 `.idx`。

一个需要重点注意的问题：

```rust
Self::try_as_offset_delta(..., 10, enable_zstdelta)
```

当前 `inner_encode` 给 `try_as_offset_delta` 传入的是硬编码 `10`，而不是 `self.window_size`。因此，只要用户传入的 `window_size` 不是 `0`，实际 delta 窗口大小都会固定为 `10`。这与 `PackEncoder::encode` 和示例注释表达的语义不一致。

### 3.5 PackEncoder::try_as_offset_delta

`try_as_offset_delta` 对单个类型桶做 delta 选择和对象编码。它的输入是：

```rust
bucket: Vec<Entry>
window_size: usize
enable_zstdelta: bool
```

输出是：

```rust
Result<Vec<(Vec<u8>, IndexEntry)>, GitError>
```

也就是每个对象的 pack 编码字节和对应的 index entry。

核心流程：

1. 建立滑动窗口：

   ```rust
   let mut window: VecDeque<(Entry, usize)> = VecDeque::with_capacity(window_size);
   ```

   窗口里保存最近已经处理过的对象和它在当前桶内的 offset。

2. 对 `bucket.iter_mut()` 中的当前对象，在 `window` 中寻找候选 base。

3. 候选 base 需要通过多层过滤：

   - base 和当前对象类型相同；
   - base 的 `chain_len` 小于 `MAX_CHAIN_LEN`；
   - base hash 和当前对象 hash 不同；
   - 两个对象大小比例不能差太多；
   - `cheap_similar` 前 128 字节 hash 相同；
   - delta rate 大于 `MIN_DELTA_RATE`。

4. 如果有多个候选 base，根据 rate 和 tie breaker 选出 `best_base`。

5. 如果存在 `best_base`，把当前对象改写为 `OffsetDelta` 或 `OffsetZstdelta`：

   ```rust
   entry.obj_type = ObjectType::OffsetDelta;
   entry.data = delta::encode(&best_base.0.data, &entry.data);
   entry.chain_len = best_base.0.chain_len + 1;
   ```

6. 使用 `encode_one_object(entry, offset)` 生成 pack 内部格式：

   - 对象头；
   - delta offset，如果是 offset delta；
   - zlib 压缩后的对象数据。

7. 将当前对象的原始版本放进窗口，供后续对象作为 base。

8. 如果窗口长度超过 `window_size`，弹出最旧对象：

   ```rust
   if window.len() > window_size {
       window.pop_front();
   }
   ```

### 3.6 PackEncoder::parallel_encode

`parallel_encode` 只允许在 `window_size == 0` 时使用：

```rust
if self.window_size != 0 {
    return Err(...);
}
```

它不做 delta 选择，流程更简单：

1. 写 pack header。
2. 循环从 channel 中收对象，每次收一个 batch。
3. batch 内使用 Rayon `par_iter()` 并行调用 `encode_one_object(entry, None)`。
4. 主线程按结果顺序写出对象数据、更新 hash 和 offset、收集 idx entries。
5. 最后写 pack trailer hash。

这个路径保留了比较好的流式特性：它不需要一次性收完所有对象，只需要按 batch 处理。

## 4. 输出数据是如何写出的

Pack 文件的数据写出路径是：

```text
encode_one_object / encode_header
        |
        v
PackEncoder::write_all_and_update / PackEncoder::send_data
        |
        v
mpsc::Sender<Vec<u8>>
        |
        v
pack writer task
        |
        v
tokio::fs::File::write_all
```

在 `write_all_and_update` 中：

```rust
self.inner_hash.update(data);
self.inner_offset += data.len();
self.send_data(data.to_vec()).await;
```

这里会更新 pack hash 和当前全局 offset，然后把字节复制成 `Vec<u8>` 发给 channel。

Pack 文件尾部还会写入 pack hash：

```rust
let hash_result = self.inner_hash.clone().finalize();
self.final_hash = Some(ObjectHash::from_bytes(&hash_result).unwrap());
self.send_data(hash_result.to_vec()).await;
```

`.idx` 文件在 pack 完成后生成：

```rust
pack_encoder.encode_idx_file().await?;
```

`IdxBuilder::write_idx` 会依次写：

1. idx v2 header；
2. fanout table；
3. object names；
4. crc32；
5. offsets；
6. trailer，也就是 pack hash 和 idx hash。

## 5. 哪些地方会分配 Vec

### 5.1 Pack header 和 offset 编码

`encode_header` 返回 `Vec<u8>`：

```rust
let mut result: Vec<u8> = vec![b'P', b'A', b'C', b'K', ...];
result.append((object_number as u32).to_be_bytes().to_vec().as_mut());
```

这里 `to_be_bytes().to_vec()` 会产生一个临时 `Vec<u8>`。

`encode_offset` 也返回 `Vec<u8>`，用于 offset delta 的偏移编码。

### 5.2 单对象编码

`encode_one_object` 中有多处分配：

- `encoded_data = Vec::new()`；
- `header_data = vec![...]`；
- zlib encoder 使用 `ZlibEncoder::new(Vec::new(), ...)`；
- `inflate.finish()` 返回压缩后的 `Vec<u8>`；
- `encoded_data.extend(compressed_data)` 将压缩结果追加进对象输出。

这是 pack 编码的主要分配热点之一，因为每个对象都会调用一次。

### 5.3 inner_encode 收集和排序

启用 delta 时，`inner_encode` 会把所有输入对象先读完并分配到四个 Vec：

```rust
let mut commits = Vec::new();
let mut trees = Vec::new();
let mut blobs = Vec::new();
let mut tags = Vec::new();
```

随后每个桶转换为 `Vec<Entry>`：

```rust
.map(|entry_with_meta| entry_with_meta.inner).collect()
```

四个桶的 `try_as_offset_delta` 又各自返回：

```rust
Vec<(Vec<u8>, IndexEntry)>
```

最后还有：

```rust
let mut all_res = vec![commit_res, tree_res, blob_res, tag_res];
let mut idx_entries = Vec::new();
```

因此 delta 路径并不是严格流式的，它会在内存中持有：

- 所有输入对象；
- 排序后的四个桶；
- 每个对象编码后的 `Vec<u8>`；
- index entries。

### 5.4 try_as_offset_delta 内部

`try_as_offset_delta` 中的主要分配：

- `VecDeque<(Entry, usize)>` 作为滑动窗口；
- `res: Vec<(Vec<u8>, IndexEntry)>` 保存所有编码结果；
- `candidates: Vec<_>` 收集所有候选 base；
- delta 编码结果 `delta::encode(...) -> Vec<u8>`；
- zstdelta 编码结果 `zstdelta::diff(...) -> Vec<u8>`；
- `encode_one_object(...) -> Vec<u8>`。

其中 `candidates: Vec<_>` 是一个可以优化的局部分配点：当前代码先并行收集全部候选，再串行挑选 best base。如果窗口较小影响有限；如果窗口变大，这个 Vec 会变成每个对象一次的额外分配。

### 5.5 parallel_encode 内部

无 delta 路径中：

- 每批创建 `batch_entries = Vec::with_capacity(batch_size)`；
- 并行编码结果收集为 `Vec<Result<(Vec<u8>, IndexEntry), GitError>>`；
- 每个对象仍由 `encode_one_object` 产生 `Vec<u8>`；
- `idx_entries = Vec::new()` 保存索引条目。

### 5.6 idx 生成

`generate_idx_file` 中：

```rust
let idx_entries = self.idx_entries.clone().ok_or(...)?;
```

这里会复制整个 `Vec<IndexEntry>`。虽然 `IndexEntry` 不包含大 `Vec<u8>`，复制成本比 `Entry` 小很多，但这仍然是不必要的整体 clone。

`pack_index.rs` 中也有一些小 Vec 分配：

- header `to_vec()`；
- `send_u32` / `send_u64` 使用 `to_be_bytes().to_vec()`；
- `write_offsets` 中 `large = vec![]`；
- hash 数据 `to_data().clone()`。

## 6. 哪些地方可能发生不必要的 clone

### 6.1 header clone

`inner_encode` 和 `parallel_encode` 都有：

```rust
self.send_data(head.clone()).await;
self.inner_hash.update(&head);
```

这里 clone 的原因是 `send_data` 需要拥有 `Vec<u8>`，而 hash 更新还要继续借用 `head`。因为 header 只有 12 字节，这个 clone 成本很小，不是优先优化点。

### 6.2 Entry 深拷贝

`try_as_offset_delta` 中：

```rust
let mut entry_for_window = entry.clone();
```

这是当前最值得关注的 clone。`Entry` 包含 `data: Vec<u8>`，所以 `entry.clone()` 会复制完整对象数据。

为什么代码需要这个 clone？

因为当前对象如果被选为 delta，会执行：

```rust
entry.data = delta;
entry.obj_type = ObjectType::OffsetDelta;
```

但是窗口里需要保存的是可作为后续 base 的原始对象数据，而不是 delta 后的数据。因此代码先 clone 出 `entry_for_window`，后续将这个原始版本放入窗口。

这个 clone 在语义上可以理解，但代价明显：

- 每个对象至少复制一次完整 `data`；
- 大 blob 的复制成本高；
- 即使当前对象最终没有被 delta，也仍会 clone。

### 6.3 obj_data clone

`try_as_offset_delta` 中：

```rust
res.push((obj_data.clone(), IndexEntry::new(entry, 0)));
current_offset += obj_data.len();
```

这里 `obj_data` 已经是新生成的 `Vec<u8>`，可以先保存长度，再直接 move 进 `res`：

```rust
let obj_len = obj_data.len();
res.push((obj_data, IndexEntry::new(entry, 0)));
current_offset += obj_len;
```

这是一个非常明确的不必要 clone。它会复制完整的压缩对象数据，属于低风险、高收益的小优化点。

### 6.4 idx_entries clone

`inner_encode` 中：

```rust
idx_entries.push(data.1.clone());
```

这里是因为循环遍历的是 `&mut all_res`，拿到的是引用，所以需要 clone。可以通过消费 `all_res` 来避免：

```rust
for res in all_res {
    for (data, mut idx_entry) in res {
        idx_entry.offset = self.inner_offset as u64;
        self.write_all_and_update(&data).await;
        idx_entries.push(idx_entry);
    }
}
```

由于 `IndexEntry` 很小，这不是最严重的问题，但可以顺手重构掉。

### 6.5 write_all_and_update 中的 to_vec

```rust
self.send_data(data.to_vec()).await;
```

这会复制一份即将写出的对象数据。当前函数签名接收 `&[u8]`，为了通过 channel 发送拥有所有权的 `Vec<u8>`，只能复制。

如果更深入优化，可以增加一个拥有所有权的写出函数：

```rust
async fn write_vec_and_update(&mut self, data: Vec<u8>) {
    self.inner_hash.update(&data);
    self.inner_offset += data.len();
    self.send_data(data).await;
}
```

这样在调用方已经有 `Vec<u8>` 的时候，可以避免 `to_vec()`。

### 6.6 generate_idx_file 中的 idx_entries.clone

```rust
let idx_entries = self.idx_entries.clone().ok_or(...)?;
```

这里可以用 `take()` 消费掉 `Option<Vec<IndexEntry>>`：

```rust
let idx_entries = self.idx_entries.take().ok_or(...)?;
```

这样可以避免复制整个索引数组。因为 `encode_idx_file` 本来就是 pack 完成后的最终步骤，消费掉 `idx_entries` 是合理的。

## 7. window_size 在 delta 选择中的作用

`window_size` 的设计意图是控制 delta base 搜索窗口大小。

在 `try_as_offset_delta` 中，窗口定义为：

```rust
let mut window: VecDeque<(Entry, usize)> = VecDeque::with_capacity(window_size);
```

每处理一个对象：

1. 只在窗口中已有的对象里找 base。
2. 当前对象处理完成后，被加入窗口。
3. 如果窗口长度超过 `window_size`，移除最旧的对象。

因此，`window_size` 控制了“当前对象最多参考最近多少个已编码对象来尝试 delta”。

影响如下：

- `window_size == 0`：不启用 delta，`PackEncoder::encode` 直接走 `parallel_encode`。
- `window_size` 较小：base 搜索范围小，速度快，内存占用低，但可能错过更好的 delta base。
- `window_size` 较大：base 搜索范围大，可能得到更小的 pack，但会增加 CPU、内存和候选比较成本。

从当前代码看，窗口不仅影响搜索质量，还直接影响这些成本：

- `VecDeque` 会保存最多 `window_size` 个 `Entry`；
- 每个 `Entry` 包含完整 `Vec<u8>`，所以窗口越大，保存的原始对象数据越多；
- 每个当前对象都会遍历窗口，做类型、hash、大小比例、cheap similarity 和 delta rate 检查；
- 窗口越大，`candidates: Vec<_>` 可能越大。

不过当前实现存在一个语义 bug：`inner_encode` 没有把 `self.window_size` 传入 `try_as_offset_delta`，而是固定传入 `10`。

这意味着：

- 用户传 `window_size = 1`，实际 delta 搜索窗口仍是 `10`；
- 用户传 `window_size = 100`，实际 delta 搜索窗口仍是 `10`；
- 只有 `window_size = 0` 的特殊含义能生效，也就是切换到无 delta 并行编码。

建议修正为：

```rust
Self::try_as_offset_delta(..., self.window_size, enable_zstdelta)
```

因为四个 `spawn_blocking` 闭包需要 move 数据，可以先保存：

```rust
let window_size = self.window_size;
```

然后每个闭包传入 `window_size`。

## 8. 本实验选择的局部优化点

本实验建议选择如下局部优化，不涉及外部 Git 库，全部可以用 Rust 在当前模块内完成。

### 优化点 A：修正 window_size 被硬编码为 10

问题位置：`PackEncoder::inner_encode` 调用 `try_as_offset_delta` 的四个地方。

当前行为：

```rust
Self::try_as_offset_delta(entries, 10, enable_zstdelta)
```

建议行为：

```rust
let window_size = self.window_size;
Self::try_as_offset_delta(entries, window_size, enable_zstdelta)
```

收益：

- 修正 API 语义；
- 让示例和注释中的 `window_size` 真正影响 delta 选择；
- 便于后续 benchmark 比较不同窗口大小的 pack size 和耗时。

风险：

- 如果已有测试隐式依赖固定窗口 `10`，结果 pack 大小可能变化；
- 但这是更符合接口定义的行为。

### 优化点 B：去掉 obj_data.clone

问题位置：`try_as_offset_delta`。

当前代码：

```rust
let obj_data = encode_one_object(entry, offset)?;
window.push_back((entry_for_window, current_offset));
if window.len() > window_size {
    window.pop_front();
}
res.push((obj_data.clone(), IndexEntry::new(entry, 0)));
current_offset += obj_data.len();
```

建议改为：

```rust
let obj_data = encode_one_object(entry, offset)?;
let obj_len = obj_data.len();
window.push_back((entry_for_window, current_offset));
if window.len() > window_size {
    window.pop_front();
}
res.push((obj_data, IndexEntry::new(entry, 0)));
current_offset += obj_len;
```

收益：

- 避免复制完整的已编码对象数据；
- 对大对象或大量对象更明显；
- 改动范围极小，不改变输出语义。

### 优化点 C：消费 idx_entries，避免整体 clone

问题位置：`generate_idx_file`。

当前代码：

```rust
let idx_entries = self.idx_entries.clone().ok_or(...)?;
```

建议改为：

```rust
let idx_entries = self.idx_entries.take().ok_or(...)?;
```

收益：

- 避免复制整个 `Vec<IndexEntry>`；
- 符合 `encode_idx_file` 是最终消费步骤的语义。

风险：

- 调用一次 `encode_idx_file` 后，`idx_entries` 会变成 `None`；
- 当前 API 语义本来就是“pack 生成后再生成 idx”，通常不需要重复生成。

### 优化点 D：新增拥有所有权的写出函数，减少 to_vec

问题位置：`write_all_and_update`。

当前代码：

```rust
async fn write_all_and_update(&mut self, data: &[u8]) {
    self.inner_hash.update(data);
    self.inner_offset += data.len();
    self.send_data(data.to_vec()).await;
}
```

可以保留该函数，同时新增：

```rust
async fn write_vec_and_update(&mut self, data: Vec<u8>) {
    self.inner_hash.update(&data);
    self.inner_offset += data.len();
    self.send_data(data).await;
}
```

这样对于 `encode_one_object` 已经返回 `Vec<u8>` 的路径，可以直接 move 给 channel，避免再次复制。

风险：

- 需要调整调用方，确保写 hash 和写 channel 的顺序不变；
- 如果调用方之后还需要使用 `data`，要先保存长度或相关信息。

## 9. 推荐实施顺序

推荐按低风险优先顺序实施：

1. 修正 `inner_encode` 中硬编码 `10` 为 `self.window_size`。
2. 删除 `try_as_offset_delta` 中的 `obj_data.clone()`。
3. 将 `generate_idx_file` 的 `self.idx_entries.clone()` 改成 `self.idx_entries.take()`。
4. 视测试情况再新增 `write_vec_and_update`，减少 `data.to_vec()`。
5. 如果还要继续优化，再考虑重构窗口保存结构，降低 `Entry::clone()` 对 `data: Vec<u8>` 的深拷贝成本。

## 10. 总结

Pack Encode 的输入是一个异步 channel，元素类型是 `MetaAttached<Entry, EntryMeta>`。输出通过 `mpsc::Sender<Vec<u8>>` 分块发送给后台 writer，最终写成 `.pack` 和 `.idx` 文件。

当前代码有两个编码路径：

- `window_size == 0`：无 delta，走 `parallel_encode`，按 batch 并行压缩，流式性较好。
- `window_size != 0`：启用 delta，走 `inner_encode`，先全量收集、分桶、排序，再按窗口选择 delta base。

主要 Vec 分配集中在对象编码、delta 结果、批处理结果、候选 base 收集、索引条目收集等位置。主要 clone 风险集中在 `Entry` 的深拷贝和已编码对象数据的 `obj_data.clone()`。

最值得优先处理的局部问题是：

1. `window_size` 在 delta 路径中被硬编码为 `10`，用户传入值没有真正生效。
2. `try_as_offset_delta` 中 `obj_data.clone()` 是明确的不必要复制。

这两个点都属于局部优化或重构，改动小、收益明确，也符合本实验“不做全面 Pack 编码性能优化”的要求。
