# MVCC Update / Delete Executor 修改笔记

这份笔记对应这次把 `UpdateExecutor` 和 `DeleteExecutor` 从 Project 3 风格改到 Project 4 MVCC 风格的实现。重点不是背代码，而是记住每一步在保护什么语义。

## 1. 和原始 Update / Delete 的最大差别

原来的实现大致是单版本逻辑：

```cpp
while (child->Next(...)) {
  // DELETE: 直接把 tuple meta 标成 deleted
  // UPDATE: 删除旧 tuple，插入新 tuple，维护 index
}
```

MVCC 版本不能这样做。现在的核心目标是：

1. 保持同一个 RID 上的版本链。
2. 写入前做 write-write conflict 检查。
3. 第一次修改 RID 时追加 undo log。
4. 同一个事务再次修改同一个 RID 时修改已有 undo log。
5. base tuple 存当前最新版本，旧版本靠 undo log 重建。
6. 发生冲突时 `SetTainted()`，然后抛 `ExecutionException`。

## 2. Pipe Breaker

MVCC 下 update/delete 要先收集目标，再统一写入：

```cpp
std::vector<std::pair<Tuple, RID>> rows;

while (child_executor_->Next(&child_tuples, &child_rids, batch_size)) {
  for (...) {
    rows.emplace_back(child_tuples[i], child_rids[i]);
  }
}

for (const auto &[visible_tuple, rid] : rows) {
  // conflict check + undo log + table update
}
```

这样可以避免边扫描边修改导致扫描结果被自己污染。`UpdateExecutor` 和 `DeleteExecutor` 都应该在第一次 `Next()` 内完成这个流程，`Init()` 只负责初始化 child 和状态位。

## 3. UpdateExecutor 当前流程

`UpdateExecutor::Next()` 的主线：

1. 如果 `has_updated_` 已经是 true，返回 false。
2. 从 child executor 收集所有 `{old_tuple, rid}`。
3. 对每个 RID 构造 `target_tuple`。
4. 调 `GetTupleAndUndoLink(...)` 拿最新 base tuple、meta、undo link。
5. 调 `IsWriteWriteConflict(base_meta, txn)` 检查冲突。
6. no-op update 的判断放在冲突检查之后。
7. 如果是本事务第一次写这个 RID，生成新的 undo log。
8. 如果本事务已经写过这个 RID，修改已有 undo log。
9. 调 `UpdateTupleAndUndoLink(...)` 原子更新 tuple 和 undo link。
10. 调 `UpdateIndexEntries(...)` 维护索引。
11. `AppendWriteSet(...)`。
12. 输出一行 update count。

no-op 判断必须放在 conflict check 后面：

```cpp
auto [base_meta, base_tuple, undo_link] = GetTupleAndUndoLink(...);
if (IsWriteWriteConflict(base_meta, txn)) {
  txn->SetTainted();
  throw ExecutionException("now has write write conflict in update");
}

if (IsTupleContentEqual(old_tuple, target_tuple)) {
  continue;
}
```

否则一个 tuple 已经被别的事务修改过，但表达式算出来刚好和 child 看到的 tuple 一样，就会错误跳过冲突检查。

## 4. DeleteExecutor 当前流程

`DeleteExecutor::Next()` 和 update 一样，先收集再写：

```cpp
std::vector<std::pair<Tuple, RID>> rows_to_delete;
while (child_executor_->Next(&child_tuples, &child_rids, batch_size)) {
  rows_to_delete.emplace_back(child_tuples[i], child_rids[i]);
}
```

写入阶段：

1. 取最新 base tuple、meta、undo link。
2. 检查 write-write conflict。
3. 如果自己已经删过这个 RID，跳过。
4. 新 meta 是 `{txn->GetTransactionTempTs(), true}`。
5. 第一次写 RID：`GenerateNewUndoLog(..., target_tuple=nullptr, ...)`。
6. 重复写 RID：`GenerateUpdatedUndoLog(..., target_tuple=nullptr, ...)`。
7. `UpdateTupleAndUndoLink(...)`。
8. `DeleteIndexEntries(...)`。
9. `AppendWriteSet(...)`。
10. 输出 delete count。

## 5. Undo Log 合并规则

第一次修改 RID：

```cpp
auto log = GenerateNewUndoLog(&schema, &base_tuple, &target_tuple, base_meta.ts_, prev_link);
auto new_link = txn->AppendUndoLog(log);
```

同一个事务再次修改同一个 RID：

```cpp
auto old_log = txn->GetUndoLog(undo_link->prev_log_idx_);
auto updated_log = GenerateUpdatedUndoLog(..., old_log);
txn->ModifyUndoLog(undo_link->prev_log_idx_, updated_log);
```

重点：`GenerateUpdatedUndoLog` 的 modified columns 不能重新用“original tuple vs final tuple”覆盖掉。它应该保留旧 log 已经记录过的列，再追加当前这一步新修改的列：

```cpp
auto changed_col = log.modified_fields_;
auto current_diff = BuildDiffTrue(schema, *base_tuple, *target_tuple);
for (uint32_t i = 0; i < col_num; i++) {
  changed_col[i] = changed_col[i] || current_diff[i];
}
```

这样即使某列后来被改回原值，undo log 里也不会把它删掉。

## 6. 冲突异常

发生 write-write conflict 时，不要只抛普通 `Exception`。测试期望：

```cpp
txn->SetTainted();
throw ExecutionException("now has write write conflict in update");
```

delete 同理。原因是 execution engine 会捕获 `ExecutionException`，然后让 SQL 执行返回失败；普通 `Exception` 会直接冒泡到测试外层。

## 7. Index 维护当前约定

这次加了三个 helper：

```cpp
DeleteIndexEntries(...)
InsertIndexEntries(...)
UpdateIndexEntries(...)
```

`UpdateIndexEntries` 会逐个 index 比较 old key 和 new key，只有 key 变化时才插入新 key：

```cpp
if (IsTupleContentEqual(old_key, new_key)) {
  continue;
}
index->InsertEntry(new_key, rid, txn);
```

注意：MVCC + index 的完整行为后面还需要继续收口。当前这轮完成标准按你的要求只跑 `TxnExecutorTest`，没有把 `txn_index_test` 当作完成标准。

## 8. 本轮验证

本轮按要求只跑了 `TxnExecutorTest` 默认测试：

```bash
./build/test/txn_executor_test
```

结果：

```text
2 tests from TxnExecutorTest ran.
2 tests passed.
10 tests disabled.
```

之前也单独跑过 update/delete 相关的 disabled 用例用于调试，包括 update conflict、insert delete conflict、update with undo log 等；但最终验收口径按这次最新要求，只记录 `TxnExecutorTest` 默认执行结果。

