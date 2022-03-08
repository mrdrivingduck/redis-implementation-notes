# Chapter 3 - 链表

Created by : Mr Dk.

2020 / 06 / 01 14:24

Nanjing, Jiangsu, China

---

## Definition

链表结点 (`adlist.h/listNode`)：

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```

显然，由 `prev` 和 `next` 指针就能把结点串成一个双向链表。另外，有一个链表头结点维护链表的整体信息会更方便 (`adlist.h/list`)：

```c
typedef struct list {
    listNode *head;
    listNode *tail;

    unsigned long len;

    void *(*dup)(void *ptr); // 结点复制函数
    void (*free)(void *ptr); // 结点释放函数
    int (*match)(void *ptr, void *key); // 结点值比较函数
}
```

关于链表的一些特性总结：

- 双向链表
- 无环 - 表头结点的 `prev` 和表尾结点的 `next` 都指向 NULL
- 获取表头结点、表尾结点、链表长度的时间复杂度都是 O(1)
- 三个函数指针用于多态 (！这里也用到了)
