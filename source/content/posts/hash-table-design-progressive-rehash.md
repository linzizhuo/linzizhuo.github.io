---
title: "Hash 表复习笔记：结构体设计与渐进式 Rehash"
date: 2026-07-24
tags: ["Redis", "数据结构", "C", "Hash表", "源码解析"]
math: false
---

> 前置知识（hash 函数、冲突原理等）略过，本文聚焦结构体设计和 rehash 策略。

---

## 1. 拉链法结构体设计

```c
// 哈希表节点
typedef struct dictEntry {
    hash_t hash;            // 缓存的 hash 值，rehash 搬迁时无需重算
    void *key;              // 键
    void *val;              // 值
    struct dictEntry *next; // 指向下一个冲突节点（单链表）
} dictEntry;

// 哈希表
typedef struct dictht {
    dictEntry **table;      // 桶数组（指针数组），每个桶指向一条链表
    unsigned long size;     // 桶数量（始终为 2 的幂）
    unsigned long sizemask; // size - 1，用于快速取模：hash & sizemask
    unsigned long used;     // 已存储的 entry 数量
} dictht;

// 顶层 dict，支持渐进式 rehash
struct dict {
    dictType *type;         // 虚函数表（hash、keyCompare、keyFree、valFree 等）
    dictht ht[2];           // 双表：ht[0] 当前使用，ht[1] rehash 目标表
    long rehashidx;         // rehash 进度：-1 表示未在 rehash，>=0 表示当前搬迁到的桶下标
    unsigned pauserehash;   // rehash 暂停计数器（>0 时暂停，用于某些需要稳定遍历的场景）
};
```

---

## 2. 开放寻址法结构体设计

```c
typedef struct dictEntry {
    hash_t hash;            // 缓存的 hash 值
    void *key;
    void *val;
    // 无 next 指针；需额外标记槽位状态（空/占用/墓碑），通常通过 flag 字段或哨兵值实现
} dictEntry;

typedef struct dictht {
    dictEntry *table;       // entry 数组（注意：不是指针数组，是 entry 本体数组）
    unsigned long size;     // 桶数量（始终为 2 的幂）
    unsigned long sizemask; // size - 1
    unsigned long used;     // 已存储的 entry 数量（不含墓碑）
    unsigned long deleted;  // 墓碑数量
} dictht;
```

---

## 3. sizemask 的作用

当 `size` 为 2 的幂时：

```c
hashval % size  =  hashval & (size - 1)
```

`sizemask = size - 1`，用位与替代取余运算，快一个数量级。这是 hash 表把 size 限定为 2 的幂的主要原因。

---

## 4. 开放寻址法优缺点

### 优点

1. **内存紧凑，缓存友好**：所有 entry 存在连续数组中，一个 cache line 可以命中多个槽位。
2. **省去了 per-entry 的堆分配**：插入不需要 malloc，减少分配器竞争。

### 缺点

1. **负载因子天花板低**：α 接近 1.0 时性能断崖式下跌，实用上限约 0.5~0.7。
2. **删除麻烦**：需要墓碑机制。删除操作让 `deleted` 增加但 `used` 不降，负载因子实际不会改善，大量删除后必须重建才能恢复性能。
3. **渐进式 rehash 不友好**：搬迁留下墓碑 → 旧表探测序列越来越长 → 搬迁期间性能持续恶化（见第 6 节）。

---

## 5. 拉链法优缺点

### 优点

1. **负载因子承受力强**：α 超过 1 仍平滑退化，不会断崖式下跌。
2. **删除简单**：直接从链表中摘除节点，O(1) 完成，无残留。
3. **渐进式 rehash 平稳**：搬迁一个桶，该桶就彻底空了，每次操作的额外开销恒定。

### 缺点

1. **每次访问都是随机内存跳转**：链表节点散落在堆上，cache 不友好。
2. **每次插入都需要分配节点**：malloc 有锁竞争和碎片开销。

---

## 6. 渐进式 Rehash：两种方案的差异

| | 拉链法 | 开放寻址 |
|---|---|---|
| 搬迁单位 | 一个桶（整条链表） | 一个槽位（单个 entry） |
| 搬迁后旧表状态 | 桶指针置 NULL，彻底空 | 留下墓碑，仍参与探测 |
| 搬迁期间查找开销 | 恒定（多查一个新桶） | 递增（墓碑越多探测越长） |
| 搬迁期间旧表变化 | 逐步"萎缩" | 越来越"脏" |
| 全量 rehash 差异 | 不大 | 不大 |

**结论**：拉链法的渐进式 rehash 代价恒定；开放寻址也可以做渐进式 rehash，但搬迁期间旧表性能持续劣化，"用延迟换平滑"的交易不划算。

---

## 7. Redis 为什么选择拉链法？

从两个维度来看：

### 维度一：渐进式 rehash 的必然性

Redis 是单线程模型，任何阻塞操作都会拖慢所有请求。全量 rehash 的延迟尖刺不可接受，因此渐进式 rehash 是必然选择。

而拉链法在渐进式 rehash 中表现平稳——每次搬一个桶，桶空就干净了，每次查找多查一个新桶，开销恒定。开放寻址因为墓碑机制，搬迁越深入旧表越脏，查找开销递增，与"平滑分摊"的目标相悖。

### 维度二：扩容频率

在相同数据量下：

- **拉链法**：α > 1 才扩容，最终表大小 ≈ 数据量，扩容次数少。
- **开放寻址**：α > 0.5~0.7 就要扩容，需要更大的表才能容纳同样数据，扩容更频繁。

**更多次的 rehash × 渐进式 rehash 太重 = 不选开放寻址的原因。**
