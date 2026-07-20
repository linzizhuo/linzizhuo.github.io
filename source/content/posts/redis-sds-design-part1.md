---
title: "Redis源码解析——一份通用的动态字符串设计方案（上）"
date: 2026-07-15
tags: ["Redis", "源码解析", "C", "数据结构", "SDS"]
math: false
---

这篇文章讲述 Redis 中动态字符串的设计，同样也是一份通用字符串模块的设计思路。你可以按照自己的需求将其应用到自己的项目中。

## 1. 渐进式设计：从 char* 到 SDS

在探讨 SDS 之前，先做一些职责分析——

一个字符串是什么样的？

```c
char*
```

一个二进制安全的字符串是什么样的？

```c
size_t len;
char*
```

一个支持扩容的字符串是什么样的？

```c
size_t len;
size_t alloc;
char*
```

一个根据容量划分变量以达到最优性能的字符串呢？

```c
size_t len;
size_t alloc;
uint8_t flags;
char*
```

一个柔性数组优化后的字符串是什么样的？

```c
size_t len;
size_t alloc;
uint8_t flags;
char[];   // 柔性数组
```

> **补充：关于柔性数组**
>
> 柔性数组并不是一个指针，它只是 C99 的一种语法，本质是 `&header + sizeof(header)`，不占用结构体空间。使用方法是 `malloc(sizeof head + N)`，容量是 N。后面在考虑的时候就可以把柔性数组撇掉了。
>
> - 好处：节省一次 malloc
> - 坏处：和 head 是连续内存，扩容不方便

## 2. 优化思路：捡起被浪费的字节

怎么对字符串进行优化？其实很简单——

在通用场景中我们可能习惯了使用 `size_t` 存储长度，但对于一个小字符串来说（`len <= 256`），长度只占用一个字节，剩下的七个字节完全是浪费的。所以优化思路也很简单：**捡起这部分被浪费的字节。**

## 3. 为什么要优化？—— 时间 vs 空间的博弈

查了一些资料、询问了 AI，他们给出的答案往往是 char 一般很小，len 和 alloc 利用不充分，很多情况下头部占了大部分空间。这些说法很合理，也确实是实实在在的需求，但仍不觉得满足。

### 3.1 内存对齐的陷阱

现代操作系统是有字长的，即 CPU 可以一次处理的比特数。现在的操作系统大多都是 64 位，而且 C/C++ 的结构体/类都存在一个机制：**内存对齐。**

在内存对齐的支持下，其实无论成员如何小，结构体都会被 padding 到最大成员的对齐边界，这样优化下来貌似起不到实际的效果？

> 补充：这里说的过于绝对——其实存在 2-3 个小成员共用一个字长的情况。但压榨 SDS 的本质手段就是放弃内存对齐，所以讨论这个，其实是在讨论"在必须对齐的情况下，压榨变量宽度到底能不能带来优化"。
>
> 补充的补充：最开始描述的太绝对了，压榨 SDS 的本质手段是捡起被浪费的内存，`__packed__` 在这个角度只能是附带品。

内存不对齐的结构体会让 CPU 多做一次 load，随机访问性能下降。

问题似乎变成了：**要时间还是要空间的博弈？**

### 3.2 为什么 Redis 选择优化结构体

个人拙见是：Redis 的开发者认为，内存操作足够快，在通用缓存这个场景，内存带宽不会成为瓶颈，反而是整个 Redis 最快的地方（这点很符合直觉）。

结构体优化不仅仅可以充分利用空间，其实还有一个隐藏的好处，在操作系统层面——**小结构的 alloc 能落入更细粒度的 size class，提升内存碎片的利用率。**

在现在的操作系统中，CPU 处理内存不对齐往往是很快的（相比于内存来说），内存不对齐对于 CPU 来说可能只是多一个周期。但内存浪费对于内存来说，可能会：

- 影响内存负载
- 降低系统整体抗压能力
- 内存碎片难以利用
- 影响内存利用率

所以总的来说，内存优化还是非常有必要的。

> 以上都是本人的一些拙见，并不是绝对的，可能也存在考虑不周的情况。

## 4. 优化到底能省多少？

```c
uint8_t len;
uint8_t alloc;
uint8_t flags;
```

只占 **3 字节**！对比原来节省 14 个字节，从 17 → 3，在小数据多的场景下这个优化是十分显著的。

以此类推，可以得出一个结论：**就算没有内存对齐的约束，这个优化也是显著的。**

