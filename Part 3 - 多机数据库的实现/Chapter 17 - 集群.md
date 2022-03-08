# Chapter 17 - 集群

Created by : Mr Dk.

2020 / 06 / 15 16:15

Nanjing, Jiangsu, China

---

Redis 集群是 Redis 提供的分布式数据解决方案。集群通过分片 (sharding) 来进行数据共享。

> 大致思想与 HDFS 有些类似。

## Node

一个 Redis 集群由多个结点组成。需要将多个独立的结点连接起来，构成一个包含多个结点的集群。一个结点就是一个运行在 **集群模式** 下的 Redis 服务器，在该模式下，结点会继续使用单机模式中使用的服务器组件：

- 文件事件处理器
- 时间时间处理器
- 数据库
- RDB、AOF 持久化
- 发布/订阅模块
- 结点复制模块
- Lua 脚本环境

每个结点会为自身状态建立一个 `clusterNode` 结构体；也会为集群中的其它结点建立该结构体，用来记录其它结点的状态：

```c
struct clusterNode {
    mstime_t ctime; // 结点创建时间
    char name[REDIS_CLUSTER_NAMELEN]; // 结点名称

    int flags; // 结点标识

    uint64_t configEpoch; // 结点当前配置纪元
    char ip[REDIS_IP_STR_LEN]; // 结点 IP 地址
    int port; // 结点端口号

    clusterLink *link; // 连接结点的相关信息

    // ...
};
```

其中，`clusterLink` 结构保存了结点连接的相关信息：

```c
typedef struct clusterLink {
    mstime_t ctime; // 连接的创建时间
    int fd; // TCP socket 描述符
    sds sndbuf; // 输出缓冲区 (等待发送给其它结点的信息)
    sds rcvbuf; // 输入缓冲区 (从其它结点接收到的信息)
} clusterLink;
```

每个结点还保存了一个 `clusterState` 结构，记录了当前结点视角下的集群状态：

```c
typedef struct clusterState {
    clusterNode *myself; // 指向当前结点
    uint64_t currentEpoch; // 集群当前配置纪元
    int state; // 集群状态 (上线 / 下线)
    int size; // 集群中至少处理着一个 slot 的结点数量
    dict *nodes; // 集群结点名单 (key 为结点名字，value 为结点 clusterState 结构体)
} clusterState;
```

将结点添加到集群中使用 `CLUSTER MEET` 命令。收到命令的集群结点 A 与待加入结点 B 进行握手，以确认彼此的存在。

- 结点 A 为结点 B 创建一个 `clusterNode` 结构，并添加到自己的 `clusterState.nodes` 字典中
- 结点 A 向结点 B 的 IP + 端口号，向结点 B 发送一条 `MEET` 消息
- 结点 B 也同样创建 `clusterNode` 维护结点 A 的元信息，返回结点 A `PONG` 信息
- 结点 A 接收到 `PONG` 信息后，知道结点 B 已经接收了自己的信息，再返回结点 B `PING` 信息
- 结点 B 接收到 `PING` 信息后，直到结点 A 已经接收了自己的信息，握手完成

## Slot Dispatch

Redis 通过分片的方式保存数据库中的键值对。通俗来说就是把部分的 key 放在一个结点上，把另一部分的 key 放在另一个结点上。key 的整体空间被称为 **槽 (slot)**，整个数据库被分为 16384 个 slot，每一个 key 都可以映射到其中的一个 slot 上。一个上线状态的集群，应当共同负责处理 16384 slot 中的所有 slot。如果有一个 slot 没有被任何结点负责，那么集群就处于下线状态 (fail)。

结点 `clusterNode` 结构中记录了结点负责处理哪些 slot：

```c
struct clusterNode {
    // ...
    unsigned char slots[16384/8];
    int numslots; // 结点负责处理的 slot 数量
    // ...
}
```

其中的 `slots` 显然又是一个 bitmap，如果某一 bit 为 1，说明结点正在负责处理这个 slot。

结点会不断通过消息通知其它结点自己负责的 slot 有哪些，接收到消息的结点更新 `clusterNode` 中记录的信息。在每个结点的集群状态 `clusterState` 结构体中记录了整个集群 16384 个 slot 的指派信息，每个 slot 都对应一个指向 `clusterNode` 的指针：

```c
typedef struct clusterState {
    // ...
    clusterNode *slots[16384];
    // ...
}
```

上述设计可以高效地进行双向查询：

- 已知 slot，查询其被指派到的结点
- 已知结点，查询其负责的 slots

通过 `CLUSTER ADDSLOTS` 命令对结点负责的 slot 进行修改。显然，在命令的实现中，需要修改结点维护的上述结构体。

## Execute Commands in Cluster

