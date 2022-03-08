# Chapter 16 - Sentinel

Created by : Mr Dk.

2020 / 06 / 14 10:42

Nanjing, Jiangsu, China

---

Sentinel 是 Redis 的高可用性 (hign availability) 解决方案。由一个或多个 Sentinel 实例组成的系统可以监视任意多个主服务器、从服务器。当被监视的主服务器下线时，自动将该服务器的某个从服务器升级为新的主服务器，代替原有的主服务器继续处理请求。

- Sentinel 首先挑选一个下线服务器的从服务器，升级为新的主服务器
- 向原主服务器下属的其它从服务器发送复制指令，复制新的主服务器 (故障转移)
- Sentinel 继续监视已下线的原主服务器，如果其重新上线，则将其设置为新的主服务器的从服务器

## Sentinel 的启动与初始化

Sentinel 的本质是一个运行在特殊模式下的 Redis 服务器。启动 Sentinel，首先需要与启动服务器类似，初始化一个服务器。然后再将一部分普通 Redis 服务器的代码替换为 Sentinel 专用代码。比如，Sentinel 有着不同于普通 Redis 服务器的命令表。

接下来，服务器初始化一个 Sentinel 的状态结构体，保存了所有与 Sentinel 功能有关的状态：

```c
struct sentinelState {
    uint_64 current_epoch; // 当前纪元 (用于故障转移)
    dict *masters; // Sentinel 监视的所有主服务器
    int tilt; // TILT 模式
    int running_scripts; // 目前正在执行脚本的数量
    mstime_t tilt_start_time; // 最后一次执行时间处理器的时间
    list *scripts_queue; // 需要执行的用户脚本的 FIFO 队列
} sentinel;
```

接下来，初始化 `master` 字典中所有被 Sentinel 监视的主服务器信息。字典的 key 是主服务器的名字，value 是描述 Redis 服务器信息的 `sentinelRedisInstance` 结构体：

```c
typedef struct sentinelRedisInstance {
    int flags; // 实例的类型与当前状态
    char *name; // 实例的名字
    char *runid; // 实例运行 ID
    uint64_t config_epoch; // 纪元，用于故障转移
    sentinelAddr *addr; // 实例地址 (IP + 端口号)

    mstime_t down_after_period; // 被判断为主观下线的时间阈值
    int quorum; // 判断实例客观下线所需的投票数量

    int parallel_syncs; // 故障转移过程中，可以同时对新的主服务器进行同步的从服务器数量

    mstime_t failover_timeout; // 刷新故障迁移状态的最大时限
}
```

上述结构体的初始化基于载入的 Sentinel 配置文件进行。初始化结束后，Sentinel 作为主服务器的客户端，向被监视的主服务器创建网络连接，从而开始监视主服务器的运行。Sentinel 会与其监视的每一个主服务器建立两个连接：

- 命令连接 - 用于向主服务器发送命令，并接收命令回复
- 订阅连接 - 专门用于订阅主服务器的 `__sentinel__:hello` 频道

## 获取主服务器信息

Sentinel 默认以十秒一次的频率，通过 **命令连接**，向主服务器发送 `INFO` 命令。根据命令回复，Sentinel 可以获得两方面信息：

- 主服务器本身的信息
- 主服务器下属的所有从服务器信息 (因此 Sentinel 可以自动发现从服务器)

Sentinel 根据获得的信息，对自身维护的主服务器状态进行更新维护，包括维护 `slaves` 字典中所有从服务器的信息。

## 获取从服务器信息

当 Sentinel 发现主服务器中有新的从服务器出现，除了建立相应的状态结构外，Sentinel 也会创建连接到从服务器的 **命令连接** + **订阅连接**。然后 Sentinel 也会默认以十秒一次的频率向从服务器发送 `INFO` 命令，获取其状态信息，并更新维护自身的结构。

## 向主服务器和从服务器发送信息

默认情况下，Sentinel 会以两秒一次的频率，通过命令连接，向所有被监视的主从服务器发送 `PUBLISH` 命令。该命令会在服务器的 `__sentinel__:hello` 频道中发送信息，信息中包含：