## 5. Redis 的做法：多级 SDS

### 5.1 类型标记

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
```

### 5.2 sdshdr5：极致的优化思路

这里讲一下 sdshdr5 的结构，提供一种"极致的优化思路"：

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_BITS 3
#define SDS_TYPE_5_LEN(f) ((f) >> SDS_TYPE_BITS)  // 右移 3 位拿长度

struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
```

sdshdr5 让长度和类型共用了 flags 这一个字节（不单独设置 len 和 alloc）：

- 低 3 bit 存储 SDS 类型标记
- 高 5 bit 存储数据长度
- 最大容量 = 2^5 = **31 字节**

### 5.3 核心宏

```c
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));

// 这是柔性数组优化的一个通用手段：在很多时候，我们操作的主体是 buf 而非整个字符串，
// header 只是提供一些配置信息而已。这里初学者看到可能会觉得莫名其妙，但只要将 buf
// 想成数组指针就可以理解了——buf 是数组名，s - sizeof(header) 算出 header 的起始地址。
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```

### 5.4 分级结构体

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;         /* used */
    uint8_t alloc;       /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len;        /* used */
    uint16_t alloc;      /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;        /* used */
    uint32_t alloc;      /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len;        /* used */
    uint64_t alloc;      /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

> **补充：为什么使用 `__packed__`** ——核心不在节省内存上，而在于柔性数组想要获得 flags 需要 `s[-1]`，但在内存对齐的情况下 flags 实际在哪个位置是很难确定的，整个设计就会存在巨大的缺陷。所以我认为这才是使用 `__packed__` 的主要原因，今后在自己设计类似模块的时候一定不要忘记开 `__packed__`。

## 6. 静态路由函数

这里直接照搬 Redis 的源码。

### 6.1 获取 SDS 类型

```c
static inline unsigned char sdsType(sds s) {
    unsigned char flags = s[-1];
    return flags & SDS_TYPE_MASK;
}
```

### 6.2 辅助位操作

```c
/* 从 sds header 的保留位中读取一个用户数据位。bit 索引 0-4。
 * 对 SDS_TYPE_5 始终返回 0。 */
static inline int sdsGetAuxBit(sds s, int bit) {
    if (sdsType(s) == SDS_TYPE_5) 
        return 0;
    
    unsigned char flags = s[-1];
    return (flags & (1U << (SDS_TYPE_BITS + bit))) != 0U;
}

/* 在 sds header 的保留位中存储一个用户数据位。bit 索引 0-4，value 为 0 或 1。
 * aux bits 在 sds 扩容时可能丢失。仅用于特殊场景，如嵌入其他结构的不可变 sds。 */
static inline void sdsSetAuxBit(sds s, int bit, int value) {
    if (sdsType(s) == SDS_TYPE_5) return;
    unsigned char flags = s[-1];
    if (value) {
        flags |= 1U << (SDS_TYPE_BITS + bit);
    } else {
        flags &= ~(1U << (SDS_TYPE_BITS + bit));
    }
    s[-1] = (char)flags;
}
```

### 6.3 sdslen — 获取长度

```c
static inline size_t sdslen(const sds s) {
    switch (sdsType(s)) {
        case SDS_TYPE_5: return SDS_TYPE_5_LEN(s);
        case SDS_TYPE_8:  return SDS_HDR(8,s)->len;
        case SDS_TYPE_16: return SDS_HDR(16,s)->len;
        case SDS_TYPE_32: return SDS_HDR(32,s)->len;
        case SDS_TYPE_64: return SDS_HDR(64,s)->len;
    }
    return 0;
}
```

### 6.4 sdsavail — 获取可用空间

```c
static inline size_t sdsavail(const sds s) {
    switch(sdsType(s)) {
        case SDS_TYPE_5: { return 0; }
        case SDS_TYPE_8:  { SDS_HDR_VAR(8,s);  return sh->alloc - sh->len; }
        case SDS_TYPE_16: { SDS_HDR_VAR(16,s); return sh->alloc - sh->len; }
        case SDS_TYPE_32: { SDS_HDR_VAR(32,s); return sh->alloc - sh->len; }
        case SDS_TYPE_64: { SDS_HDR_VAR(64,s); return sh->alloc - sh->len; }
    }
    return 0;
}
```

### 6.5 sdssetlen / sdsinclen — 设置/增加长度

