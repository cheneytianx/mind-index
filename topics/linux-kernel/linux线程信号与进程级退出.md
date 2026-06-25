---
tags: [linux-kernel, signal, thread, concurrency]
created: 2026-06-25
source: conversation
---

# Linux 线程信号与进程级退出

> 一句话定义：Linux 中无法用信号"只杀死一个线程而不影响进程"。所有致命信号最终都走 `do_group_exit()`，内核刻意不允许线程组部分存活。

## 背景 / 问题

Linux 中每个线程本质是一个 LWP（Light Weight Process），拥有独立的 `task_struct` 和 TID（Thread ID）。初看似乎可以用 `kill -9 <TID>` 定向杀死单个线程，但实际上 **SIGKILL 无论发给哪个 TID，整个线程组必死**。

## 核心内容

### 线程标识层级

```
主线程：PID == TGID == TID
子线程：PID == TGID（与主线程相同），TID 不同
```

- `PID` → 实际是 `TGID`（线程组 ID）
- `TID` → 每个线程的唯一标识
- 所有线程共享：`mm_struct`（地址空间）、`fd_table`、`signal_struct`

详见 [[linux内核id详解]]。

### 信号处置路径

致命信号按内核标志分为两类，最终都到 `do_group_exit()`：

| 信号类型 | 内核标志 | 最终调用 |
|---------|---------|---------|
| SIGKILL | `sig_kernel_exit()`，无条件 group exit | `do_group_exit()` |
| SIGTERM/SIGINT/SIGHUP/SIGQUIT/SIGPIPE... | 默认处置为 terminate | `do_group_exit()` |
| SIGSEGV/SIGBUS/SIGFPE/SIGILL... | `sig_kernel_coredump()` → `SIGNAL_GROUP_COREDUMP` | coredump → `do_group_exit()` |

### 为什么内核不允许部分线程存活

这不是疏忽，是**共享地址空间的必然约束**：

1. **Coredump 完整性**：所有线程共享 `mm_struct`，只 dump 一个线程看到的内存是损坏快照，无调试价值
2. **不可恢复性**：一个线程 SIGSEGV 意味着共享地址空间已损坏，继续运行 = 传播错误
3. **无隔离边界**：线程之间没有内存保护，一个线程的故障不能假定对其他线程无影响
4. **资源泄漏**：强制移除线程 → 其栈（mmap 区域）永不回收，持有的 futex 永久锁定，`pthread_join` 行为未定义

### 如果某个线程真的单独退出了

在边缘场景（如线程自身 `pthread_exit()` 或被 `pthread_cancel()` 成功取消），可能出现单线程退出：

| 该线程持有的资源 | 后果 |
|---------------|------|
| 互斥锁（非 robust）| 锁被遗弃 → **死锁**，其他等待线程永久卡住 |
| Robust mutex | 其他线程收到 `EOWNERDEAD`，有机会恢复 |
| 自旋锁/内核锁 | 在内核态被干掉 → 内核 panic |
| 栈内存 | 线程栈不被回收，直到进程结束 |
| 共享数据写入一半 | 其余线程读到半成品、数据不一致 |
| fd/引用计数 | 持有 fd 未 close → 资源泄漏 |

### 信号定向投递

虽然致命信号总走 group exit，**非致命信号可以定向投递**：

- `tgkill(tgid, tid, sig)` — 向指定 TID 发信号
- `pthread_kill(thread, sig)` — 用户态封装
- `SIGRTMIN+N` 实时信号 + 自定义 handler → 在 handler 中 `pthread_exit()` 实现可控单线程退出

## 关键结论

> Linux 内核在信号处理路径上，通过 `do_group_exit` 从根本上禁止了"定向杀单线程"。所有致命信号必然影响整个线程组。要终止单个线程的执行逻辑，唯一安全方式是**用户态协作**：标志位 + `pthread_exit()`，或 `pthread_cancel()` 配合 cancellation point。

## 相关概念

[[linux内核id详解]] · [[task_struct]] · [[致命信号与信号处置]] · [[do_group_exit]] · [[mm_struct]] · [[futex]] · [[pthread线程终止]] · [[Coredump]] · [[spin-lock-vs-mutex]] · [[健壮互斥锁]]
