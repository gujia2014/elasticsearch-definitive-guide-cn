## 添加索引

为了将数据添加到Elasticsearch，我们需要**索引(index)**——一个存储关联数据的地方。实际上，索引只是一个用来指向一个或多个**分片(shards)**的**“逻辑命名空间(logical namespace)”**.

一个**分片(shard)**是一个最小级别**“工作单元(worker unit)”**,它只是保存索引中所有数据的一小片。在接下来的《深入分片》一章，我们将详细说明分片的工作原理，但是现在只要知道分片是一个单一的Lucene实例既可，并且它本身就是一个完整的搜索引擎。我们的文档存储和被索引在分片中，但是我们的程序不知道如何直接与它们通信。取而代之的是，他们直接与索引通信。

分片用于Elasticsearch在你的集群中分配数据。想象把分片当作数据的容器。文档存储在分片中，然后分片分配给你集群中的节点上。当你的集群扩容或缩小，Elasticsearch将会自动在你的节点间迁移分片，以使集群保持平衡。

分片可以是**主分片(primary shard)**或者**复制分片(replica shard)**。你索引中的每个文档属于一个单独的主分片，所以主分片的数量决定了你最多能存储多少数据。

> 理论上主分片对存储多少数据没有限制，限制取决于你实际的使用情况。碎片的最大容量完全取决于你的使用状况：硬件存储的大小、文档的大小和复杂度、如何索引和查询你的文档，以及你期望的响应时间。

复制分片只是主分片的一个副本，它用于提供数据的冗余副本，在硬件故障之后提供数据保护，同时服务于像搜索和检索等只读请求。

主分片的数量会在其索引创建完成后修正，但是复制分片的数量会随时变化。

让我们在集群中一个空节点上创建一个叫做`blogs`的索引。一个索引默认指派5个主分片，但是为了演示的目的，我们只指派3个主分片和一个复制分片（每个主分片有一个复制分片对应）：

```Javascript
PUT /blogs
{
   "settings" : {
      "number_of_shards" : 3,
      "number_of_replicas" : 1
   }
}
```

附带索引的单一节点集群：
![附带索引的单一节点集群](https://raw.githubusercontent.com/looly/elasticsearch-definitive-guide-cn/master/images/elas_0202.png)

我们的集群现在看起来像**单节点集群(cluster-one-node)**——三个主分片都被分配到`Node 1`。如果我们现在想检查**集群健康(cluster-health)**，我们将见到以下信息：

```Javascript
{
   "cluster_name":          "elasticsearch",
   "status":                "yellow", <1>
   "timed_out":             false,
   "number_of_nodes":       1,
   "number_of_data_nodes":  1,
   "active_primary_shards": 3,
   "active_shards":         3,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     3 <2>
}
```

- <1> 集群的 `status` 现在是 `yellow`.
- <2> 我们的三个复制分片还没有被分配到节点上。

集群的健康状况`yellow`意味着所有的**主分片(primary shards)**启动并且运行了——集群已经可以成功的接受任意请求——但是**复制分片(replica shards)**还没有全部可用。事实上所有的三个复制分片现在是`unassigned`（未分配）状态——它们还未被分配给节点。在同一个节点上保存相同的数据副本是没有必要的，如果这个节点故障了，那所有的数据副本也会丢失。

现在我们的集群已经功能完备，但是依旧存在因硬件故障而导致的数据丢失的风险。