- Sentinel 自身的信息
- 被监视的主服务器信息

## 接收来自主从服务器的频道信息

Sentinel 会通过 **订阅连接** 向服务器发送 `SUBSCRIBE __sentinel__:hello`，对该频道的订阅会持续到 Sentinel 与服务器的连接断开为止。

> Sentinel 通过命令连接向服务器的 `__sentinel__:hello` 频道 **发送** 信息，又通过订阅连接从服务器的这个频道 **接收** 消息。

对于监视同一个服务器的多个 Sentinel，一个 Sentinel 发送的信息将被其它 Sentinel 接收到，用于更新其它 Sentinel 对发送信息的 Sentinel 和服务器的认知。Sentinel 维护的主服务器状态结构中有一个 `sentinel` 字典，保存了同样监视着该主服务器的其它 Sentinel 的信息，如 IP、端口号、运行 ID、配置纪元等。

Sentinel 还会对其发现的另一个 Sentinel 创建命令连接，**不创建** 订阅连接。最终，监视同一主服务器的多个 Sentinel 将形成一个互联网络。

## 检测主观下线状态

默认情况下，Sentinel 以每秒一次的频率向所有命令连接 (主服务器、从服务器、Sentinel) 发送 `PING` 命令，根据配置文件中的 `down-after-milliseconds` 毫秒内连续向 Sentinel 返回无效回复，那么 Sentinel 在其维护的实例结构中标记这个服务器为 **主观下线状态** ("我觉得它已经下线了，你们怎么看？")。

## 检测客观下线状态

Sentinel 将一个主服务器标记为主观下线后，会向同样监视该主服务器的其它 Sentinel 发送询问命令 `SENTINEL is-master-down-by-addr`，以确认这个主服务器是否真的下线了。被询问的 Sentinel 发送回复后，如果统计其它 Sentinel 同意主服务器已下线的数量超过了判断阈值 (类比半数以上的投票)，那么 Sentinel 会将主服务器实例结构的 `SRI_O_DOWN` 标识打开，表示主服务器进入 **客观下线状态** (大家都觉得这个服务器挂了)。

- 主观下线状态是由每个 Sentinel 自行探测得到的结论
- 客观下线状态取决于投票阈值的设定，每个 Sentinel 都可能不一样

## 选举领头 Sentinel

在主服务器被判断为客观下线后，监视该主服务器的各 Sentinel 会进行协商，选举出一个领头 Sentinel 来进行之后的 **故障转移**。所有的 Sentinel 都有被选举为领头 Sentinel 的资格。

- 每一轮选举后，所有 Sentinel 的配置纪元都会 +1
- 每轮选举中，每个 Sentinel 都能投一票
- 每个认为主服务器客观下线的 Sentinel 都会要求其它 Sentinel 投自己一票 (通过自己的运行 ID)
- 进行投票的 Sentinel 的规则是先到先得，谁先找我拉票我就投谁一票，之后的拉票请求被我忽视
- 如果某个 Sentinel 被半数以上的 Sentinel 投票，那么就成为了领头 Sentinel
- 如果给定时间内没有选出领头，那么一段时间后再进行一轮选举，直到选出领头 Sentinel 为止

## 故障转移

领头 Sentinel 将对已下线的主服务器执行故障转移。

首先，在已下线的主服务器的所有从服务器中，挑选出一个状态良好、数据完整的从服务器，然后发送 `SLAVEOF no one`，将这个从服务器转换为主服务器。挑选原则：

- 从服务器正常在线
- 最近与 Sentinel 成功进行过通信
- 与主服务器较晚断开连接 (从服务器保存的数据比较新)
- 偏移量较大的从服务器 (最新数据)
- 运行 ID 最小 (?)

选出新的主服务器后，领头 Sentinel 会让所有的从服务器复制新的主服务器。

最终，将已下线的主服务器设置为新主服务器的从服务器。当这个断线服务器重新上线时，Sentinel 会让其从新的主服务器拷贝副本。
