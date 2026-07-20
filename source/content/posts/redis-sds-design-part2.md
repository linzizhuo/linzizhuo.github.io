---
title: "Redis源码解析——一份通用的动态字符串设计方案（下）"
date: 2026-07-16
tags: ["Redis", "源码解析", "C", "数据结构", "SDS"]
math: false
---

之前我们讲过了 Redis SDS 中的分类设计，今天来讲讲 `new` 和 `expand` 这两个部分。

两者看上去大差不差，但——

- 如果在分段头设计上 SDS 体现出了一种思想：**按需分配，绝不多做事情**；
- 那么在 `new`/`expand` 上则体现出了一种：**该保守保守，该激进激进**的设计哲学。

两者本质上是相同的，都是在有限情况下对资源的最优解。

首先声明，这里不会讲 SDS 所有涉及 `new`/`expand` 的实现，而是主要围绕两个核心函数展开：

- `_sdsnewlen`
- `_sdsMakeRoomFor`

> **补充**：因为在拆解的过程中不仅接触到了主逻辑、复杂的边界判断，还有很深的设计哲学。这里主要从前两方面来讲解。

---

## 一、`_sdsnewlen` — 创建新 SDS 字符串

### 函数签名与注释

```c
/* 根据 'init' 指针和 'initlen' 指定的内容创建一个新的 sds 字符串。
 * 如果 'init' 为 NULL，则字符串用零字节初始化。
 * 如果使用 SDS_NOINIT，则缓冲区保持未初始化状态；
 *
 * 字符串始终以 null 结尾（所有 sds 字符串都是如此），因此
 * 即使你这样创建一个 sds 字符串：
 *
 * mystring = sdsnewlen("abc",3);
 *
 * 你也可以用 printf() 打印该字符串，因为字符串末尾隐式包含 \0。
 * 然而该字符串是二进制安全的，中间可以包含 \0 字符，
 * 因为长度存储在 sds 头部中。 */
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;

    char type = sdsReqType(initlen);
    /* 空字符串通常是为了后续追加而创建的。使用 type 8，
     * 因为 type 5 不适合此场景。 */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    size_t bufsize;

    if (trymalloc) {
        /* 防止 size_t 溢出 */
        if (initlen + hdrlen + 1 <= initlen)
            return NULL;
    } else {
        assert(initlen + hdrlen + 1 > initlen); /* 捕获 size_t 溢出 */
    }
    
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &bufsize) :
        s_malloc_usable(hdrlen+initlen+1, &bufsize);
    if (sh == NULL) return NULL;

    adjustTypeIfNeeded(&type, &hdrlen, bufsize);
    return sdsnewplacement(sh, bufsize, type, init, initlen);
}
```

### 核心要点

#### 1. 缓冲区三种初始化方式

`new` 后的缓冲区有三种处理方式：

| 方式 | 说明 |
| --- | --- |
| 拷贝 | 将 `init` 指向的内容拷贝到新缓冲区 |
| 零初始化 | `init` 为 NULL 时，用零字节填充 |
| 不初始化 | 使用 `SDS_NOINIT`，缓冲区保持未初始化状态 |

#### 2. `trymalloc` 与非 `trymalloc` — 两种错误处理策略

- **`trymalloc`**：将 `size_t` 溢出和 `malloc` 分配失败视为可恢复错误，**优雅降级**（返回 NULL）
- **非 `trymalloc`**：使用 `assert` 直接**内部崩溃**，适用于不可能失败或失败即致命的场景

#### 3. 空串 Type 5 → Type 8 的主动升级

这是设计中的关键决策：

- **Type 5 的设计缺陷**：没有 `alloc` 字段，想要获得缓冲区长度需要进行系统调用
- **空串的唯一切实使用场景就是追加**（append）
- **设计哲学**：与其等它扩容时再付出系统调用的代价，不如在创建阶段就分配 `alloc` 字段，用极小的成本规避掉一次可能性极大的系统调用

#### 4. 充分利用 malloc 多给的内存

在一些操作系统中，`malloc` 可能会多给一些内存。通过 `s_malloc_usable` / `s_trymalloc_usable` 获取实际可用大小（`bufsize`），再经由 `adjustTypeIfNeeded` 将这些"多余"的内存也充分利用起来——可能因此升级到更大的 type，获得更大的容量。

### 代码层面的技巧

**溢出判断**：

```c
initlen + hdrlen + 1 <= initlen
```

用溢出本身去判断溢出，简洁高效——如果加法结果反而小于等于原值，说明发生了回绕（溢出）。

**宏包装的抽象层**：

```c
#define s_trymalloc_usable ztrymalloc_usable
#define s_malloc_usable zmalloc_usable
```

使用纯宏包装的纯抽象层，不仅将模块解耦，还巧妙地将不同操作系统的差异包装在内部。

---

## 二、`_sdsMakeRoomFor` — 扩容 SDS 字符串

### 函数签名与注释

