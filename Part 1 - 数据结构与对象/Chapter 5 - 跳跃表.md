# Chapter 5 - 跳跃表

Created by : Mr Dk.

2020 / 06 / 01 18:36

Nanjing, Jiangsu, China

---

跳跃表 (skiplist) 是一种 **有序数据结构**，在每个结点中维持了多个指向其它结点的指针 (空间开销)，从而达到快速访问。跳跃表支持平均 O(log(N))，最坏 O(N) 的查找复杂度。

在大部分情况下，跳跃表的查找效率与平衡树相当，但实现与维护比平衡树更加简单。关于书本中的跳跃表我觉得讲得不是很透彻，[这篇博客](https://www.jianshu.com/p/43039adeb122) 中的图不错。大致含义是，在普通链表上建立一级索引、二级索引......索引层数越多，空间开销越多，查找越快。

## Node Definition

```c
typedef struct zskiplistNode {
    struct zskiplistNode *backward; // 后向指针
    double score; // 分值 (顺序)
    robj *obj;

    // 层
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span; // 跨度 
    } level[];
} zskiplistNode;
```

创建 skiplist 时，根据 *幂次定律 (Power Law)* (越大的数出现的概率越小) 在 1-32 之间选择一个数作为 `level[]` 数组的大小，即索引高度。

* `level[]` 中的每一层都有一个前向指针 `forward`，用于从表头方向访问表尾方向的结点
* `span` 跨度实际上是用于计算排位，而不是用于遍历
* 后退指针每次只能回退一个结点 (因为只有一层)

跳跃表中的内容分为：

* `obj` - 成员对象 (指向一个 SDS)
* `score` - 分值 (跳跃表的排序基准，从小到大)

成员对象必须是唯一的，`score` 的值可以相同，相同时按 `obj` 对应 SDS 的字典序排序。

## Table Definition

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;

    unsigned long length; // 表中结点数量

    int level; // 表中索引的最高层数
} zskiplist;
```

其中，表头结点不算在 `length` 和 `level` 中。

---

