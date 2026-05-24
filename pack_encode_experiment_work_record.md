# Pack Encode 局部优化工作记录

## 1. 工作目标

本次工作根据 `pack_encode_experiment_task_book.md` 执行实验 2：Pack Encode 局部优化。

目标是：

- 阅读当前 Rust 文件，理解 Pack Encode 的核心流程；
- 根据调研结果选择一个局部问题；
- 修改代码完成优化或重构；
- 运行测试验证正确性；
- 用 Markdown 记录工作过程和结果。

## 2. 执行步骤

### 步骤 1：阅读示例入口

阅读文件：

```text
examples/encode_pack.rs
```

理解到的流程：

1. 示例先创建字符串内容。
2. 字符串转换为 `Blob`。
3. `Blob` 转换为 `Entry`。
4. `Entry` 包装为 `MetaAttached<Entry, EntryMeta>`。
5. 通过 Tokio `mpsc` channel 发送给 encoder。
6. `encode_and_output_to_files` 将编码结果写成 `.pack` 和 `.idx` 文件。

结论：

Pack Encode 的直接输入是：

```rust
mpsc::Receiver<MetaAttached<Entry, EntryMeta>>
```

### 步骤 2：阅读 PackEncoder::encode

阅读文件：

```text
src/internal/pack/encode.rs
```

关键逻辑：

```rust
if self.window_size == 0 {
    self.parallel_encode(entry_rx).await
} else {
    self.inner_encode(entry_rx, false).await
}
```

结论：

- `window_size == 0`：不做 delta，走 `parallel_encode`。
- `window_size != 0`：启用 delta，走 `inner_encode`。

### 步骤 3：阅读 PackEncoder::inner_encode

重点阅读内容：

- pack header 写出；
- 输入对象分桶；
- `magic_sort` 排序；
- 调用 `try_as_offset_delta`；
- 写出编码对象；
- 维护 pack hash；
- 收集 idx entries。

发现的问题：

```rust
Self::try_as_offset_delta(..., 10, enable_zstdelta)
```

虽然 `PackEncoder` 有 `window_size` 字段，但 delta 路径中实际传入的是固定值 `10`。

影响：

- 用户传入 `window_size = 1` 时，实际仍使用 10；
- 用户传入 `window_size = 100` 时，实际仍使用 10；
- 只有 `window_size == 0` 的开关效果生效，具体窗口大小不生效。

处理方式：

```rust
let window_size = self.window_size;
Self::try_as_offset_delta(..., window_size, enable_zstdelta)
```

### 步骤 4：阅读 PackEncoder::try_as_offset_delta

重点阅读内容：

- `VecDeque` 滑动窗口；
- base 候选对象筛选；
- delta rate 计算；
- `OffsetDelta` / `OffsetZstdelta` 改写；
- 当前对象加入窗口；
- 编码对象结果收集。

发现的问题：

```rust
res.push((obj_data.clone(), IndexEntry::new(entry, 0)));
current_offset += obj_data.len();
```

`obj_data.clone()` 会复制完整编码对象数据。

处理方式：

```rust
let obj_len = obj_data.len();
res.push((obj_data, IndexEntry::new(entry, 0)));
current_offset += obj_len;
```

这样先保存长度，再把 `obj_data` move 进结果数组。

### 步骤 5：阅读 PackEncoder::parallel_encode

重点阅读内容：

- batch 接收对象；
- Rayon 并行编码；
- batch 结果按顺序写出；
- idx entries 收集。

发现的问题：

原写出路径会把已有 `Vec<u8>` 借用成 `&[u8]`，再在 `write_all_and_update` 内部 `to_vec()` 复制。

处理方式：

将写出路径改为直接消费 `Vec<u8>`。

### 步骤 6：修改写出函数

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

修改后：

- 数据写出内容不变；
- hash 更新逻辑不变；
- offset 更新逻辑不变；
- 避免 `data.to_vec()` 的额外复制。

### 步骤 7：优化 idx_entries 的消费方式

原代码：

```rust
let idx_entries = self.idx_entries.clone().ok_or(...)?;
```

新代码：

```rust
let idx_entries = self.idx_entries.take().ok_or(...)?;
```

理由：

`.idx` 文件生成只需要消费一次 `idx_entries`，没有必要复制整个 `Vec<IndexEntry>`。

## 3. 实际修改清单

修改文件：

