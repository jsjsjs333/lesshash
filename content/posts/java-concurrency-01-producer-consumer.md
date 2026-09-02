---
title: "Java并发练习01：手写生产者-消费者（wait/notifyAll 与 Lock/Condition 双版本）"
date: 2026-09-02T20:30:00+08:00
slug: java-concurrency-01-producer-consumer
draft: false
tags: ["Java", "并发编程", "生产者消费者", "wait/notify", "Lock/Condition", "编程教程"]
categories: ["Java并发"]
series: ["Java并发编程练习"]
author: "lesshash"
description: "从零手写生产者-消费者模型：第一版用 synchronized + wait/notifyAll，第二版升级为 ReentrantLock + 双 Condition 精准唤醒。附完整可运行代码、真实运行输出、为什么必须用 while 而不是 if、以及一个肉眼可见的'栈不是队列'翻车现场。"
---

## 🎯 练习目标

生产者-消费者是并发编程的"Hello World"：一个线程往缓冲区放数据，一个线程从缓冲区取数据，缓冲区满了生产者等，空了消费者等。消息队列、线程池的任务队列、日志异步刷盘，底层全是这个模型。

这次练习分两版实现，体会从"JVM 内置监视器锁"到"JUC 显式锁"的升级到底强在哪：

```
版本一：synchronized + wait() + notifyAll()   ← 一个条件队列，全员广播
版本二：ReentrantLock + 两个 Condition        ← 两条条件队列，精准唤醒
```

## 🌟 版本一：synchronized + wait/notifyAll

```java
public class ProducterAndConsumer {
    private final int[] buffer;
    private int count;
    private final int capacity;

    public ProducterAndConsumer(int capacity) {
        this.capacity = capacity;
        this.buffer = new int[capacity];
        this.count = 0;
    }

    public synchronized void produce(int item) throws InterruptedException {
        while (count == capacity) {
            wait(); // Buffer is full, wait for consumer to consume
        }
        buffer[count++] = item;
        System.out.println("Produced: " + item);
        notifyAll(); // Notify consumer that an item has been produced
    }

    public synchronized int consume() throws InterruptedException {
        while (count == 0) {
            wait(); // Buffer is empty, wait for producer to produce
        }
        int item = buffer[--count];
        System.out.println("Consumed: " + item);
        notifyAll(); // Notify producer that an item has been consumed
        return item;
    }
}
```

测试代码：容量 5 的缓冲区，生产者 10 个产品（每个 100ms），消费者消费 10 个（每个 150ms）——故意让消费者更慢，逼出"缓冲区满"的场景：

