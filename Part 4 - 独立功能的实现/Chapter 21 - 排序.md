# Chapter 21 - 排序

Created by : Mr Dk.

2020 / 06 / 11 19:48

Nanjing, Jiangsu, China

---

Redis 的 `SORT` 命令可以对列表的 key、集合的 key 或有序集合的 key 进行排序。

## Implementation of SORT <key>

这是 `SORT` 命令最简单的使用方式，用于对某个数字值的 key 进行排序。比如，对一个包含大量数字的列表进行排序。

服务器首先创建一个与被排序的列表长度相同的数组，数组的类型为 `redisSortObject` 结构：

```c
typedef struct _redisSortObject {
    robj *obj; // 被排序的对象

    union {
        double score; // 数值排序权重
        robj *cmpobj; // 字符串排序指针
    } u;

} redisSortObject;
```

其中，`obj` 指针分别指向被排序列表的每一项，使 `obj` 指针与被排序的列表元素构成一一对应的关系。将每个 `obj` 指向的列表项转换为一个 `double` 类型的浮点数保存在 `u.score` 中。

根据 `u.score` 对 `redisSortObject` 数组进行快速排序，然后依次遍历数组的每一项，将每一项 `obj` 对应的列表项返回给客户端。

## Implementation of ALPHA

通过 `ALPHA` 选项，`SORT` 命令可以对包含字符串的 key 进行排序。具体实现与上述类似：

* 创建一个与排序对象长度相同的 `redisSortObject` 数组
* 遍历数组，使数组每一项的 `obj` 指针分别指向待排序对象，`cmpobj` 指向被比较的字符串对象
* 根据 `cmpobj` 对数组进行快速排序
* 遍历数组，依次将数组每一项 `obj` 指针指向的对象返回给客户端

## Implementation of ASC and DESC

默认情况下，`SORT` 服从升序排序，排序后的结果从小到大排列 (`ASC`)。如果需要降序结果，那么就要显式指定 `DESC`。排序的总体流程与上述相同，唯一的区别是，在进行快速排序时，比较函数产生 **升序对比结果** 和 **降序对比结果**。

## Implementation of BY

默认情况下，`SORT` 使用被排序 key 包含的元素作为排序权重。比如，对集合来说，集合中的每个元素就被作为排序权重。另外，可以通过 `BY` 选项让 `SORT` 对某个特定的 field 进行排序，比如：

```
> SADD fruits "apple" "banana" "cherry"
> MSET apple-price 8 banana-price 5.5 cherry-price 7
> SORT fruits BY *-price
```

首先 Redis 会创建一个与 `fruit` 长度相同的排序数组，其中的 `obj` 指针依次指向待排序数据。然后遍历数组，根据 `BY` 给定的 pattern，查找相应的权重 (比如对于 `apple`，返回 `apple-price` 对应的 `8`) 并转换为 `double`。最终执行排序。

## Implementation of ALPHA BY

```
SORT fruits BY *-id ALPHA
```

与上一节的唯一区别是，被排序的权重值是字符串。因此，将查找到的权重值作为字符串进行排序。

## LIMIT

`LIMIT` 选项可以让 `SORT` 命令只返回一部分已排序的元素。具体的做法是，在完成上述的排序数组创建、关联、排序后，只将排序数组指定位置开始的元素返回给客户端。

## GET

通过使用 `GET` 选项，可以让 `SORT` 命令在排序后，只返回 `GET` 指定模式的元素。其中，也需要先完成上述的建立排序数组、关联、排序。最终，遍历排序数组，根据 `obj` 指向的元素和 `GET` 指定的 pattern，过滤出符合条件的 key；然后取得 key 对应的值并返回。

一个 `SORT` 命令可以带有多个 `GET` 选项。

## Implementation of STORE

默认情况下，`SORT` 只向客户端返回结果，但不保存排序结果。通过 `STORE` 选项可以把 `SORT` 的排序结果保存到一个指定的 key 中。

排序的部分还是一致：建立排序数组、关联、排序。然后，Redis 首先判断指定保存的 key 是否存在，如果存在，先删除它；然后遍历排序数组，依次将排序后的元素加入到指定 key 对应的数据结构中 (比如列表)。最终向客户端返回排序后的结果。

---

## Order of Options

上述选项的执行顺序如何？典型的 `SORT` 执行顺序：

1. 排序 (`ALPHA` `ASC`/`DESC` `BY`)
2. 限制排序结果的长度 (`LIMIT`)
3. 获取外部 key (`GET`)
4. 保存排序结果 (`STORE`)
5. 向客户端返回排序结果

其中，除了 `GET` 以外，其它选项在命令中的位置不会影响最终结果；而只有保证 `GET` 选项的顺序不变才能保证最终的排序结果不变。

---

