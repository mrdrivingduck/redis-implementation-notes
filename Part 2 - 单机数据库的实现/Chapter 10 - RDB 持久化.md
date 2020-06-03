# Chapter 10 - RDB 持久化

Created by : Mr Dk.

2020 / 06 / 03 10:14

Nanjing, Jiangsu, China

---

RDB 持久化功能可以将 Redis 内存中的数据库状态保存到磁盘中。具体做法是生成一个压缩的二进制 RDB 格式文件，通过这个文件，可以还原当时的数据库状态。

## SAVE and BGSAVE

`SAVE` 和 `BGSAVE` 两条命令都可以用于生成 RDB 文件：

* `SAVE` 阻塞 Redis 服务器进程，直到 RDB 文件创建完毕，期间服务器无法处理任何请求
* `BGSAVE` 创建了一个子进程负责 RDB 文件的写入，服务器进程继续处理请求
    * 期间 `SAVE` / `BGSAVE` / `GBREWRITEAOF` 命令会被拒绝，防止产生竞争条件

由于 AOF 持久化的频率比 RDB 高，因此只有 AOF 持久化功能关闭时，服务器才使用 RDF 文件来还原数据库状态。

## Automatic Saving

Redis 允许用户通过设置服务器的 `save` 选项，让服务器每隔一段时间自动执行一次 `BGSAVE`。`save` 属性中可以设置保存多个条件，只要有一个条件被满足，服务器就会执行 `BGSAVE` 命令。默认的条件如下，含义为 xxx 秒内对数据库进行了 xxx 次修改后触发：

```
save 900 1
save 300 10
save 60 10000
```

这些条件保存在 `redisServer` 结构体中：

```c
struct redisServer {
    // ...
    struct saveparam *saveparams;
    // ...
};
```

这个结构体数组中的每个元素保存了一个 `save` 选项的条件：

```c
struct saveparam {
    time_t seconds; // 秒数
    int changes; // 修改次数
}
```

另外，服务器还维护以下两个相关属性：

```c
struct redisServer {
    // ...
    long long dirty; // 修改计数器，每修改一次 +1
    time_t lastsave; // 上次进行持久化操作的时间
    // ...
}
```

Redis 服务器的周期性操作函数默认 100ms 执行一次，会对 `save` 选项下的每一个条件进行检查：

* 计算当前时间与 `lastsave` 的时间差
* 计算在一定的时间间隔内，Redis 修改次数是否满足条件
* 如果满足条件，那么执行 `BGSAVE`，`dirty` 计数器被重置为 0

## RDB 文件结构

完整的 RDB 文件包含以下部分：

* REDIS (magic value)
* db_version (RDB 第六版)
* databases - 每一个非空数据库
* EOF - RDB 正文内容结束
* check_sum - 对前四个部分计算得到的校验和

其中，每一个非空数据库的文件结构：

* SELECTDB (magic value)
* db_number - 数据库编号
* key_value_pairs

而每个 `key_value_pairs` 的结构：

* 不带过期时间：
    * TYPE (magic value)
    * key
    * value
* 带有过期时间：
    * EXPIRETIME_MS (magic value)
    * ms - 毫秒 UNIX 时间戳
    * TYPE (magic value)
    * key
    * value

> 其中，`TYPE` 用于解析 value 的类型，因为 key 一定是字符串。

---