```c
/* 扩大 sds 字符串末尾的可用空间，使调用者在调用此函数后，
 * 可以安全地在字符串末尾之后覆盖最多 addlen 字节，
 * 外加一个用于 null 终止符的额外字节。
 * 如果已有足够的可用空间，此函数直接返回，不做任何操作；
 * 如果可用空间不足，它会分配所需的空间，甚至可能分配更多：
 * 当 greedy 为 1 时，分配比所需更多的空间，以避免增量增长时未来需要再次重新分配。
 * 当 greedy 为 0 时，仅分配刚好足够的空间来容纳 'addlen'。
 *
 * 注意：这不会改变 sdslen() 返回的 sds 字符串的*长度*，
 * 只会改变我们拥有的可用缓冲区空间。 */
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    char type, oldtype = sdsType(s);
    int hdrlen;
    size_t bufsize, usable;
    int use_realloc;

    /* 如果剩余空间足够，尽快返回。 */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    reqlen = newlen = (len+addlen);
    assert(newlen > len);   /* 捕获 size_t 溢出 */
    if (greedy == 1) {
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;
    }

    type = sdsReqType(newlen);

    /* 不要使用 type 5：用户正在向字符串追加内容，而 type 5
     * 无法记住空闲空间，因此每次追加操作都必须调用 sdsMakeRoomFor()。 */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > reqlen);  /* 捕获 size_t 溢出 */
    use_realloc = (oldtype == type);
    if (use_realloc) {
        newsh = s_realloc_usable(sh, hdrlen + newlen + 1, &bufsize, NULL);
        if (newsh == NULL) return NULL;
        s = (char*)newsh + hdrlen;
        if (adjustTypeIfNeeded(&type, &hdrlen, bufsize)) {
            memmove((char *)newsh + hdrlen, s, len + 1);
            s = (char *)newsh + hdrlen;
            s[-1] = type;
            sdssetlen(s, len);
        }
    } else {
        /* 由于头部大小发生了变化，需要将字符串向前移动，
         * 并且不能使用 realloc */
        newsh = s_malloc_usable(hdrlen + newlen + 1, &bufsize);
        if (newsh == NULL) return NULL;
        adjustTypeIfNeeded(&type, &hdrlen, bufsize);
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    usable = bufsize - hdrlen - 1;
    assert(type == SDS_TYPE_5 || usable <= sdsTypeMaxSize(type));
    sdssetalloc(s, usable);
    return s;
}
```

### 核心要点

#### 1. 贪婪 vs 非贪婪 — 两种扩容策略

| 模式 | 行为 | 适用场景 |
| --- | --- | --- |
| **贪婪** (`greedy=1`) | 小于 `SDS_MAX_PREALLOC` 时分配 **2 倍**空间；大于时每次只增加 `SDS_MAX_PREALLOC` | 后续可能继续追加的场景 |
| **非贪婪** (`greedy=0`) | 仅分配刚好足够容纳 `addlen` 的空间 | 明确知道不会再扩容的场景 |

这是一种**该保守保守，该激进激进**的设计哲学：

- 小字符串阶段激进预分配（2 倍增长），减少频繁 `realloc`
- 大字符串阶段保守增长（固定增量），避免内存浪费

#### 2. 同类型扩容：`realloc` 的巧妙利用

当 `oldtype == type`（类型未变）时使用 `realloc`：

- `realloc` 可能在**原地扩展**，省去一次 `memcpy`
- 也可能**搬家**——由分配器决定，SDS 不关心
- 这是 C 标准库提供的能力，SDS 只是善加利用

#### 3. 不同类型扩容：`malloc` + `memcpy` + `free`

当 type 发生变化（如从 type8 升级到 type16）时：

1. `malloc` 新的缓冲区
2. `memcpy` 拷贝数据
3. `free` 旧缓冲区

因为头部大小变了，无法使用 `realloc`。

#### 4. 与 `_sdsnewlen` 共享的设计

- **Type 5 → Type 8 升级**：防止无 `alloc` 字段导致的后续性能问题
- **`assert` 溢出捕获**
- **`adjustTypeIfNeeded`**：充分利用 `malloc` 实际分配的内存

---

## 三、设计哲学总结

### 两条核心原则

1. **按需分配，绝不多做事情**（header 分类设计）→ 体现在 type 的精细化分级
2. **该保守保守，该激进激进**（new/expand 策略）→ 体现在：
    - 空串创建时主动升级 type（激进地预防未来开销）
    - 小字符串 2 倍扩容（激进预分配）、大字符串固定增量（保守增长）
    - 尽量利用 `malloc` 多给的内存（精打细算）
    - 同类型 `realloc` 避免拷贝（能省则省）

### 学习心得

初次学习这段代码时，不应该追求面面俱到的"完美理解"。因为这段代码从创建之初到现在可能经历了很多次迭代，里面有许多难以理解的历史遗留因素，甚至某些判断在现在看来是重复的。

**设计是可以学习的，但代码本身是一次性脑力劳动的结果**——我们很难根据一段代码去追溯作者在写这段代码的时候在想什么。抓住设计哲学，比逐行理解更重要。

（完）
