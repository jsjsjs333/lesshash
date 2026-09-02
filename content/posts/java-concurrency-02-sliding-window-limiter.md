---
title: "Java并发练习02：手写滑动窗口限流器（三版演进与一次翻车）"
date: 2026-09-02T20:50:00+08:00
slug: java-concurrency-02-sliding-window-limiter
draft: false
tags: ["Java", "并发编程", "限流器", "滑动窗口", "高并发", "编程教程"]
categories: ["Java并发"]
series: ["Java并发编程练习"]
author: "lesshash"
description: "手写滑动窗口限流器三个版本：TreeSet 版温和测试全过、压力测试下却永久饿死——被拒请求的时间戳留在窗口里占坑。附翻车根因分析、ArrayDeque 修复版、环形数组 O(1) 内存版，以及真实压测输出对比。"
---

## 🎯 练习目标

限流器是高并发系统的守门员：接口只允许"每秒 N 个请求"，超了直接拒绝，保护后端不被打挂。滑动窗口（Sliding Window）是最常用的算法之一——窗口不按整秒切分，而是像传送带一样随时间平滑前移：

```
固定窗口（按整秒切）:              滑动窗口（连续前移）:
▔▔▔▔▔▔▔|▔▔▔▔▔▔▔            ←━━━━窗口━━━━→
500ms处打满5个,            任意连续1秒内
下一秒开头又能打5个          都不能超过5个
→ 边界突刺可达2倍峰值        → 无突刺
```

目标：实现"任意 1000ms 内最多放行 5 个请求"。本次练习写了三版，过程里翻了一次很典型的车，完整记录如下。

## 🌟 版本一：TreeSet 版（翻车现场）

第一反应：用 `TreeSet<Long>` 存所有放行请求的时间戳，TreeSet 自动排序，清理过期和计数都方便：

```java
import java.util.TreeSet;

public class SlidingWindowLimiter {
    private final int limitCount;
    private final long limitTime;
    private final TreeSet<Long> timestamps = new TreeSet<>();

    SlidingWindowLimiter(int limitCount, long limitTime) {
        this.limitCount = limitCount;
        this.limitTime = limitTime;
    }

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        timestamps.add(now);
        timestamps.headSet(now - limitTime).clear(); // 清掉窗口外的旧时间戳
        return timestamps.size() <= limitCount;
    }
}
```

测试：10 个请求、每个间隔 100ms，限 5 次/秒：

```java
public class SlidingWindowLimiterTest {
    public static void main(String[] args) throws InterruptedException {
        SlidingWindowLimiter limiter = new SlidingWindowLimiter(5, 1000); // Allow 5 requests per second

        for (int i = 0; i < 10; i++) {
            if (limiter.tryAcquire()) {
                System.out.println("Request " + (i + 1) + " allowed");
            } else {
                System.out.println("Request " + (i + 1) + " denied");
            }
            Thread.sleep(100); // Sleep for 100ms between requests
        }
    }
}
```

输出：

```
Request 1 allowed
Request 2 allowed
Request 3 allowed
Request 4 allowed
Request 5 allowed
Request 6 denied
Request 7 denied
Request 8 denied
Request 9 denied
Request 10 denied
```

前 5 个放行、之后全拒——看起来完全正确。**但这版有致命 bug，温和测试根本测不出来。**

### 压力测试：持续流量下的永久饿死

把测试改成持续 4 秒的流量（40 个请求、间隔 100ms），统计每版总共放行多少：

```
测试脚本核心逻辑：
    for (int i = 0; i < 40; i++) {
        limiter.tryAcquire() ? 记A : 记.
        Thread.sleep(100);
    }

TreeSet 版:  AAAAA...................................  → 放行  5/40
Queue 版:    AAAAA.....AAAAA.....AAAAA.....AAAAA.....  → 放行 20/40
Array 版:    AAAAA.....AAAAA.....AAAAA.....AAAAA.....  → 放行 20/40
```

正确行为是每过一秒窗口滑出旧请求、恢复放行（20/40）；TreeSet 版在最初 5 个之后**一个都不再放行**——服务等于被打死了。限流器本来是保护系统的，自己先成了故障点。

### 根因分析

