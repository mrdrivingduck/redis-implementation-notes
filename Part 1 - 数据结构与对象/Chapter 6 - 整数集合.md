# Chapter 6 - 整数集合

Created by : Mr Dk.

2020 / 06 / 01 16:20

Nanjing, Jiangsu, China

---

整数集合 (intset) 是 Redis 保存整数值集合的抽象数据结构，并保证集合中不会出现重复元素。

## Definition

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
}
```

其中，`contents` 数组的数据类型取决于 `encoding`：

* `INTSET_ENC_INT16` - `int16_t`
* `INTSET_ENC_INT32` - `int32_t`
* `INTSET_ENC_INT64` - `int64_t`

`contents` 内维护集合内的所有元素，以 **有序** 的方式排列。

## Upgrade

当新元素被添加到集合中时，如果新元素的数据长度比当前集合中的长度等级长，将触发集合的升级：

* 扩展整数集合底层数组的空间，并分配新空间
* 将原有数据全部转换为新类型，并添加到正确的位置上，保证有序
* 添加新元素

由于升级时要对底层数组中已有的元素进行转换，所以时间复杂度为 O(N)。

整数集合不支持降级。一旦对数组进行了升级，编码就一直保持升级后的状态。

---

