# Chapter 8 - 对象

Created by : Mr Dk.

2020 / 06 / 01 19:03

Nanjing, Jiangsu, China

---

Redis 利用之前章节中的数据结构实现了一个类型系统，并在命令执行前，根据对象的类型来判断对象能否执行该命令。另外，可以针对不同的使用场景，为对象设置不同的底层数据结构，从而优化不同场景下的使用效率。

## Object Type and Encoding

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    void *ptr; // 指向底层数据结构

    // ...
} robj;
```

对象类型有：

* `REDIS_STRING` - 字符串对象
* `REDIS_LIST` - 列表对象
* `REDIS_HASH` - 哈希对象
* `REDIS_SET` - 集合对象
* `REDIS_ZSET` - 有序集合对象

而 `encoding` 属性记录了底层数据结构 `ptr` 具体由什么数据结构实现。Encoding 可以有：

* `REDIS_ENCODING_INT`
* `REDIS_ENCODING_EMBSTR` - embstr 编码的 SDS 字符串
* `REDIS_ENCODING_RAW` - SDS 字符串
* `REDIS_ENCODING_HT` - 字典
* `REDIS_ENCODING_LINKEDLIST` - 双向链表
* `REDIS_ENCODING_ZIPLIST` - 压缩列表
* `REDIS_ENCODING_INTSET` - 整数集合
* `REDIS_ENCODING_SKIPLIST` - 跳跃表和字段

每种对象类型都至少使用了两种不同的编码实现。Redis 可以根据不同的使用场景来为对象设置不同的编码，极大提升了灵活性和效率。

Redis 中所有的 key 都是字符串对象，而 value 可以是任意对象。当称呼一个数据库键为 XX 键时，实际上指的是 value 对应 XX 类型的对象。

## Redis String

字符串对象的编码有三种：

* int
* raw
* embstr

其中，int 对应一个可以表示为数字的字符串。那么直接将这个数字存储在 `ptr` 所占用的空间中 (不当指针用了)；如果字符串的长度 > 39B，那么使用 SDS 来保存，编码类型为 raw。

如果字符串的长度 < 39B，使用专门用于优化短字符串的 embstr 编码方式。这种编码方式下，redisObject 的内存空间与其中 `ptr` 指针指向的 SDS 内存空间将通过一次内存分配直接分配完毕：

* 只需要一次内存分配
* 只需要一次内存回收
* 对象和字符串值保存在连续的内存中，可以利用缓存带来的性能优势

### Encoding Conversion

三种类型编码的字符串，在经过字符串修改之后，可能必须要进行编码转换。比如，当一个 int 编码的字符串被 append 了一些字母，或是 embstr 被 append (embstr 编码的字符串是只读的)，字符串的编码都会转换为 raw。

## List Object

列表对象的编码可以为：

* ziplist
* linkedlist

### Encoding Conversion

* 列表中所有字符串元素的长度都 < 64B
* 列表中保存的元素数量 < 512 个

当这两个条件无法被满足时，就只能使用双向链表编码。这两个阈值可以在配置中修改。

## Hash Object

哈希对象的编码方式可以为 ziplist 或 hash table。

如果使用 ziplist 编码，那么 key 和 value 将会作为紧挨着的两个结点加入到压缩列表的表尾。

如果使用 hash table，那么 key 对象将会被存放在 hash table 数组中，value 对象将会被 hash table 结点的指针引用。

### Encoding Conversion

* 所有 key 和 value 的字符串长度都 < 64B
* 键值对数量 < 512

当这两个条件无法被满足时，就只能使用 hash table 编码。这两个阈值可以在配置中修改。

## Collection Object

集合对象的编码为 intset 或 hash table。

### Encoding Conversion

* 集合中所有元素都是整数值
* 集合中保存的元素数量不超过 512 个

不能满足以上条件的集合只能使用 hash table 编码。

## Ordered Collection Object

有序集合对象的编码只能使用 ziplist 或 skiplist。

当使用 ziplist 保存时，两个紧挨着的结点分别用于保存元素的 member 和 score。集合元素按 score 从小到大进行排序，score 较大的元素靠近表尾。

而 skiplist 编码使用以下结构作为底层实现：

```c
typedef struct zset {
    zskiplist *zsl;
    dict *dict;
} zset;
```

其中，`zsl` 跳跃表结构按照 score 从小到大保存了所有集合元素 - 跳跃表结点的 `obj` 属性保存 member，`score` 属性保存分值；另外，`dict` 结构以 key-value pair 的形式保存了每一个集合元素：

* key 对应元素 member
* value 对应元素 score

这样可以以 O(1) 的时间复杂度查找 member 的 score。

另外，这两个数据结构通过指针共享相同的元素内存，而不是引用不同的副本。

> 理论上，这种复合实现方式的性能会比任意一种单独实现的方式要好。跳跃表保证了 **有序性** 以及 **范围查询** 的性能；而字典则保证了能以 O(1) 的时间复杂度查询 score，否则跳跃表就只能以 O(n * log(n)) 来查询元素的 score。

### Encoding Conversion

* 有序集合保存的元素个数 < 128
* 有序集合保存的元素 member 的长度都 < 64B

如果不能满足这两个条件，那么有序集合对象只能使用 skiplist 编码。

---

## Type Check

为了保证只有指定类型的值可以执行某些特定命令，Redis 首先会检查输入的类型是否正确 - 由检查 `redisObject` 结构体的 `type` 属性实现。

## Polymorphism

Redis 根据对象的编码方式，来选择一个操作的具体实现方式。比如说，对于字符串对象，int 编码和 SDS 编码的字符串，取得其长度的方式显然不同。所以可以认为一些命令是 **多态** 的 - 根据编码方式的不同，相同的对象操作将有不同的行为。

## Reference Counting

Redis 在每个对象的结构体中保存了引用计数：

```c
typedef struct redisObject {
    // ...
    int refcount;
    // ...
} robj;
```

* 创建新对象时，引用计数被初始化为 1
* ++
* --
* 引用计数为 0 时，对象占用的内存将被释放

## Object Sharing

对于可以共享的对象，比如两个 key 指向了相同的 value，共享内存可以有效节约内存。在共享前，需要将被共享对象的引用计数 +1。

目前，Redis 在初始化服务器时，会创建 `0` - `9999` 的字符串对象。当服务器需要使用这些对象时，将使用共享对象，而不再重新创建对象。Redis 只对包含整数值的字符串变量进行共享 (因为验证其它字符串是否相等的开销太大)。

> 此处多么像 Java Integer 类中的 IntegerCache。

## LRU

在对象的结构体中还记录了对象最后一次被程序访问的时间：

```c
typedef struct redisObject {
    // ...
    unsigned lru;
    // ...
}
```

当 Redis 中的一些特定选项被打开，如果服务器占用的内存数超过阈值，那么 LRU 值较高的对象将优先被服务器释放，从而回收内存。

---

