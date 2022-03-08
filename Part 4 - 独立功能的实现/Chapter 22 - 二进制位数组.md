# Chapter 22 - 二进制位数组

Created by : Mr Dk.

2020 / 06 / 13 13:38

Nanjing, Jiangsu, China

---

Redis 提供四个命令用于操作位数组 (与 bitmap 类似)。

## Representation of Bit Array

Redis 使用字符串对象 (SDS) 来表示 bit array。从 SDS 的 `[0]` 的第一个 bit 开始存放最低位。每次以字节为单位对位数组进行扩展。

## Implementation of GETBIT

`GETBIT` 返回 bit array 在指定 offset 上的二进制位的值：

- 首先，计算 `offset / 8` 得到对应的 bit 在哪一个字节上
- 计算 `offset % 8` 得到对应的 bit 在字节中的哪个 bit 上
- 返回这个 bit 上的数值

所有计算可以在常数时间内完成，因此算法复杂度为 `O(1)`。

## Implementation of SETBIT

`SETBIT` 用于设置 bit array 在 offset 偏移量上的 bit：

- 计算 `offset / 8` 得到 bit 在哪一个字节上
- 如果这个字节已经超出了 SDS 的长度，则需要将 SDS 字符串扩展到这个长度 (SDS 的空间预分配策略还会额外分配 2B 的空闲空间)
- 计算 `offset % 8` 得到对应的 bit 在字节中的哪个 bit 上
- 首先保存这个 bit 上的旧值，然后在这个 bit 上设置新值
- 返回旧值

## Implementation of BITCOUNT

`BITCOUNT` 命令用于统计给定的 bit array 中值为 1 的 bit 的数量。

显然，最简单的算法是遍历整个 bit array，统计其中值为 1 的 bit 数量。但是这个算法的效率与 bit array 的长度有关，可能效率会非常低。

**查表算法** 是一种以空间换时间的优化操作：比如，提前将一个字节中所有可能的位排列，以及对应的 `1` 个数放到一张表中。那么，对于一个字节 (8-bit)，只需要查一次表就能得到 8 个 bit 中有多少个 1：

| Bit Array | Count of `1` |
| --------- | ------------ |
| 0000 0000 | 0            |
| 0000 0001 | 1            |
| 0000 0010 | 1            |
| ...       |              |
| 1111 1110 | 7            |
| 1111 1111 | 8            |

表所支持的位数越多，表空间就越大，但是可以节约的时间就越多。这种典型的以空间换时间的策略是一种折衷：

1. 表空间不能超过服务器所能接受的内存开销
2. 表越大那么 CPU cache miss 的几率也就越高，最终也会影响效率

因此目前只能考虑 8-bit 的表 (数百 B) 或 16-bit 的表 (数百 KB)。一个 32-bit 的表需要 (10GB+)。

对于这个问题 (计算 Hamming Weight) 效率最高的算法是 _variable-precision SWAR_ 算法，可以在常数时间内计算多个字节的 Hamming Weight，并且不需要额外内存。但是具体流程没看懂。。。

Redis 在实现中使用了查表算法和 variable-precision SWAR 算法：

- 查表算法的 key 为 8-bit
- variable-precision SWAR 算法每次循环载入 128-bit，调用四次算法来计算

执行 `BITCOUNT` 时，程序根据剩余未处理的二进制位数量来决定使用哪个算法：

- 如果 >= 128-bit，则使用 variable-precision SWAR 算法
- 如果 < 128-bit，则使用查表算法

## Implementation of BITOP

`BITOP` 命令对两个 bit array 直接进行 `&` / `|` / `^` / `~` 操作，C 语言的位操作直接支持这些运算。Redis 首先会创建一个空白的 bit array 用于保存结果，然后直接进行运算。
