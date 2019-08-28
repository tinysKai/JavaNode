## ES-集群知识

#### 集群

ES隐藏的分布式机制

- 选主(多数派 (quorum))
- 分片机制
- 集群发现机制(自实现的ZenDiscovery)
- 负载均衡
- 扩容的数据重哈希(原理?)

#### 节点类型

![img](https://i.loli.net/2019/08/27/XK5gj8SwnWqcPr3.jpg)

- A,B,C - 可选主节点
- A,B,D - 数据节点
- E - proxy节点,仅转发请求

```yaml
#通过以下配置，可以产生四种不同类型的 Node：
conf/elasticsearch.yml:
    node.master: true/false
    node.data: true/false

#官方推荐将所有节点设置为可选主节点    
# It is recommended that the unicast hosts list be maintained as the list of master-eligible nodes in the cluster.
```

#### 集群配置

```yaml
# 节点发现配置
conf/elasticsearch.yml:
    discovery.zen.ping.unicast.hosts: [1.1.1.1, 1.1.1.2, 1.1.1.3]

# 选主最小节点数配置
conf/elasticsearch.yml:
    discovery.zen.minimum_master_nodes: 2
```

#### Master选举

当一个节点发现包括自己在内的多数派的 master-eligible 节点认为集群没有 master 时，就可以发起 master 选举。

节点选主PK遵从的原则 

- clusterStateVersion 越大，优先级越高
- clusterStateVersion 相同时，节点的 Id 越小，优先级越高

#### Master选主算法

ES自实现的选主算法基于多数派来选主.但其无法保证一个节点在同一个选举周期只选举一个主,因此一个节点可能选出两个节点造成脑裂.(raft算法引入周期[term]概念来解决一个选举周期只有一票,raft算法选主若最后发现两个主则比较term).当然，这种脑裂很快会自动恢复，因为不一致发生后某个 master 再次发布 cluster_state 时就会发现无法达到多数派条件，或者是发现它的 follower 并不构成多数派而自动降级为 candidate 等。

#### 集群扩容

当新添加的节点仅为数据节点时

```yaml
# 扩容配置 
conf/elasticsearch.yml:
    cluster.name: es-cluster # 集群名
    node.name: node_Z #新添加的节点名
    discovery.zen.ping.unicast.hosts: ["x.x.x.x", "x.x.x.y", "x.x.x.z"] # 原集群中的可选主节点列表
```

当新添加的节点可参与选主时,需避免脑裂,重新配置一个最小节点数

```json
//修改最小节点数
curl -XPUT localhost:9200/_cluster/settings -d '{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 3
    }
}'
```

#### 自实现选主协议与Raft对比

**相同点**

1. 多数派原则：必须得到超过半数的选票才能成为 master。
2. 选出的 leader 一定拥有最新已提交数据：在 raft 中，数据更新的节点不会给数据旧的节点投选票，而当选需要多数派的选票，则当选人一定有最新已提交数据。在 es 中，version 大的节点排序优先级高，同样用于保证这一点。

**不同点**

1. 正确性论证：raft 是一个被论证过正确性的算法，而 ES 的算法是一个没有经过论证的算法，只能在实践中发现问题，做 bug fix，这是我认为最大的不同。
2. 是否有选举周期 term：raft 引入了选举周期的概念，每轮选举 term 加 1，保证了在同一个 term 下每个参与人只能投 1 票。ES 在选举时没有 term 的概念，不能保证每轮每个节点只投一票。
3. 选举的倾向性：raft 中只要一个节点拥有最新的已提交的数据，则有机会选举成为 master。在 ES 中，version 相同时会按照 NodeId 排序，总是 NodeId 小的人优先级高。

#### 参考

[文章](https://zhuanlan.zhihu.com/p/35291900)