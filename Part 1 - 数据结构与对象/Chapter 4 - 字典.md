# Chapter 4 - 字典

Created by : Mr Dk.

2020 / 06 / 01 15:47

Nanjing, Jiangsu, China

---

字典是用于保存 key-value pair 的抽象数据结构，其中的每一个 key 都是独一无二的。由于 C 内置没有这样的数据结构，Redis 通过实现一个 hash 表来作为字典的底层实现。

## Definition

首先是 hash 表的定义，表结构本身的定义：

```c
typedef struct dictht {
    dictEntry **table; // Hash 表数组

    unsigned long size; // 表大小 (2^n)
    unsigned long sizemask; // 表大小掩码 == size - 1

    unsigned long used; // 已有结点的数量
} dictht;
```

表结点的定义：

```c
typedef struct dictEntry {
    void *key;

    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    sturct dictEntry *next;
} dictEntry;
```

这里的实现与 JDK 8 中的 HashMap 实现类似。`struct dictht` 中，通过 `table` 指针开辟一个一维的数组。每个元素通过 `sizemask` 映射到一维数组的相应位置。如果该位置已经存在元素，说明出现了 hash 冲突，冲突结点通过 `next` 指针的链地址法处理冲突。

另外，由于冲突链表是一个单向链表，没有表尾指针，因此新加入的冲突结点总是插入到链表的表头位置，这样时间复杂度为 O(1)。

最上层的字典定义：

```c
typedef struct dict {
    dictType *type;
    void *privdata;

    dictht ht[2]; // 哈希表

    int trehashidx; // rehash 索引
}
```

用途不同的字典会有不同类型的函数：

```c
typedef sturct dictType {
    // 计算 hash 的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制 key 的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制 value 的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比 key 的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁 key 的函数
    void (*keyDestructor)(void *privdata, void *key);

    // 销毁 value 的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

## Hash Calculation

关于 hash 值的计算是显而易见的。通过定义好的 hash 函数对 key 运算，将得到的结果与 mask 作与运算，就能得到在 hash table 中的 index。

```c
hash = dict->type->hashFunction(key);
index = hash & dict->ht[x].sizemask;
```

## Rehash

与 JDK HashMap 类似，最复杂的操作应该就是 hash table 的扩容了。当 hash table 的 load factor 超出了一定阈值，就要对表进行扩容，然后对原有的结点重新计算 hash，映射到新的位置上。在字典的 `dict` 结构体的定义中，为什么要定义两个 hash table `ht[2]` 呢？

在平时，hash table 只使用 `ht[0]`；当 rehash 时，hash table 使用 `ht[1]` 分配内存。分配内存时有两种情况，但需要保证分配的内存对应的结点数为 **2 的整数次幂**：

- 扩容
- 收缩

内存分配完毕后，将 `ht[0]` 上的点 **逐步迁移** 到 `ht[1]` 中，直到 `ht[0]` 称为一个空表。将 `ht[1]` 赋值给 `ht[0]`，使 `ht[1]` 为下一次 rehash 做准备。

## Trigger Condition of Rehash

Rehash 在什么条件下才会触发呢？Redis 中定义了一个负载因子的概念：

```c
load_factor = ht[0].used / ht[0].size;
```

在什么条件下触发 rehash 呢？

1. 扩容 - 在 `BGSAVE` 或 `BGREWRITEAOF` 命令执行期间，负载因子超过 5
2. 扩容 - 否则，负载因子超过 1
3. 收缩 - 负载因子小于 0.1

> 为什么在 `BGSAVE` 和 `BGREWRITEAOF` 命令执行期间，rehash 的触发条件更加苛刻呢？
>
> 在这两个命令执行期间，Redis 会创建子进程，而大部分 OS 使用 copy-on-write 机制来优化子进程。在这期间，如果进行 rehash，那么将带来页面写操作，OS 不得不为写页面的进程重新分配内存。因此，将负载因子阈值调高一些，使 rehash 较难发生。

## Gradual Rehash

Rehash 期间，从 `ht[0]` 到 `ht[1]` 的迁移不是一次性完成的，否则可能会因为一次性迁移量太多，而使服务器停止服务。

Redis 通过 `rehashidx` 标志字典的 rehash 状态。在 rehash 进行期间，每次对字典进行增删改查时，首先到 `ht[0]` 上查找，如果找不到再找 `ht[1]`。在完成相应操作的同时，顺便将 `ht[0]` 上的结点迁移到 `ht[1]` 上 (但是新插入结点就只会在 `ht[1]` 上)。直到 `ht[0]` 成为空表。

这种方法，避免了集中式 rehash 带来的庞大计算量，将迁移成本均摊到了每次对字典进行增删改查上。666666 👍

---

与 JDK 中的 HashMap 可以说是各有千秋吧，挺有趣的。