```java
public class ProducterAndConsumerTest {
    public static void main(String[] args) throws InterruptedException {
        ProducterAndConsumer buffer = new ProducterAndConsumer(5); // Buffer capacity of 5

        // Producer thread
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    buffer.produce(i);
                    Thread.sleep(100); // Simulate time taken to produce an item
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // Consumer thread
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    buffer.consume();
                    Thread.sleep(150); // Simulate time taken to consume an item
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

### 真实运行输出

```
Produced: 0
Consumed: 0
Produced: 1
Consumed: 1
Produced: 2
Consumed: 2
Produced: 3
Produced: 4
Consumed: 4
Produced: 5
Consumed: 5
Produced: 6
Produced: 7
Consumed: 7
Produced: 8
Consumed: 8
Produced: 9
Consumed: 9
Consumed: 6
Consumed: 3
```

程序正常结束，没有死锁。但注意最后两行：**消费顺序乱了**，6 和 3 是最后才被消费的。这不是并发调度的随机性，而是一个实打实的设计问题，下文详解。

## 💡 三个必懂的知识点

### 1. 为什么用 while 判断条件，而不是 if？

`wait()` 有两大特性决定了必须用 while：

- **虚假唤醒（spurious wakeup）**：JVM 规范允许线程在没人 notify 的情况下"自己醒来"，这是底层 futex 机制留下的自由度；
- **唤醒后条件未必成立**：`notifyAll()` 叫醒的是"所有"等在同一个监视器上的线程。被叫醒的线程从 `wait()` 处恢复后，必须重新抢锁、重新检查条件——此时条件可能又被别的线程消费掉了。

```
if (count == 0) {          while (count == 0) {
    wait();       ✗            wait();         ✓
}                          }
// 醒来直接取数据          // 醒来再检查一遍，条件被抢就继续等
// 条件已失效 → 崩        // 永远不会在失效条件下动手
```

经验法则：**wait 永远住在 while 里**。JDK 源码里所有 wait 调用无一例外。

### 2. 为什么是 notifyAll() 而不是 notify()？

因为这里只有**一个**条件队列（监视器对象自身），生产者和消费者混在一起等。`notify()` 只随机叫醒一个：

```
情景：缓冲区满，3 个生产者都在 wait
  消费者消费 1 个 → notify() 随机叫醒 1 个生产者 ✓ 运气好
  但若叫醒的是另一个消费者呢？它检查 count != 0... 不对，
  消费者只在空时 wait，此情景不会发生——可一旦生产者/消费者
  各有多个、条件交叉，notify 就可能叫醒"不需要醒的人"，
  真正该醒的没人管 → 全员休眠 → 死锁
```

`notifyAll()` 广播叫醒所有人，让大家自己抢锁、自己检查条件，安全但低效——这就是"惊群"。版本二正是为了消灭它。

### 3. 翻车现场：这是栈，不是队列

看消费逻辑 `buffer[--count]`：每次从**顶部**取元素。再看真实输出的结尾：

```
Produced: 0..9 顺序生产
Consumed: 0,1,2,4,5,7,9,8,6,3   ← 不是 0,1,2,3,...
```

消费者较慢（150ms），生产者把缓冲区堆到 5 个（比如 3,4,5,6,7），消费者每次取走的是**最新**压进去的那个（7），老元素（3）沉底。这就是 LIFO——后进先出。

对生产者-消费者的多数场景（任务队列、消息队列）我们要的是 FIFO。修复思路：加一个 `head` 游标从底部取，或者直接换 `ArrayDeque`/`ArrayBlockingQueue`。本练习保留现场作为记录：**数据结构选错，程序照跑，只有看输出才知道**。

## 🌟 版本二：ReentrantLock + 双 Condition

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ProducterAndConsumer {
    private final int[] buffer;
    private int count;
    private final int capacity;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition productCondition = lock.newCondition(); // 生产者的等待室
    private final Condition consumeCondition = lock.newCondition(); // 消费者的等待室

    public ProducterAndConsumer(int capacity) {
        this.capacity = capacity;
        this.buffer = new int[capacity];
        this.count = 0;
    }

    public void produce(int item) throws InterruptedException {
        lock.lock();
        try {
            while (count == capacity) {
                productCondition.await(); // Buffer is full, wait for consumer to consume
            }
            buffer[count++] = item;
            System.out.println("Produced: " + item);
            consumeCondition.signal(); // Notify consumer that an item has been produced
        } finally {
            lock.unlock();
        }
    }

    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                consumeCondition.await(); // Buffer is empty, wait for producer to produce
            }
            int item = buffer[--count];
            System.out.println("Consumed: " + item);
            productCondition.signal(); // Notify producer that an item has been consumed
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

（测试类与版本一完全相同。）

核心升级：**一把锁，两条条件队列**。

```
              ┌─────────────────────────────┐
              │      ReentrantLock（锁）      │
              └─────────────────────────────┘
                    │                │
        ┌───────────▼──────┐ ┌──────▼───────────┐
        │ productCondition │ │ consumeCondition │
        │  生产者等待室      │ │  消费者等待室      │
        │  满 → await      │ │  空 → await       │
        └──────────────────┘ └──────────────────┘