```text
src/internal/pack/encode.rs
```

修改函数：

- `PackEncoder::inner_encode`
- `PackEncoder::try_as_offset_delta`
- `PackEncoder::parallel_encode`
- `PackEncoder::write_vec_and_update`
- `PackEncoder::generate_idx_file`

新增文档：

```text
pack_encode_local_optimization_report.md
pack_encode_experiment_task_book.md
pack_encode_experiment_work_record.md
```

## 4. 验证记录

### 编译检查

命令：

```bash
cargo test --no-run test_pack_encoder
```

结果：

```text
Finished `test` profile
```

结论：

测试目标可以成功编译。

### 格式检查

命令：

```bash
cargo fmt --check
```

结果：

- 检查通过；
- 出现项目已有 nightly-only rustfmt 配置 warning；
- warning 不影响本次修改。

### 小范围行为测试

命令：

```bash
cargo test --lib test_pack_encoder_rejects_unencodable_ai_type
```

结果：

```text
2 passed; 0 failed
```

### PackEncoder 相关完整测试

命令：

```bash
cargo test test_pack_encoder
```

默认 sandbox 中结果：

```text
6 passed; 8 failed
```

失败原因：

```text
PermissionDenied: 拒绝访问。
src/internal/pack/cache.rs:178
```

这属于 Windows sandbox 对测试缓存目录的权限限制。

在沙箱外重跑同一命令后：

```text
14 passed; 0 failed
```

结论：

PackEncoder 相关测试通过，说明本次修改没有破坏 pack 编码、delta 编码、zstdelta 编码、idx 文件生成和 pack 解码校验。

## 5. 简单性能分析

本次没有做严格 benchmark，但从代码路径可以确定减少了以下复制：

1. 删除 `try_as_offset_delta` 中对完整 `obj_data` 的 clone。
2. delta 写出路径中，已有 `Vec<u8>` 直接 move，避免 `to_vec()`。
3. parallel 写出路径中，已有 `Vec<u8>` 直接 move，避免 `to_vec()`。
4. `.idx` 生成时用 `take()` 消费 `Vec<IndexEntry>`，避免 clone。

这些优化不会改变算法复杂度，但会减少内存复制和临时分配。对象越大、对象数量越多，收益越明显。

测试日志中出现的 pack size 和耗时仍正常输出，例如：

```text
new pack file size: 51515670
original total size: 51524886
compression rate: 0.02%
space saved: 9216 bytes
```

这些数据只能作为功能运行观察，不作为严格 benchmark 结论。

## 6. 最终结论

本次工作已按照任务书完成：

- 完成 Pack Encode 工作流程调研；
- 明确 Encode 输入和输出方式；
- 梳理 `Vec` 分配与 clone 热点；
- 修正 `window_size` 未真正传入 delta 窗口的问题；
- 删除多个不必要的数据复制；
- 创建调研报告、任务书和工作记录；
- 通过相关测试验证修改正确性。

本次修改满足实验要求：

- 未使用外部 Git 库；
- 实现全部在 Rust 代码内完成；
- 选择的是 Pack Encode 的局部优化问题；
- 没有进行超出实验范围的大规模重构。

## 7. 追加任务执行记录

本节记录后续追加的五项改动：

- 给 `window_size` 补专门单元测试；
- 给 `try_as_offset_delta` 增加统计信息；
- 做 release 模式 benchmark，对比修改前后耗时、pack size、内存占用；
- 继续优化 `entry.clone()`，把窗口里的 `Entry` 改成轻量 base 结构；
- 把 `candidates: Vec<_>` 改成并行 reduce，减少临时分配。

### 7.1 给 window_size 补专门单元测试

为什么做：

之前已经发现 `inner_encode` 中曾经把 delta 窗口硬编码为 `10`。虽然代码已经改为传入 `self.window_size`，但还需要一个专门测试证明不同窗口大小确实会影响 delta base 搜索范围。

怎么做：

在 `src/internal/pack/encode.rs` 的测试模块中新增：

```rust
#[test]
fn test_try_as_offset_delta_respects_window_size()
```

测试构造 3 个 Blob entry：

- A：256 字节 `a`；
- B：256 字节 `b`，与 A/C 都不相似；
- C：与 A 基本相同，只在第 200 字节不同。

排列顺序是：

```text
A -> B -> C
```

这样设计的原因：

