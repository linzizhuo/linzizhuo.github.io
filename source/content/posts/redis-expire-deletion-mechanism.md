---
title: "Redis过期删除机制：从惰性删除到定期抽样"
date: 2026-07-20
tags: ["Redis", "源码解析", "过期策略", "内存管理", "惰性删除"]
math: false
---

> 解析 Redis 过期 key 删除策略的层层设计——从惰性删除到定期抽样，理解"宁可少删，不可阻塞"的哲学。

---

## 1. 过期删除的三种设计思路

设计 key 过期机制，通常有三种方案：

### 方案一：定时全量扫描

维护一张 TTL 表，定时扫描表中哪些 key 过期了。

**问题**：O(n) 遍历，极其低效。百万级 key 扫一轮，CPU 全耗在上面了。

### 方案二：系统中断驱动

利用系统调用的定时器中断，精确在 key 过期的那一刻触发删除。

**问题**：看似优雅，但对 Redis 这种海量 key 的内存数据库来说，未知的中断 = 不可预知的阻塞。频繁中断也会严重影响系统吞吐，违背 Redis 单线程高性能的设计哲学。

### 方案三：惰性删除（Redis 的选择）

不主动清理过期 key，只在访问时顺手看一眼有没有过期。

**优点**：几乎没有额外 CPU 开销。

**缺点**：用空间换时间——已经过期但长期未被访问的 key 会一直占着内存，成为"内存垃圾"。

## 2. 惰性删除的兜底：定期抽样删除

惰性删除本身的缺陷是内存回收不及时，所以需要一个兜底机制来主动扫描。但这又回到了方案一的问题——全量扫描太重。

Redis 的解法是**定期 + 抽样**：不扫全部，每次只随机抽一小部分，用概率换取性能。

## 3. SLOW / FAST 双模式

Redis 将定期删除分为两种模式，配合当前的事件循环模型：

```c
while (true) {
    // epoll 处理网络事件
    ...

    // beforeSleep 里触发，每个事件循环都来
    tryFAST();

    // serverCron 里触发，每秒 hz 次
    if (到 serverCron 的点了)
        activeExpireCycle(SLOW);
}
```

|        | SLOW | FAST |
|--------|------|------|
| 触发路径 | serverCron | beforeSleep（每个事件循环） |
| 频率 | server.hz（默认 10Hz，动态） | 事件循环级（高频） |
| 时间预算 | 25% CPU / hz（默认约 25ms） | 1ms 固定 |
| 职责 | 主力清理 | 见缝插针，给 SLOW 补刀 |

### FAST 前置拦截

FAST 不是每次都真正执行，有两个前置条件：

```c
tryFAST():
    // 条件一：上次 SLOW 没超时 且 全局过期比不高 → 跳过
    if (!timelimit_exit && stale_perc < config_cycle_acceptable_stale)
        return;

    // 条件二：距上次 FAST 太近（小于 fast_duration * 2）→ 跳过
    if (距上次 FAST < config_cycle_fast_duration * 2)
        return;

    activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);
```

也就是说：

- FAST 大部分时候被条件一挡掉，并不执行
- 只有当 SLOW 上次超时了（`timelimit_exit == 1`，说明有大量过期 key 没清完），或者系统统计的过期比例偏高，FAST 才不再跳过，用碎片时间帮忙清理
- 条件二防止 FAST 高频执行，加了一个最小间隔

## 4. activeExpireCycle 内部结构

