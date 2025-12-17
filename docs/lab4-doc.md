# 实验四指导文档

## 任务四 分布式事务处理

分布式事务需保证跨节点操作的 ACID 特性，本实验实现两阶段提交（2PC）协议及分布式锁机制。

### 任务 4.1 分布式事务管理器

管理分布式事务的生命周期（开始、准备、提交、中止），协调各参与节点。

需实现DistributedTransactionManager类：
```c++
class DistributedTransactionManager {
public:
    // 开始分布式事务，生成全局事务ID
    TxnId Begin();

    // 准备阶段：询问所有参与节点是否可提交
    bool Prepare(TxnId txn_id, const std::vector<NodeId>& participants);

    // 提交阶段：通知所有节点提交事务
    bool Commit(TxnId txn_id, const std::vector<NodeId>& participants);

    // 中止阶段：通知所有节点回滚事务
    bool Abort(TxnId txn_id, const std::vector<NodeId>& participants);
};
```

Begin：生成全局唯一事务 ID，记录事务涉及的节点（参与者）。
Prepare：协调者向所有参与者发送准备请求，只有全部同意才能进入提交阶段。

### 任务 4.2 分布式锁管理器

实现跨节点的锁机制（表级 / 行级锁），解决并发冲突，确保隔离性。

需补全DistributedLockManager类：

```cpp
class DistributedLockManager {
public:
    // 申请行级共享锁
    bool LockShared(TxnId txn_id, NodeId node_id, const Rid& rid, int timeout_ms);

    // 申请行级排他锁
    bool LockExclusive(TxnId txn_id, NodeId node_id, const Rid& rid, int timeout_ms);

    // 释放事务持有的所有锁
    bool UnlockAll(TxnId txn_id);

    // 检测死锁并解除（如终止优先级低的事务）
    bool DetectAndResolveDeadlock();
};
```

LockShared/LockExclusive：通过 RPC 向目标节点的本地锁管理器申请锁，支持超时机制。
DetectAndResolveDeadlock：通过构建等待图检测死锁，选择牺牲事务中止以解除阻塞。

### 任务 4.3 事务恢复机制

实现分布式事务的故障恢复，支持日志回放与事务状态修复。

需实现DistributedRecoveryManager类：

```bash
class DistributedRecoveryManager {
public:
    // 节点恢复时，从日志重建事务状态
    bool RecoverNode(NodeId node_id, LogManager* log_manager);

    // 处理事务中断：根据日志完成未提交事务的回滚或提交
    bool HandleInterruptedTxns(LogManager* log_manager);
};
```

RecoverNode：节点重启后，读取本地日志和集群全局日志，恢复未完成的事务状态。
HandleInterruptedTxns：对故障时处于准备阶段的事务，根据协调者日志决定提交或回滚。

### 测试点及分数
任务点	测试文件	分值
任务 4.1 事务管理器	src/transaction/distributed_txn_manager_test.cpp	35
任务 4.2 分布式锁管理	src/transaction/distributed_lock_test.cpp	35
任务 4.3 事务恢复机制	src/transaction/distributed_recovery_test.cpp	30

### 编译测试命令
```bash
cd build
make distributed_txn_manager_test
./bin/distributed_txn_manager_test

make distributed_lock_test
./bin/distributed_lock_test

make distributed_recovery_test
./bin/distributed_recovery_test
```