看 `tryAcquire()` 的执行顺序：

```java
timestamps.add(now);                          // ① 先无条件登记本次请求
timestamps.headSet(now - limitTime).clear();  // ② 清理过期
return timestamps.size() <= limitCount;       // ③ 用 size 判断放行
```

**被拒绝的请求，时间戳也留在了窗口里占坑。**

推演一下持续流量下发生了什么：

```
t=0~400ms   放行5个，窗口里5个时间戳（合法占用）
t=500ms     第6个请求：先add自己 → size=6 > 5 → 拒绝
            但它的时间戳赖在窗口里不走了！
t=600~900ms 后续请求同样：被拒 → 占坑 → size 越滚越大
t=1000ms    老时间戳开始过期，但每毫秒都有新的"被拒时间戳"补位
            → size 永远 ≥ 5 → 永远拒绝
            → 只要流量不断，一个请求都进不来（永久饿死）
```

修复思路很直接：**判断放行之后再登记，被拒的请求不留痕**。

还有个隐藏问题：`TreeSet` 是去重的——同一毫秒内的多个请求只算一个时间戳，高并发下会**少计**，放行量超预期。另外每个请求一个 `Long` 装箱 + 红黑树节点，攻击者每秒打 10 万个请求，这棵树就长出 10 万个节点，GC 压力自己先把自己压垮。

顺带一提：这版练习最初连测试都跑不起来——Test 里把类名写成了 `ProducterAndConsumer`（上一个练习复制粘贴的残留），编译直接失败。复制粘贴是并发练习第一坑。

## 🌟 版本二：ArrayDeque 版（修复）

用 `ArrayDeque` 当滑动队列：**先清过期、再判容量、放行才入队**：

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class SlidingWindowLimiter {
    private final int limitCount;
    private final long limitTime;
    private final Deque<Long> timestamps = new ArrayDeque<>();

    SlidingWindowLimiter(int limitCount, long limitTime) {
        this.limitCount = limitCount;
        this.limitTime = limitTime;
    }

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        while (!timestamps.isEmpty() && timestamps.peekFirst() <= now - limitTime) {
            timestamps.pollFirst();                      // ① 队头过期则出队（从老到新逐个清）
        }
        if (timestamps.size() < limitCount) {
            timestamps.addLast(now);                     // ② 放行才登记
            return true;
        }
        return false;                                    // ③ 被拒不留痕
    }
}
```

三个关键修复：

1. **先清过期再判断**：`peekFirst()` 是最老的时间戳，它没过期则更年轻的一定都没过期，所以一个 while 从队头清到不过期为止，均摊 O(1)；
2. **被拒请求不进队**：饿死 bug 根治，压力测试恢复 20/40；
3. **Deque 不去重**：同毫秒请求各占一坑，计数准确。

内存上，队列长度严格 ≤ limitCount（被拒不入队），有界。这是正确性、复杂度都达标的一版，日常够用。

## 🌟 版本三：环形数组版（O(1) 定长内存）

再进一步：Deque 虽有界，仍要扩容/装箱（`Long` 对象）。既然窗口最多只存 limitCount 个时间戳，干脆开一个**定长数组 + head 游标**环形复用：

```java
public class SlidingWindowLimiter {
    private final int limitCount;
    private final long limitTime;
    private final long[] timestamps;
    private int size = 0;   // 已填充数量，范围 [0, limitCount]，封顶不涨
    private int head = 0;   // 环形游标，指向最老的时间戳，范围 [0, limitCount)

    SlidingWindowLimiter(int limitCount, long limitTime) {
        this.limitCount = limitCount;
        this.limitTime = limitTime;
        this.timestamps = new long[limitCount];
    }

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();
        if (size < limitCount) {                      // 填充期：直接放行
            timestamps[size++] = now;
            return true;
        }
        if (now - timestamps[head] >= limitTime) {    // 已满：最老的过期则淘汰
            timestamps[head] = now;
            head = (head + 1) % limitCount;           // 游标循环，永不溢出
            return true;
        }
        return false;
    }
}
```

环形复用的精髓：

```
limitCount=5，数组5格，head 指向最老的时间戳：

初始填充:   [ t0 ][ t1 ][ t2 ][ t3 ][ t4 ]   head=0
              ↑最老