```c
activeExpireCycle(int type):
    // ── 参数计算 ──
    // 根据 server.active_expire_effort（1~10）调整各项阈值
    config_keys_per_loop         // 每轮抽样 key 数，默认 20
    config_cycle_fast_duration   // FAST 时间上限，默认 1000µs
    config_cycle_slow_time_perc  // SLOW CPU 占比，默认 25%
    config_cycle_acceptable_stale // 过期比阈值，默认 10%

    // ── FAST 前置拦截 ──
    if (type == FAST) {
        if (!timelimit_exit && stale_perc < acceptable_stale) return;
        if (距上次 FAST < fast_duration * 2)                return;
    }

    // ── 时间预算 ──
    // 公式：SLOW 占比% / hz，保证每秒总 CPU 预算恒定为 25%
    timelimit = config_cycle_slow_time_perc * 1000000 / server.hz / 100;

    // ── 三层循环 ──
    for (遍历数据库; 上限 dbs_per_call; j++)
        do {
            // 内层：随机抽样
            // 最多找 num 个 key（≈20），最多扫 num*20 个 bucket
            while (sampled < num && checked_buckets < num * 20):
                随机扫描哈希表 slot → 判断过期 → 收集 sampled/expired

            // 每轮立即算过期比，决定是否继续
            repeat = (expired * 100 / sampled) > config_cycle_acceptable_stale;

            // 每 16 轮检查一次超时
            if ((iteration & 0xf) == 0 && elapsed > timelimit):
                timelimit_exit = 1;
                break;
        } while (repeat);

    // ── 循环后 ──
    // 统计本次耗时 + 更新 stale_perc 滑动平均（本样本权重 5%）
```

### 关键宏定义

```c
#define ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP 20     // 每轮抽样 key 数
#define ACTIVE_EXPIRE_CYCLE_FAST_DURATION 1000    // FAST 每次最多 1000 微秒
#define ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 25     // SLOW 占用 CPU 的 25%
#define ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE 10   // 过期比 > 10% 就继续扫
```

## 5. server.hz 动态调整

`server.hz` 不是固定值，会根据客户端数量自动翻倍：

```c
hz = config_hz（默认 10）
while (clients / hz > MAX_CLIENTS_PER_CLOCK_TICK)
    hz *= 2           // 上限 CONFIG_MAX_HZ（500）
```

| hz | 每秒调用次数 | 单次时间预算 | 每秒总预算 |
|----|------------|------------|-----------|
| 10（空闲） | 10 | 25ms | 250ms |
| 40 | 40 | 6.25ms | 250ms |
| 100（繁忙） | 100 | 2.5ms | 250ms |

核心思想：每秒留给 expire 的总 CPU 预算恒为 25%（250ms），hz 翻倍时单次预算等比缩小，整体占用不变。 客户端越多，cron 跑得越勤，每次干得越少，避免争抢 CPU。

## 6. 总结

Redis 的过期删除策略是一个精巧的多层防御体系：

```
惰性删除（访问时检查）
    ↓ 兜底
定期抽样删除
    ├─ SLOW 模式：低频主力，25ms 预算
    └─ FAST 模式：高频补刀，1ms 预算
        ↓ 超时保护
每 16 轮检查墙上时钟，超时即退出
```

核心设计哲学：**宁可少删，不可阻塞。** 所有限流机制（时间预算、抽样上限、过期比阈值）都指向同一个目标——在不过度占用 CPU 的前提下，尽可能回收过期内存。

这种设计模式并不局限于 expire，它在很多系统中反复出现：

```
上层（多线程/高并发）  →  尽快接住请求，不做重活
中间（队列/缓冲）      →  削峰填谷，解耦
下层（单线程/串行）    →  慢慢干，不阻塞上游
```

| 场景 | 上层 | 中间层 | 下层 |
|------|------|--------|------|
| expire 删除 | beforeSleep 触发 FAST | `repeat` 标志位控制循环 | 随机采样 + 逐 key 删除 |
| 秒杀系统 | Redis 缓存 + 高并发接收 | 消息队列 | 订单服务串行处理 |
| Redis 6.0 多线程 | 多线程 epoll 接包 | 无锁队列 | 单线程内存读写 |

本质上都在解决同一个问题：**上下层速度不匹配时，别让上游等下游——接住、缓冲、慢慢消化。**