# Chapter 23 - 慢查询日志

Created by : Mr Dk.

2020 / 06 / 13 14:08

Nanjing, Jiangsu, China

---

Redis 的慢查询日志功能用于记录执行时间超过给定时长的命令请求。用户可以通过慢查询日志来监视和优化查询速度。

- `slowlog-log-slower-than` 指定执行时间超过该阈值的命令会被记录
- `slowlog-max-len` 指定服务器最多保存多少条慢查询日志

与该功能相关的数据结构：

```c
struct redisServer {
    // ...
    long long slowlog_entry_id; // 下一条慢查询日志 id
    list *slowlog; // 保存了所有慢查询日志的链表

    long long slowlog_log_slower_than;
    unsigned long slowlog_max_len;
    // ...
}
```

其中，第一条慢查询日志的 id 为 `0`，之后每创建一条日志 id + 1。慢查询日志的链表结构定义：

```c
typedef struct slowlogEntry {
    long long id;
    time_t time; // 命令执行时的时间
    long long duration; // 执行命令消耗的时间
    robj **argv; // 命令与命令参数
    int argc; // 命令与命令参数的数量
} slowlogEntry;
```

每次执行命令前后，程序都会记录微秒格式的 UNIX 时间戳，两个时间戳的差就是服务器执行命令所耗费的时长。如果超出了设定阈值，就产生一条新的慢查询日志加入链表；如果链表长度超出阈值，则删除最早的慢查询日志。