淘汰t0后:   [ t5 ][ t1 ][ t2 ][ t3 ][ t4 ]   head=1
             新值覆写老坑，head 后移指向新的最老(t1)

再淘汰t1:   [ t5 ][ t6 ][ t2 ][ t3 ][ t4 ]   head=2
              ...数组永远只有5格，零扩容零装箱
```

- `long[]` 基础类型数组，**零装箱、零节点分配、零 GC**；
- 内存分配一次定死：5 个槽就是 40 字节，与流量大小无关——攻击者打多猛，内存纹丝不动；
- 全程 O(1)，没有 while 清理循环（最多淘汰一个最老的即可，因为每次请求只消耗一个名额）。

一个小边界差异值得留意：TreeSet 版清理用的是 `headSet(now - limitTime)`（严格小于才清），Queue/Array 版用 `>= limitTime` 判过期（含等于）。1ms 的边界差不影响工程正确性，但写限流器时**过期边界的开闭要心里有数**。

## 📊 三版对比

| 维度 | TreeSet 版 | ArrayDeque 版 | 环形数组版 |
|---|---|---|---|
| 正确性 | ❌ 被拒占坑→永久饿死 | ✅ | ✅ |
| 同毫秒计数 | ❌ 去重少计 | ✅ | ✅ |
| 单次操作复杂度 | O(log n) | 均摊 O(1) | O(1) |
| 内存 | 无界（随攻击流量涨） | ≤ limitCount | 恒定 limitCount×8B |
| 装箱/GC | Long 装箱+树节点 | Long 装箱 | 原生 long，零分配 |
| 适用 | 只适合当反面教材 | 日常够用 | 高频热点路径 |

## 🧭 离生产还有多远

这三版都是 `synchronized` 整体加锁的单机限流器，练习到此为止。生产环境还要走更远：

- **降低锁竞争**：单机可用 `LongAdder` 思路分片，或直接用轮询 CAS 替换锁；
- **时间片分桶（Sentinel 的做法）**：上面的"逐请求记时间戳"叫滑动窗口日志（log）版，占内存与请求数成正比；Sentinel 的 `LeapArray` 把 1 秒切成 20 个 50ms 的桶，每桶只存一个计数，窗口滑动时按桶聚合——内存与**请求数无关**，只与桶数有关，这才是工业级滑动窗口；
- **分布式限流**：多实例共享计数就得把窗口搬到 Redis（Lua 脚本保证"取过期+判断+写入"原子性），或用网关层统一限流；
- **策略选择**：滑动窗口之外还有令牌桶（允许突发）、漏桶（绝对平滑），按业务形态选型。

## ⚠️ 踩坑清单

1. **温和测试全过 ≠ 正确**：间隔 100ms 的测试永远暴露不了"被拒占坑"，必须上持续流量压测。本篇翻车的唯一原因就是第一轮测试太温柔；
2. **先写后判 vs 先判后写**：一进方法就 `add` 是"记录所有到达"，先判断再 `add` 才是"记录所有放行"，限流器要的是后者；
3. **TreeSet/Map 去重语义**：当 multiset 用必翻车，时间戳恰好是高概率撞同一毫秒的值；
4. **无界集合 = 攻击面**：凡是"存请求痕迹"的结构，先问一句被 10 万 QPS 打时会变成多大；
5. **复制粘贴改类名**：本次 Test 类里残留上个练习的类名，编译期才暴露，`javac` 是最后的安全网；
6. **过期边界开闭**（`<` vs `<=`）：三版口径不一致并不影响结论，但团队内应统一。

## 📝 练习收获

- 滑动窗口限流器骨架就三步：**清过期 → 判容量 → 放行登记**，顺序错了就是事故；
- 判断一个限流器好坏，看三件事：被拒请求是否污染窗口、内存是否随流量增长、高并发下计数是否准确；
- 压测发现的 bug 与温和测试的关系：并发组件的正确性证明不了，只能靠**贴近真实流量形态的测试**去证伪；
- 上一练：[手写生产者-消费者（wait/notifyAll 与 Lock/Condition 双版本）](/2026/09/java-concurrency-01-producer-consumer/)。