- 当 `window_size = 1` 时，处理 C 时窗口里只剩紧邻的 B；B 与 C 不相似，所以不能 delta。
- 当 `window_size = 10` 时，处理 C 时窗口里还能看到更早的 A；A 与 C 相似，所以可以 delta。

测试断言：

```rust
assert_eq!(window_1_stats.delta_count, 0);
assert_eq!(window_10_stats.delta_count, 1);
assert!(window_10_stats.candidate_count > window_1_stats.candidate_count);
assert_eq!(window_10_stats.average_chain_len(), 1.0);
```

验证命令：

```bash
cargo test --lib test_try_as_offset_delta_respects_window_size
```

结果：

```text
1 passed; 0 failed
```

### 7.2 给 try_as_offset_delta 增加统计信息

为什么做：

原来的 `try_as_offset_delta` 只返回编码结果，看不到 delta 选择过程中的关键指标。为了分析 `window_size` 和 delta 策略，需要统计：

- 处理了多少对象；
- 产生了多少 delta；
- 有多少个合格候选 base；
- 平均 delta chain length。

怎么做：

新增内部结构：

```rust
struct DeltaEncodeStats {
    object_count: usize,
    delta_count: usize,
    candidate_count: usize,
    chain_len_sum: usize,
}
```

并提供：

```rust
fn average_chain_len(&self) -> f64
```

同时将原来的 delta 编码核心拆成：

```rust
fn try_as_offset_delta_with_stats(...) -> Result<(Vec<(Vec<u8>, IndexEntry)>, DeltaEncodeStats), GitError>
```

原有外部调用保持：

```rust
fn try_as_offset_delta(...) -> Result<Vec<(Vec<u8>, IndexEntry)>, GitError>
```

它内部调用带 stats 的版本，然后通过 `tracing::info!` 输出统计：

```rust
tracing::info!(
    "delta encode stats: objects={} deltas={} candidate_bases={} avg_chain_len={:.2}",
    stats.object_count,
    stats.delta_count,
    stats.candidate_count,
    stats.average_chain_len()
);
```

这样做的好处：

- 不改变 `try_as_offset_delta` 原来的返回类型；
- 测试可以直接调用 `try_as_offset_delta_with_stats` 做断言；
- 正常编码路径能输出可观察的 delta 统计。

### 7.3 优化 entry.clone：使用轻量 DeltaBase 窗口

为什么做：

原实现中窗口保存的是：

```rust
VecDeque<(Entry, usize)>
```

每处理一个对象时会执行：

```rust
let mut entry_for_window = entry.clone();
```

`Entry` 包含：

```rust
pub data: Vec<u8>
```

所以 `entry.clone()` 会深拷贝完整对象数据。对于大 Blob，这个成本很高。

怎么做：

新增轻量窗口结构：

```rust
struct DeltaBase {
    obj_type: ObjectType,
    data: Vec<u8>,
    hash: ObjectHash,
    chain_len: usize,
    offset: usize,
}
```

然后把窗口改成：

```rust
let mut window: VecDeque<DeltaBase> = VecDeque::with_capacity(window_size);
```

核心思路：

原来 `bucket.iter_mut()` 借用对象，因此为了把原始对象放入窗口，只能 clone。现在改为消费 `bucket`：

```rust
for mut entry in bucket {
```

随后用：

```rust
let original_data = std::mem::take(&mut entry.data);
```

把原始 `Vec<u8>` 从 `entry` 中移出。

如果当前对象被 delta：

- 用 `original_data` 和 base 计算 delta；
- `entry.data` 放 delta 数据，用于 pack 编码；
- `original_data` move 进 `DeltaBase`，作为后续对象的 base。

如果当前对象不被 delta：

- 把 `original_data` 放回 `entry.data`；
- 编码完成后再用 `std::mem::take(&mut entry.data)` 移进 `DeltaBase`。

这样避免了生产路径中为窗口 base 克隆完整 `Entry`。

### 7.4 把 candidates Vec 改成并行 reduce

为什么做：

原实现会为每个对象分配候选列表：

```rust
let candidates: Vec<_> = window.par_iter().filter_map(...).collect();
```

然后再串行遍历 `candidates` 选出最佳 base。

问题：

- 每个对象都会多一次候选 `Vec` 分配；
- `window_size` 越大，候选列表潜在越大；
- 先 collect 再选 best 属于中间结果，可以直接用 reduce 消掉。

怎么做：