```c
static inline void sdssetlen(sds s, size_t newlen) {
    switch(sdsType(s)) {
        case SDS_TYPE_5: {
            unsigned char *fp = ((unsigned char*)s)-1;
            *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
        } break;
        case SDS_TYPE_8:  SDS_HDR(8,s)->len = newlen;  break;
        case SDS_TYPE_16: SDS_HDR(16,s)->len = newlen; break;
        case SDS_TYPE_32: SDS_HDR(32,s)->len = newlen; break;
        case SDS_TYPE_64: SDS_HDR(64,s)->len = newlen; break;
    }
}

static inline void sdsinclen(sds s, size_t inc) {
    switch(sdsType(s)) {
        case SDS_TYPE_5: {
            unsigned char *fp = ((unsigned char*)s)-1;
            unsigned char newlen = SDS_TYPE_5_LEN(s) + inc;
            *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
        } break;
        case SDS_TYPE_8:  SDS_HDR(8,s)->len += inc;  break;
        case SDS_TYPE_16: SDS_HDR(16,s)->len += inc; break;
        case SDS_TYPE_32: SDS_HDR(32,s)->len += inc; break;
        case SDS_TYPE_64: SDS_HDR(64,s)->len += inc; break;
    }
}
```

### 6.6 sdsAllocSize — 总分配大小

返回 SDS 字符串的总分配大小，包括：
- SDS header
- 字符串数据
- 尾部空闲缓冲区
- 隐式 null 终止符

```c
static inline size_t sdsAllocSize(sds s) {
    switch(sdsType(s)) {
        case SDS_TYPE_5:  return sizeof(struct sdshdr5) + SDS_TYPE_5_LEN(s) + 1;
        case SDS_TYPE_8:  return sizeof(struct sdshdr8) + SDS_HDR(8,s)->alloc + 1;
        case SDS_TYPE_16: return sizeof(struct sdshdr16) + SDS_HDR(16,s)->alloc + 1;
        case SDS_TYPE_32: return sizeof(struct sdshdr32) + SDS_HDR(32,s)->alloc + 1;
        case SDS_TYPE_64: return sizeof(struct sdshdr64) + SDS_HDR(64,s)->alloc + 1;
    }
    return 0;
}
```

### 6.7 sdsalloc — 获取 alloc 字段

```c
/* sdsalloc() = sdsavail() + sdslen() */
static inline size_t sdsalloc(const sds s) {
    switch(sdsType(s)) {
        case SDS_TYPE_5:  return SDS_TYPE_5_LEN(s);
        case SDS_TYPE_8:  return SDS_HDR(8,s)->alloc;
        case SDS_TYPE_16: return SDS_HDR(16,s)->alloc;
        case SDS_TYPE_32: return SDS_HDR(32,s)->alloc;
        case SDS_TYPE_64: return SDS_HDR(64,s)->alloc;
    }
    return 0;
}
```

### 6.8 sdssetalloc — 设置 alloc 字段

```c
static inline void sdssetalloc(sds s, size_t newlen) {
    switch(sdsType(s)) {
        case SDS_TYPE_5: /* Nothing to do, this type has no total allocation info. */ break;
        case SDS_TYPE_8:  SDS_HDR(8,s)->alloc = newlen;  break;
        case SDS_TYPE_16: SDS_HDR(16,s)->alloc = newlen; break;
        case SDS_TYPE_32: SDS_HDR(32,s)->alloc = newlen; break;
        case SDS_TYPE_64: SDS_HDR(64,s)->alloc = newlen; break;
    }
}
```

### 6.9 sdsHdrSize — Header 大小

```c
static inline int sdsHdrSize(char type) {
    switch(type & SDS_TYPE_MASK) {
        case SDS_TYPE_5:  return sizeof(struct sdshdr5);
        case SDS_TYPE_8:  return sizeof(struct sdshdr8);
        case SDS_TYPE_16: return sizeof(struct sdshdr16);
        case SDS_TYPE_32: return sizeof(struct sdshdr32);
        case SDS_TYPE_64: return sizeof(struct sdshdr64);
    }
    return 0;
}
```

## 7. 补充说明

其实用到静态路由的场景还有扩容，但扩容属于 SDS 内部逻辑，在 Redis 源码中夹带了很多私货（指的是一些 Redis 其他模块里的方法、函数、宏），并不干净，所以这里就不扩展了，原理是一样的。

下一篇会补全 `.c` 里的 `sdsnew`（创建）和扩容逻辑，把整个动态字符串方案讲完整。

（未完待续）
