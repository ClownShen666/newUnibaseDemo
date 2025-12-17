# 实验三指导文档

## 任务三 分布式索引构建与查询

分布式索引需在多个节点上协同维护索引结构，支持跨节点高效查询。本实验实现分布式LSM树索引及查询路由功能。

### 任务 3.1 分布式 LSM 树索引构建

基于分片策略构建分布式 LSM 树，索引节点分布在不同存储节点，根节点和元数据集中管理。

需修改DistributedLSMTree类：

```cpp
class DistributedLSMTree {
public:
    // 初始化分布式LSM树，指定分片键和节点分布
    bool Init(const TableInfo& table, const ColMeta& index_col, const ShardingStrategy& strategy);

    // 插入键值对到分布式索引
    bool Insert(const KeyType& key, const Rid& rid, Context* ctx);

    // 删除分布式索引中的键值对
    bool Delete(const KeyType& key, const Rid& rid, Context* ctx);
};
```

Init：根节点存储在主节点，叶子节点按分片策略分布到各存储节点，记录节点与分片的映射。
Insert/Delete：根据键的分片信息路由到目标节点，执行本地索引操作，同步更新根节点元数据。

### 任务 3.2 分布式索引查询路由

根据查询条件定位索引所在节点，聚合多节点结果，返回最终查询结果。

需实现DistributedIndexRouter类：

```cpp
class DistributedIndexRouter {
public:
    // 根据查询键定位目标节点
    std::vector<NodeId> LocateNodes(const KeyType& key, const DistributedLSMTree& index);

    // 执行跨节点索引查询并聚合结果
    std::vector<Rid> Query(const KeyType& key, const DistributedLSMTree& index, Context* ctx);

    // 范围查询：定位所有可能包含目标范围的节点
    std::vector<NodeId> LocateRangeNodes(const KeyRange& range, const DistributedLSMTree& index);
};
```

LocateNodes：基于分片键的哈希 / 范围信息，确定存储目标键的节点。
Query：向目标节点发送查询请求，收集并合并结果（去重、排序）。

### 任务 3.3 索引一致性维护

确保分布式索引在节点故障或数据迁移时的一致性，支持索引修复与同步。

需补全IndexConsistencyManager类：

```cpp
class IndexConsistencyManager {
public:
    // 检测索引不一致（如主从节点索引不匹配）
    bool CheckConsistency(const DistributedLSMTree& index);

    // 修复不一致的索引分片
    bool RepairShard(NodeId node_id, const DistributedLSMTree& index);

    // 数据迁移时同步索引
    bool SyncIndexOnMigration(NodeId src_node, NodeId dst_node, const ShardInfo& shard);
};
```

CheckConsistency：定期比对各节点索引元数据（如键范围、版本号）。
RepairShard：从健康节点同步索引数据到故障恢复节点，重建索引。


## 分数说明

任务点	测试文件	分值
任务 3.1 分布式 LSM 树构建	src/index/distributed_lsm_tree_test.cpp	40
任务 3.2 索引查询路由	src/index/index_router_test.cpp	30
任务 3.3 索引一致性维护	src/index/index_consistency_test.cpp	30

## 编译测试命令
cd build
make distributed_lsm_tree_test
./bin/distributed_lsm_tree_test

make index_router_test
./bin/index_router_test

make index_consistency_test
./bin/index_consistency_test

