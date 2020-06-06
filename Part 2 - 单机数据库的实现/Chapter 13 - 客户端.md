# Chapter 13 - 客户端

Created by : Mr Dk.

2020 / 06 / 06 17:09

Nanjing, Jiangsu, China

---

## 客户端状态

Redis 通过 I/O 多路复用来实现与多个客户端建立网络连接。Redis 服务器如何维护所有客户端的状态？

对于每个与服务器连接的客户端，服务器建立了 `redisClient` 结构维护该客户端的信息。这个信息以链表的形式维护在服务器状态结构体中：

```c
struct redisServer {
    // ...
    list *clients;
    // ...
};
```

而每个客户端的状态维护在 `redisClient` 结构体中：

```c
typedef struct redisClient {
    // ...

    int fd;

    robj *name;

    int flags;

    sds querybuf;
    
    robj **argv;
    int argc;

    redisCommand *cmd;

    char buf[REDIS_REPLY_CHUNK_BYTES];
    int bufpos;

    list *reply;

    int authenticated;

    time_t ctime;
    time_t lastinteraction;
    time_t obuf_soft_limit_reached_time;

    // ...
} redisClient;
```

### 套接字描述符

`fd` 记录了客户端正在使用的 socket 描述符。当其为 `-1` 时，代表该客户端是一个伪客户端：

* 载入 AOF 文件还原数据库状态
* 执行 Lua 脚本中包含的 Redis 命令

### 客户端名称

默认情况下，客户端是没有名字的。可以通过命令设置名称，让客户端的身份变得更清晰。如果客户端没有名字，`name` 为 `NULL`；否则指向一个字符串对象。

### 标志

`flag` 记录了客户端的角色、状态等。

### 输入缓冲区

`querybuf` 用于保存客户端发送的命令请求。缓冲区的大小会随着客户端输入的内容动态减小或扩大，如果大小超过 1GB，则该客户端会被服务器关闭。

### 命令请求与参数

`argv` 和 `argc` 保存服务器对输入缓冲区中的命令进行解析后，得到的拆分后的命令及其参数。

> 与从 shell 运行 `main(argv, argc)` 类似。

### 命令的实现函数

服务器根据 `argv[0]` 的值，使 `cmd` 指针指向对应的 `redisCommand` 结构体。这个结构体中保存了命令的实现函数、标志、命令的执行次数、总执行时长等。之后服务器可以通过 `redisCommand`、`argv`、`argc`，来调用命令对应的实现函数。

所有的 `redisCommand` 结构体被维护在一个字典中，称为 **命令表**。字典的 key 是 SDS 字符串，对应了命令的名字；字典的 value 对应 Redis 命令的 `redisCommand` 结构体。

### 输出缓冲区

用于保存执行命令后得到的结果。每个客户端有两个可用的输出缓冲区

* `buf[REDIS_REPLY_CHUNK_BYTES]` 对应一个固定长度的缓冲区，由 `bufpos` 记录已使用的字节数；默认为 16KB
    * 用于保存长度较小的回复
* `reply` 对应一个可变大小的缓冲区，由它来组织一个由多个字符串对象构成的链表
    * 处理 `buf` 放不下的情况

### 身份认证

`authenticated` 记录客户端是否通过了身份认证。

### 时间

* `ctime` 记录了客户端的创建时间，可用于计算连接已经持续了多久
* `lastinteraction` 记录了客户端与服务器最后一次互动的时间，可用于计算客户端空闲了多久
* `obuf_soft_limit_reached_time` 记录了输出缓冲区第一次到达软性限制的时间

---

## 客户端的创建与关闭

在客户端使用 `connect` 连接到服务器后，该客户端的状态就会被添加到服务器状态 `client` 链表末尾。

客户端可能因为多种原因而被关闭：

* 客户端进程结束
* 客户端发送了不符合协议格式的命令请求
* 客户端称为 `CLIENT KILL` 命令的目标
* 客户端空闲时间超过阈值
* 客户端发送的命令请求超过输入缓冲区限制
* 发送给客户端的结果超出输出缓冲区的限制

为了避免发送给客户端的结果过大，占用过多服务器资源，服务器会时刻检查客户端输出缓冲区的大小。

* 硬性限制 - 如果输出缓冲区大小超过了硬性限制，则立刻关闭客户端
* 软性限制 - 如果输出缓冲区大小超过了软性限制，客户端在 `obuf_soft_limit_reached_time` 记录下该时刻
    * 如果客户端一直超出软性限制，持续时间超过阈值，则服务器关闭客户端
    * 如果输出缓冲区的大小在指定时间内不再超出软性限制，则 `obuf_soft_limit_reached_time` 清零

---

## 伪客户端

Redis 服务器在初始化时会创建一个负责执行 Lua 脚本中 Redis 命令的伪客户端，这个客户端在服务器运行期间一直存在，直到服务器被关闭：

```c
struct redisServer {
    // ...
    redisClient *lua_client;
    // ...
}
```

而服务器载入 AOF 文件而创建的伪客户端将在载入完成之后被关闭。

---

