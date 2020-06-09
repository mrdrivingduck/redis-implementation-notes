# Chapter 18 - 发布与订阅

Created by : Mr Dk.

2020 / 06 / 09 20:53

Nanjing, Jiangsu, China

---

Redis 客户端可以订阅一个或多个频道，称为频道的订阅者，每当其它客户端向被订阅的频道发送消息时，频道的 **所有订阅者** 都会收到这条消息。另外，客户端还可以订阅一个 **频道模式**，当其它客户端向某个频道发布消息时，如果频道与频道模式匹配，那么除了频道订阅者以外，频道模式的订阅者也能接收到消息。

## Subscription

Redis 服务器将所有订阅关系维护在服务器状态的 `pubsub_channels` 字典中：

```c
struct redisServer {
    // ...
    dict *pubsub_channels;
    // ...
};
```

字典的 key 是被订阅的频道名；字典的 value 是一个链表，记录了订阅该频道的所有客户端。当客户端发出订阅请求时，服务器首先查找对应的频道名是否存在，如果存在，那么直接将客户端加入链表；如果不存在，则会新建一个键，并将客户端放到链表的第一个位置。

解除订阅的过程恰恰相反。服务器通过遍历订阅字典，把客户端从频道 key 对应的链表中删除。如果客户端被移除后，频道 key 对应的链表为空，那么这个 key 也将会从字典中删除。

## Pattern Subscription

服务器将所有模式的订阅信息维护在服务器状态中：

```c
struct redisServer {
    // ...
    list *pubsub_patterns;
    // ...
};
```

这个属性是一个链表，其中的每个结点都记录了一个被订阅的 pattern 和对应的客户端：

```c
typedef struct pubsubPattern {
    redisClient *client;
    robj *pattern;
}
```

当客户端请求订阅模式时，将构造一个新的 `pubsubPattern` 结构，并添加到链表尾部；退订时，只需要将该结构从链表中移除。

---

## Publishing Message

当 Redis 客户端执行 `PUBLISH <channel> <message>` 后，服务器将执行如下动作：

1. 将 message 发给 channel 的所有订阅者
2. 将消息发送给所有与 channel 匹配的 pattern 对应的订阅者

第一步，服务器从 `pubsub_channels` 字典中取得该频道对应的客户端链表，遍历链表依次将消息发送给订阅者客户端；第二步，服务器遍历 `pubsub_patterns` 链表，如果 pattern 匹配，也将消息推送给对应的客户端。

---

## Subscription Information

客户端可以通过命令查看服务器中的所有订阅关系。

### PUBSUB CHANNELS

该命令返回服务器当前被订阅的频道。遍历服务器的频道字典即可。如果命令还指定了 pattern，则只返回匹配的频道。

### PUBSUB NUMSUB

返回频道的订阅者数量。服务器通过 `pubsub_channels` 字典，找到频道对应的订阅者链表，然后返回订阅者链表的长度。

### PUBSUB NUMPAT

返回服务器当前订阅模式的数量。服务器直接返回 `pubsub_patterns` 链表的长度。

---

