---
tags: [linux-kernel, security, sandbox, container]
created: 2026-06-26
source: conversation
---

# seccomp 与 Linux 安全机制全景

> 一句话定义：Linux 容器/沙箱依赖七层防御纵深，seccomp 是最前线的系统调用级拦截点，排在所有 LSM 钩子之前。

## Linux 沙箱的七层安全体系

```
用户态程序
    │
    ▼
① seccomp-BPF     ← 系统调用号/参数过滤，最早触发
    │
    ▼
② Namespaces      ← 视图隔离（PID/mount/net/user/...）
    │
    ▼
③ Capabilities     ← root 权限原子化拆分
    │
    ▼
④ LSM (SELinux/AppArmor) ← 标签/路径级强制访问控制
    │
    ▼
⑤ Landlock         ← 无特权进程自主文件路径沙箱
    │
    ▼
⑥ cgroups          ← 资源配额与记账
    │
    ▼
⑦ chroot/pivot_root ← 文件系统根视图切换
```

## ① seccomp（Secure Computing Mode）

**保证语义：** 能调哪些系统调用（以及参数约束条件）。

### 两种模式

| | STRICT 模式 | FILTER 模式 |
|---|---|---|
| **引入** | 2005（Andrea Arcangeli） | 2012（Will Drewry, Google Chrome） |
| **允许** | read(0), write(1), _exit(), sigreturn() | 任意 BPF 程序定义的白名单 |
| **违规** | SIGKILL | KILL / ERRNO / TRAP / TRACE / LOG / ALLOW / USER_NOTIF |
| **安装** | `prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT)` | `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)` |

### BPF 程序如何工作

每次系统调用入口，内核运行已安装的 cBPF 程序，输入为 `struct seccomp_data`：

```c
struct seccomp_data {
    int   nr;            // 系统调用号
    __u32 arch;          // 架构（AUDIT_ARCH_X86_64）
    __u64 instruction_pointer;
    __u64 args[6];       // 系统调用的 6 个参数
};
```

输出五种判决：`KILL_PROCESS` / `KILL_THREAD` / `TRAP` / `ERRNO` / `ALLOW`。

### 多过滤器叠加 → 最严裁决

进程可叠加多个过滤器，内核评估取最严格判决——过滤器 A 说 ALLOW，过滤器 B 说 KILL → 最终 KILL。这保证嵌套容器场景的安全性。

### 不可撤销

一旦开启，进程直到死亡不可退出。不能"先开再临时关闭"。

### 局限性

- **不能读指针内容：** args[0] 是地址值，BPF 拿不到指针指向的路径字符串——无法做文件名过滤
- **全局生效：** 对所有线程生效
- **cBPF 限制：** 不能循环、不能分配内存、指令有上限（4096 条）

### 关键进化：seccomp user notification（Linux 5.0+）

`SECCOMP_RET_USER_NOTIF` 把拦截决定权交给另一个用户态 supervisor 进程。内核用 fd 传递系统调用上下文，supervisor 可以代理执行甚至修改参数。Docker/podman 的 rootless mode 依赖此机制模拟特权操作。

## ② Namespaces — 隔离"看到什么"

| namespace | 隔离内容 |
|---|---|
| Mount (mnt) | 文件系统挂载点 |
| PID | 进程编号视图 |
| Network (net) | 网卡、路由表、iptables |
| UTS | 主机名、域名 |
| IPC | 共享内存、信号量、消息队列 |
| User | UID/GID 映射（容器内 root → 宿主机 uid=10000） |
| Cgroup | cgroup 文件系统视图 |
| Time | 系统时钟 |

`docker run` 本质：`clone()` + 七个 `CLONE_NEW*` 标志 = 一个"看不见外面"的进程。

## ③ Capabilities — 限制"能干什么"

把 root 万能权限拆成 40+ 个可独立开关的原子能力。Docker 默认把容器 root 的 capabilities 砍到剩 14 个。

```
传统: uid=0 → 一切允许
现代: uid=0 + CAP_SYS_ADMIN=off → 不能 mount、不能 insmod
```

## ④ LSM (SELinux / AppArmor) — 复杂策略

| | SELinux | AppArmor |
|---|---|---|
| **策略模型** | 基于 label（type/role/user） | 基于路径（/usr/bin/nginx） |
| **复杂度** | 高 | 低 |
| **典型部署** | RHEL/CentOS/Android | Ubuntu/SUSE |

`docker run --security-opt label=type:container_t` 驱动 SELinux 给容器进程打标签，配合 MCS 保证容器互不可见。

## ⑤ Landlock — 无特权自沙箱

2018 年引入，进程不需 root/CAP_SYS_ADMIN 就能限制自己的文件访问：

```c
landlock_create_ruleset(&attr, sizeof(attr), 0);
landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &path_attr, 0);
landlock_restrict_self(ruleset_fd, 0);
// 从此刻起只能访问指定路径
```

与 seccomp 的互补关系：
- seccomp：看系统调用号，不能看文件路径
- Landlock：看文件路径，不管走的是 open/openat/openat2

## ⑥ cgroups — 限制"用多少"

| 控制器 | 功能 |
|---|---|
| cpu | CPU 份额 |
| memory | 内存上限 |
| blkio | 磁盘 IOPS/BW 限制 |
| pids | 进程数上限（防 fork 炸弹） |
| devices | 设备访问控制 |

## ⑦ chroot / pivot_root — 文件系统视图切换

```
chroot("/new/root")         ← 老 syscall，可能被 chdir+chroot 逃逸
pivot_root("/new", "/old")  ← 现代方式，配合 mount namespace 使用
```

## 一次系统调用穿过所有层的示例

容器内 `open("/etc/shadow")` 的完整穿行路径：

```
进程: open("/etc/shadow")
   ↓
① seccomp:   open() 在白名单？→ YES
   ↓
② namespace: /etc/shadow → 宿主机上哪个真实路径？
   ↓
③ capability: 需要 CAP_DAC_OVERRIDE？→ 容器 root 有 → 放行
   ↓
④ LSM:       container_t 能读 /etc/shadow？→ NO → EACCES ❌
   ↓
⑤ Landlock:  (如果进程设了规则) → 路径在黑名单 → 拒绝
```

## 本质洞察

1. **纵深防御，没有银弹：** 每层都有盲区，组合才有意义。Docker 的隔离强不是因为单一技术。
2. **触发顺序反映成本：** seccomp 做粗粒度快速拦截（只看系统调用号和参数寄存器值），LSM 做细粒度策略决策（需要查标签/路径），cgroups 做资源记账——成本递增，精度也递增。
3. **特权不对称：** seccomp 是进程自愿放弃能力的沙箱，不是外部强加的限制。进程可以自己进笼子但不能出来——单向门设计保证了安全性不能被回滚。

## 相关概念

[[seccomp]] · [[namespaces]] · [[capabilities]] · [[SELinux]] · [[AppArmor]] · [[Landlock]] · [[cgroups]] · [[chroot]] · [[pivot_root]] · [[线程单独被杀的内核机制]] · [[linux线程信号与进程级退出]] · [[容器安全纵深防御]]