```

- 生产成功 → `consumeCondition.signal()`：只叫醒**一个消费者**，因为确切知道"有数据了"；
- 消费成功 → `productCondition.signal()`：只叫醒**一个生产者**。

没有惊群，没有"叫错人"，连 while 都显得更纯粹——只剩防虚假唤醒一个职责。单生产者单消费者下两版输出完全一致，差异要在多线程竞争下才显现：线程越多，notifyAll 的无效唤醒开销越大，双 Condition 的精准打击越值钱。

另外两个细节：

- **`lock.unlock()` 必须放 finally**：中途抛异常也能释放锁，否则锁永久持有，等价于死锁；
- **`await()` 放在 try 里、signal 可以在任意位置**：await 会自动释放锁并挂起，被唤醒后自动重新抢锁再从 await 处继续。

## 📊 两版对比

| 维度 | synchronized + notifyAll | Lock + 双 Condition |
|---|---|---|
| 条件队列 | 1 个（监视器内置），所有角色混住 | 每个角色一间等待室，按需开 |
| 唤醒精度 | 广播，全员惊群 | signal 精准唤醒单个 |
| 释放锁 | 退出 synchronized 块自动释放 | 必须手动 unlock()（配 finally） |
| 灵活性 | 无法中断等待、无超时、非公平 | 可中断、可超时、可公平锁 |
| 代码量 | 少 | 多一点，但结构更清晰 |

选择建议：简单场景 synchronized 够用且不容易写错；一旦出现**多组等待条件**（这里就是"满"和"空"两组），Condition 的两间等待室就是标准答案。

## 🧭 和 JUC 的关系：你手写的就是 ArrayBlockingQueue 的骨架

打开 `ArrayBlockingQueue` 源码，看到的正是版本二的结构：

```java
/** Main lock guarding all access */
final ReentrantLock lock;
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;
```

`notEmpty` / `notFull` 两条条件队列，put 成功后 `notEmpty.signal()`，take 成功后 `notFull.signal()`——和我们的 `consumeCondition` / `productCondition` 一一对应。区别在于 JUC 还处理了中断响应、公平性、批量迁移（drainTo）等工程细节。

所以日常业务直接用 `BlockingQueue`（put/take 自动阻塞），但面试问"BlockingQueue 怎么实现阻塞"，答案就是这篇文章。

## ⚠️ 踩坑清单

1. **wait 用 if 包裹** → 虚假唤醒/条件失效时越界执行。永远用 while。
2. **notify 乱用** → 混合条件下可能叫错人导致死锁。内置锁场景默认 notifyAll。
3. **持有锁时休眠**：`wait()` 会释放锁，`sleep()` 不会！在 synchronized 块里 sleep 是"抱着锁睡觉"，其他线程全部干等。
4. **unlock 不放 finally** → 异常路径锁泄漏。
5. **wait/notify 必须在持有监视器锁时调用**，否则抛 `IllegalMonitorStateException`——这就是为什么它们必须写在 synchronized 方法/块内。
6. **消费端用 `--count` 导致 LIFO** → 本篇翻车现场，看输出才暴露。
7. **InterruptedException 的正确处理**：catch 后调用 `Thread.currentThread().interrupt()` 恢复中断标志位，让上层调用链还能感知中断（测试代码里就是这么写的）。

## 📝 练习收获

- 生产者-消费者的本质：**共享缓冲区 + 两个条件（满/空）+ 条件不满足就挂起、状态变化就唤醒**；
- while 包 wait 是铁律，notifyAll 是内置锁下的保守正确，双 Condition 是显式锁下的精确高效；
- 跑真实输出比看代码多看出一个问题（LIFO），并发代码一定要亲眼盯输出；
- 下一练：[手写滑动窗口限流器（三版演进与一次翻车）](/2026/09/java-concurrency-02-sliding-window-limiter/)——那里有一个温和测试全过、压力测试直接饿死的隐蔽 bug。