在数据库的 16384 个 slot 都有结点负责后，集群就开始进入上线状态，客户端可以向集群中的结点发送数据命令。接收命令的结点根据 key 计算出 key 所在的 slot 被哪个结点负责：

- 如果 key 对应的 slot 刚好被自身结点负责，那么结点直接执行这个命令
- 如果 key 对应的 slot 被其它结点负责，则向客户端返回一个 `MOVED` 错误，指引客户端重定向到正确的结点

结点计算 key 对应 slot 的算法 (计算 CRC16 校验和)：

```
return CRC16(key) & 16383;
```

然后判断 `clusterState` 结构体中，负责这个 slot 的结点指针是否指向自己。如果不是，则在 `MOVED` 错误中向客户端返回负责这个 slot 的结点的 IP 和端口号。在集群模式的 redis-cli 客户端中，`MOVED` 错误实际上会被隐藏，自动进行结点重定向。

结点还会使用跳跃表来保存集群中 slot 和 key 之间的关系。跳跃表的 score 是 slot 编号，member 是数据库中的 key：

```c
typedef struct clusterState {
    // ...
    zskiplist *slots_to_keys;
    // ...
}
```

## 重新分片

Redis 集群的重新分片操作可以将任意数量的已指派 slot 改为指派到另一个结点，相应的数据也会随即转移。重新分片操作可以在线进行。Redis 的集群管理软件 _redis-trib_ 负责重新分片操作：

- 让目标结点准备好被导入的 slot
- 让源结点准备好 slot 的迁移
- 从源结点获取最多 count 个属于 slot 的 key
- 对于上述每个 key，将 key 原子地迁移到目标结点
- 直到 slot 对应的所有 key 都被迁移成功

## ASK Error

在迁移 slot 的过程中，如果有客户端的数据请求怎么办？

- 源结点首先在自己的数据库中查找，如果能够找到，就直接返回客户端
- 源结点没能在自己的数据库里找到 key，则向客户端返回 ASK 错误，指引客户端重定向到目标结点再次执行命令

源结点检查集群状态的 `clusterState.migrating_slots_to[i]`，查看 key 对应的 slot i 是否正在进行迁移。如果正在迁移，则返回 ASK 错误。假设此时迁移正在进行，slot i 还没有正式由目标结点负责。此时，如果 key 已经被迁移到了目标结点上，那么对目标结点的访问将会导致 `MOVED` 错误。因为目标结点的 `clusterState.importing_slots_from[i]` 显示结点正在导入一个 slot。如果客户端在发送命令之前，提前发送一个 `ASKING` 命令，会打开客户端的 `REDIS_ASKING` 标识，目标结点将破例执行之后的命令一次。`ASKING` 命令带来的效果是 **一次性的**。

因此，在客户端接收到 `ASK` 错误时，需要先向目标结点发送一个 `ASKING` 命令，再重新发送想要执行的命令。

## 复制与故障转移

Redis 集群中的结点也是分主从的。主结点用于负责 slot，从结点用于复制某个主结点。为一个主结点设置从结点后，主从结点分别会维护对方的信息，这些信息也会被传播到集群中。最终，集群中的所有结点都会知道某个从结点正在复制某个主结点。

```c
struct clusterNode {
    // ...
    struct clusterNode *slaveof; // 从结点指向主结点
    // ...
    int numslaves;
    struct clusterNode **slaves; // 主结点指向其所有从结点
    // ...
};
```

集群中的每个结点会定期地向集群中的其它结点发送 `PING` 信息，以检测对方是否在线。如果结点没有在规定的时间内返回 `PONG`，那么结点就被视为进入 **疑似下线状态**，疑似下线的结点会被添加到 `clusterNode` 结构的 `fail_reports` 链表中：

```c
struct clusterNode {
    // ...
    list *fail_reports;
    // ...
};
```

```c
struct clusterNodeFailReport {
    struct clusterNode *node; // 已下线结点
    mstime_t time; // 最后一次从结点收到下线报告的时间
} typedef clusterNodeFailReport;
```

如果集群中半数以上主结点都判断某个主结点进入疑似下线状态，那么该结点将会被标记为已下线，并被集群广播，开始进行故障转移：

1. 下线主结点的所有从结点里，选出一个
2. 被选出的从结点成为新的主结点
3. 新的主结点撤销所有对原主结点的 slot 指派，并将这些 slot 全部指派给自己
4. 新的主结点向集群广播 `PONG`，通知集群自己已接管了原主结点负责的 slot
5. 故障转移完成，新的主结点开始接收对自己负责处理的 slot 的命令请求

新的主结点是通过选举产生的。下线主结点的从服务器发现其主服务器下线后，开始向所有其它主服务器拉票。投票的具体方式与 Sentinel 非常相似，基于配置纪元，进行一轮一轮的投票，直到某个从结点得到了多余半数主结点的投票。
