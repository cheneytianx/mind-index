---
tags: [concurrent, os, lock, mutex, kernel]
created: 2026-05-18
source: conversation
---

# Spin Lock 与 Mutex 的底层实现与场景

> 从硬件指令到内核调度，理解 [[Spin Lock]] 与 [[Mutex]] 的底层差异及其各自适用场景。

## 一、Spin Lock 底层实现

### 核心原理

一个共享内存的整型变量 + CPU 原子操作指令。

最简单的 TAS（Test-And-Set）实现：

```c
void spin_lock(volatile int *lock) {
    while (__sync_lock_test_and_set(lock, 1)) {
        // lock 原为 0 → 拿到锁
        // lock 原为 1 → 原地死等
    }
}
```

### 硬件原子原语

| 指令 | 架构 | 行为 |
|:--|:--|:--|
| `xchg` | x86 | 原子交换寄存器和内存的值 |
| `ldaxr`/`stlxr` | ARM | Load-Exclusive / Store-Exclusive |
| CAS | 通用 | 比较旧值，相等才写入新值 |

### x86 上 TAS 的实际流程

```asm
    mov     eax, 1
spin_loop:
    xchg    [lock], eax    ; 原子交换，读回原值
    test    eax, eax
    jz      locked
    pause                   ; PAUSE 指令：标识 spin-wait loop，降功耗提性能
    jmp     spin_loop
locked:
    ret
```

### 关键细节

- **`PAUSE` 指令**标记自旋等待循环，让 CPU 流水线不影响超线程兄弟核心，提升约 30% 性能并降功耗
- 现代变种：**Ticket Spin Lock**（按排队号持有，保证公平）、**MCS Lock**（每个核本地变量，避免多核缓存一致性风暴）

---

## 二、Mutex 底层实现

### 核心原理

用户态原子快速路径 + 内核态睡眠挂起。

```
mutex_lock:
  1. 原子 CAS 检查锁是否空闲 → 空闲则直接拿到（0 次系统调用）
  2. 被占则调用 futex(FUTEX_WAIT) 进入内核
  3. 内核挂到等待队列，线程睡眠，CPU 切走
  4. 锁释放时 futex(FUTEX_WAKE) 唤醒

mutex_unlock:
  1. 原子释放锁
  2. 检查是否有等待者 → 有则 futex(FUTEX_WAKE)
```

### Futex（Fast Userspace Mutex）精髓

- **锁空闲时全程用户态**，零系统调用
- **锁被占才进内核**——此时线程反正要睡眠，系统调用代价值得
- 内核只维护等待队列，不轮询

---

## 三、本质差异

| | Spin Lock | Mutex |
|:--|:--|:--|
| 等待方式 | CPU 100% busy-wait | 睡眠 → 切走 CPU → 唤醒 |
| 上下文切换 | 无 | 有（每次 1-5μs 成本） |
| 公平性 | 默认不公平（TAS） | 等待队列通常 FIFO |
| 能否 | |
| 能否在中断用 | 可以（配合关中断） | 不行（会死锁） |
| 临界区长度 | 纳秒级 | 微秒级以上 |

---

## 四、适用场景

### Spin Lock 适合

- [[中断上下文]]（不允许睡眠的地方），如 Linux 内核 `spin_lock_irqsave`
- [[多核 CPU]] 上纳秒级临界区（仅修改几个变量）
- 内核调度器自身的数据结构保护

### Mutex 适合

- 用户态 99% 的场景（文件 I/O、网络、数据库）
- 临界区执行时间 > 两次上下文切换开销（约 1-5μs）
- 锁竞争频繁的场景（spin lock 此时 CPU 空转浪费严重）

### 性能分界经验值

```
临界区 ← 短 ----------- 2μs ----------- 长 →
       |← Spin Lock 舒适区 →|← Mutex 舒适区 →|
```

---

## 五、公平性与活锁

**TAS Spin Lock** 对多核不公平——刚释放锁的核很可能再次抢到，远端的核可能一直拿不到。→ Ticket Lock 诞生（排队号保证 FIFO）。

**Mutex** 等待队列通常 FIFO 排序，但也有优先级继承等机制。

---

## 六、工程实践：混合锁

现代实现先自旋再睡眠：

```c
// 伪代码
void hybrid_lock() {
    if (try_lock()) return;
    for (int i = 0; i < 100; i++) {
        if (try_lock()) return;
        // 如果锁快释放了，继续自旋
    }
    futex_wait(); // 自旋失败，进入睡眠
}
```

Linux 内核 `mutex` 实现了这个优化，glibc 在一些架构上也会先 trylock 再进内核。

---

## 三句话总结

- **Spin Lock** 是"我就在这等着，你快点出来"——纳秒级短临界区、中断上下文专用
- **Mutex** 是"你先用，我睡会，好了叫我"——任何可能阻塞的场景，99% 用户态代码选它
- **现代系统让它们底层帮你选**——但写底层基础设施时仍必须知道区别

---

## 相关概念

[[Spin Lock]] · [[Mutex]] · [[Futex]] · [[中断上下文]] · [[原子操作]] · [[并发控制]]