新增候选结构：

```rust
struct DeltaCandidate {
    index: usize,
    rate: f64,
    chain_len: usize,
}
```

新增比较函数：

```rust
fn choose_delta_candidate(
    current: Option<DeltaCandidate>,
    candidate: DeltaCandidate,
    tie_epsilon: f64,
) -> Option<DeltaCandidate>
```

在 `try_as_offset_delta_with_stats` 中改为：

```rust
let (best_candidate, candidate_count) = window
    .par_iter()
    .enumerate()
    .with_min_len(3)
    .fold(...)
    .reduce(...);
```

这样每个 Rayon worker 在本地维护一个 best candidate 和 candidate count，最后 reduce 合并。

收益：

- 不再分配 `candidates: Vec<_>`；
- 保留并行遍历窗口；
- 仍然统计合格候选 base 数量；
- 最终选择逻辑与原先保持一致，包括 `tie_epsilon` 和 chain length tie-breaker。

### 7.5 Release 模式 benchmark

为什么做：

debug 测试可以验证正确性，但性能分析应该尽量使用 release 模式。为了对比修改前后，创建了一个临时 baseline worktree：

```text
C:\tmp\git-internal-benchmark-baseline
```

baseline 使用仓库 `HEAD`，代表本次优化前的代码。由于临时 worktree 中 LFS 文件一开始是 pointer 文本，第一次运行失败，错误为：

```text
InvalidPackHeader("118,101,114,115")
```

这是因为 pack 文件内容实际是 LFS pointer，以 `version ...` 开头。随后将当前工作区可用的测试 pack 数据复制到 baseline worktree 中，再重新运行 benchmark。

benchmark 命令：

```bash
cargo test --release --lib internal::pack::encode::tests::test_pack_encoder_large_file_with_delta -- --exact --nocapture
```

为了尽量不混入编译时间，先分别执行过：

```bash
cargo test --release --lib test_pack_encoder_large_file_with_delta --no-run
```

同时用 PowerShell 轮询 `cargo` 和 `git_internal*` 进程的 `WorkingSet64`，记录近似峰值内存。

结果对比：

| 版本 | test executed | pack size | compression rate | space saved | 近似 peak WorkingSet |
| --- | ---: | ---: | ---: | ---: | ---: |
| baseline HEAD | 726.98ms | 51,566,925 | -0.08% | 0 bytes | 155.60 MB |
| 当前优化后 | 720.89ms | 51,515,728 | 0.02% | 9,158 bytes | 128.51 MB |

观察：

- 编码耗时基本持平，当前版本略低；
- pack size 更小，主要来自 `window_size` 真实生效后 delta 选择发生变化；
- 近似 WorkingSet 峰值下降约 27 MB；
- 当前版本输出了 delta 统计：

```text
delta encode stats: objects=5000 deltas=5 candidate_bases=5 avg_chain_len=1.00
```

注意：

该 benchmark 是一次运行结果，不是严格统计学基准。更严谨的 benchmark 应该固定机器负载，多次运行取均值/方差，并区分编码阶段和解码校验阶段的内存。

### 7.6 追加验证结果

运行格式检查：

```bash
cargo fmt --check
```

结果：

- 通过；
- 仍有项目现有 rustfmt nightly-only 配置 warning。

运行窗口测试：

```bash
cargo test --lib test_try_as_offset_delta_respects_window_size
```

结果：

```text
1 passed; 0 failed
```

运行 AI 类型拒绝测试：

```bash
cargo test --lib test_pack_encoder_rejects_unencodable_ai_type
```

结果：

```text
2 passed; 0 failed
```

运行完整 PackEncoder 相关测试：

```bash
cargo test test_pack_encoder
```

结果：

```text
14 passed; 0 failed
```

说明：

- pack 编码、delta 编码、zstdelta 编码、idx 输出、pack 解码校验均通过；
- 新增 `window_size` 专门测试不在 `test_pack_encoder` 过滤条件内，已单独运行通过。

### 7.7 本轮追加任务结论

本轮追加任务全部完成：

- `window_size` 有了专门单元测试；
- `try_as_offset_delta` 有了 delta 统计；
- release benchmark 已完成并记录结果；
- 窗口从完整 `Entry` 改为轻量 `DeltaBase`，减少深拷贝；
- 候选 base 选择从 `Vec` collect 改为并行 fold/reduce；
- 相关测试均通过。
