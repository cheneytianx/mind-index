---
tags: [linux, kernel, concept, process]
created: 2026-05-23
source: conversation
---

# Linux 内核 ID 详解：PID、TID、TGID、PGID、SID

> 从内核数据结构 `task_struct` 出发，彻底理清 Linux 中各类 ID 的底层含义与相互关系。

## 核心数据结构：`task_struct`

内核中每个**执行实体**（进程或线程）都是一个 `task_struct`。内核调度器调度的单位是 `task_struct`，不是传统意义上的"进程"。

关键字段：

```c
struct task_struct {
    pid_t   pid;      // 全局唯一的执行实体 ID
    pid_t   tgid;     // 线程组 ID（用户态 getpid() 返回这个）

    struct task_struct *group_leader;  // 线程组 leader

    // 不同类型的 pid 哈希表链接
    struct pid_link   pids[PIDTYPE_MAX];

    // signal 相关 → 线程组共享
    struct signal_struct *signal;
    struct sighand_struct *sighand;
};
```

内核用 `struct pid` 管理命名空间层面的 PID 映射，但核心逻辑围绕上述字段展开。

---

## PID — 内核视角的"执行实体 ID"

**`task_struct.pid` 是全局唯一标识符**。每个 task_struct 都有一个唯一的 pid，无论是进程还是线程。

- `fork()` → 创建新 `task_struct`，`pid = 新分配值`
- `clone()` → 同上
- 用户态 `gettid()` 返回的就是这个 `pid`

用户态**永远不会直接看到这个值叫 "PID"**——用户态叫它 TID。

---

## TID 与 TGID — 最容易混淆的一对

### 关系总结

| 视角 | 名称 | 对应字段 |
|------|------|----------|
| 内核 | PID | `task_struct.pid` |
| 内核 | TGID | `task_struct.tgid` |
| 用户态 | TID | `task_struct.pid`（内核 PID） |
| 用户态 | PID | `task_struct.tgid`（线程组 ID） |

**一句话**：用户态 PID = 内核 TGID；用户态 TID = 内核 PID。

### 多线程进程的内存映射

```
用户态视角：
  PID = 1234（进程 ID）
  TID = 1234, 1235, 1236, 1237（线程 ID）

内核 task_struct 布局：
  A: pid=1234, tgid=1234, group_leader=A  ← 线程组 leader
  B: pid=1235, tgid=1234, group_leader=A
  C: pid=1236, tgid=1234, group_leader=A
  D: pid=1237, tgid=1234, group_leader=A
```

### 关键行为

- `getpid()` 读 `tgid`
- `gettid()` 读 `pid`
- **`execve()` 不改变任何 ID**——只替换地址空间
- `pthread_create()` 不改变已有线程的 pid/tgid，仅新线程获得新 pid

---

## 内核创建逻辑：`copy_process` 内部

```c
p->pid = alloc_pid(ns);      // 分配全新唯一 pid

if (clone_flags & CLONE_THREAD) {
    // 创建线程：共享 tgid
    p->tgid = current->tgid;
    p->group_leader = current->group_leader;
} else {
    // 创建进程：tgid = 新 pid
    p->tgid = p->pid;
    p->group_leader = p;
}

// PGID 和 SID：无论 fork 还是 clone，都继承
p->pgrp    = current->pgrp;
p->session = current->session;
```

---

## 各类 ID 在不同操作下的变化速查表

| 操作 | PID（内核） | TGID | PGID | SID | 备注 |
|------|------------|------|------|-----|------|
| `fork()` | 新 | 新（= pid） | 继承 | 继承 | 创建新进程 |
| `clone(CLONE_THREAD)` | 新 | 继承 | 继承 | 继承 | 创建新线程 |
| `execve()` | 不变 | 不变 | 不变 | 不变 | 只换地址空间 |
| `setpgid()` | 不变 | 不变 | 变 | 不变 | 移动进程组 |
| `setsid()` | 不变 | 不变 | 变（新） | 变（新） | 创建新会话 |
| `exit()` | 回收 | — | — | — | 终止；孤儿进程组可能受影响 |

---

## PGID — 进程组 ID

一组进程可以组成一个进程组，用于作业控制和信号分发。

### Shell 管道示例

```bash
cat file.txt | grep "error" | wc -l
```

Shell 创建**一个进程组**，cat、grep、wc 三个进程共享同一个 PGID。

### 信号分发

- `kill(-pgid, SIGTERM)` → 发给整个进程组
- `Ctrl+C` → 终端向前台进程组发 `SIGINT`
- `Ctrl+Z` → 向前台进程组发 `SIGTSTP`

### `setpgid()` 的限制

1. 只能修改自己或**未 exec 的子进程**的 PGID
2. 子进程 `execve()` 后父进程不能再改
3. 不能跨会话移动

---

## SID — 会话 ID

会话是一组进程组的集合，主要用于终端管理。

### `setsid()` 的条件与效果

- 调用者**不能是进程组 leader**（否则返回 `-EPERM`）
- 成功 → 新 SID = 自己的 PID，新 PGID = 自己的 PID，**脱离控制终端**

### 守护进程化范式

```c
pid_t pid = fork();
if (pid > 0) exit(0);    // 父进程退出
setsid();                  // 子进程创建新会话，脱离终端
// 此后不受终端挂断信号影响
```

**为什么先 fork 再 setsid？** 直接 `setsid()` 时如果调用者是进程组 leader 会失败。先 fork 保证子进程不是进程组 leader。

---

## 完整层级关系示意

```
Session (SID = 1000)
├── 进程组 (PGID = 2000)         ← 前台，绑定终端
│   ├── 进程 (tgid=2000, pid=2000)
│   ├── 进程 (tgid=3000, pid=3000/3001/3002)  ← 多线程
│   │   ├── 线程 (tgid=3000, pid=3000) ← group_leader
│   │   ├── 线程 (tgid=3000, pid=3001)
│   │   └── 线程 (tgid=3000, pid=3002)
│   └── 进程 (tgid=4000, pid=4000)
├── 进程组 (PGID = 5000)         ← 后台
│   └── 进程 (tgid=5000, pid=5000)
```

## 查看命令

```bash
ps -eo pid,tid,pgid,sid,tgid,comm
ls /proc/<pid>/task/            # 查看进程内所有线程
cat /proc/<pid>/status | grep -E "Pid|Tgid|PPid"
cat /proc/<tid>/status          # 每个线程也有独立的 /proc 目录
```

---

## 边界情况：进程组 Leader 退出

- 进程组 **仍然存活**，只要组内还有其他进程
- Leader 的 PID 就是 PGID，Leader 退出后该 PID 可能被回收
- **内核分配新 PID 时会跳过仍在使用的 PGID 和 SID**，防止新进程意外收到旧信号
- 若组内所有进程退出，PGID 可被回收复用

### 孤儿进程组

进程组 leader 退出 + 组内其他进程的父进程在不同会话中 → 这些进程收到 `SIGHUP` + `SIGCONT`。内核用此机制防止"悬空进程组"挂载在已无效的终端上。

---

## 相关概念

[[task_struct]] · [[fork]] · [[clone (系统调用)]] · [[execve]] · [[进程组]] · [[会话 (Linux)]] · [[守护进程]] · [[控制终端]] · [[孤儿进程组]] · [[PID 命名空间]] · [[信号 (Linux)]]