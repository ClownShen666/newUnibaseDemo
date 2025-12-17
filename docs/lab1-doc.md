# 实验一指导文档

## 任务一 数据分片基础

数据分片是分布式数据库将数据分散存储在多个节点的核心技术，通过合理分片可提升系统扩展性和查询效率。本实验需实现水平分片、垂直分片及分片一致性维护功能。

### 任务1.1 水平分片实现

水平分片将表中记录按条件分配到不同节点（如按范围、哈希），需实现RangeSharding和HashSharding策略。
需补全HorizontalSharder类接口：

```cpp
class HorizontalSharder {
public:
    // 按范围分片：根据指定列的范围划分记录
    std::map<NodeId, std::vector<Rid>> RangeSharding(
        const TableInfo& table, 
        const ColMeta& shard_col, 
        const std::vector<Range>& ranges
    );

    // 按哈希分片：对指定列哈希后分配到节点
    std::map<NodeId, std::vector<Rid>> HashSharding(
        const TableInfo& table, 
        const ColMeta& shard_col, 
        size_t node_count
    );
};
```

RangeSharding：例如按用户 ID 范围（1-1000 到节点 A，1001-2000 到节点 B），需计算每条记录的分片键所属范围并分配。
HashSharding：使用 MurmurHash 计算分片键的哈希值，取模节点数确定目标节点，确保数据均匀分布。

### 任务1.2 垂直分片实现

垂直分片将表按列拆分（如高频访问列与低频访问列分离），需维护分片间的关联关系。
需实现VerticalSharder类：

```cpp
class VerticalSharder {
public:
    // 按列组分片：将表的列分组分配到不同节点
    std::map<NodeId, VerticalShardInfo> VerticalSharding(
        const TableInfo& table, 
        const std::vector<std::vector<ColMeta>>& col_groups
    );

    // 生成分片间的关联元数据（如主键映射）
    ShardLinkMeta GenerateShardLink(const TableInfo& table, const std::map<NodeId, VerticalShardInfo>& shards);
};
```

VerticalSharding：需确保每个分片包含主键列（或关联列），以便跨分片查询时关联数据。
GenerateShardLink：记录各分片的列信息及关联键，支持后续跨分片连接。

### 任务1.3 分片一致性维护

当节点扩缩容或数据分布失衡时，需动态调整分片，确保一致性。

需补全ShardRebalancer类：        

```cpp
class ShardRebalancer {
public:
    // 检测分片失衡（如某节点数据量超过阈值）
    bool DetectImbalance(const std::map<NodeId, ShardStats>& shard_stats, double imbalance_threshold);

    // 重新平衡分片：迁移部分数据到其他节点
    std::map<NodeId, std::vector<Rid>> RebalanceShards(
        const std::map<NodeId, ShardInfo>& current_shards, 
        const std::vector<NodeId>& new_nodes
    );
};
```

DetectImbalance：基于节点数据量、负载等指标判断是否需要重平衡。
RebalanceShards：迁移数据时需加锁防止不一致，并更新分片元数据。


## 实验计分

在本实验中，每个任务对应一个单元测试文件，每个测试文件中包含若干测试点。通过测试点即可得分，满分为100分。

测试文件及测试点如下：

| 任务点                 | 测试文件                                 | 分值 |
| ---------------------- | ---------------------------------------- | ---- |
任务 1.1 水平分片实现	src/sharding/horizontal_sharder_test.cpp	35
任务 1.2 垂直分片实现	src/sharding/vertical_sharder_test.cpp	30
任务 1.3 分片一致性维护	src/sharding/shard_rebalancer_test.cpp	35

编译生成可执行文件进行测试：

```bash
cd build
make horizontal_sharder_test
./bin/horizontal_sharder_test

make vertical_sharder_test
./bin/vertical_sharder_test

make shard_rebalancer_test
./bin/shard_rebalancer_test
```
