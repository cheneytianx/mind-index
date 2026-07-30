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

> 核心范式转变：传统上安全策略由管理员（root）从外部施加，Landlock 让**普通进程自己限制自己能做什么**——不需要任何特权。

### 设计哲学

类比：你把自己关进房间里，从内部锁上门，钥匙扔掉。门是内核已有的 LSM hook 机制，Landlock 给了你一把「不需要管理员批准」的锁。

**与 seccomp 共同的「单向门」设计：** 规则只能叠加收紧，不可撤销。进程可以自己进笼子但不能出来。

### 基本用法

2018 年引入（Linux 5.13 正式合入），进程不需 root/CAP_SYS_ADMIN：

```c
// 第一层：只允许读 /tmp
struct landlock_ruleset_attr ruleset_attr = {
    .handled_access_fs = LANDLOCK_ACCESS_FS_READ_FILE |
                         LANDLOCK_ACCESS_FS_WRITE_FILE,
};
int ruleset_fd = landlock_create_ruleset(&ruleset_attr, sizeof(ruleset_attr), 0);

struct landlock_path_beneath_attr path_beneath = {
    .allowed_access = LANDLOCK_ACCESS_FS_READ_FILE,
    .parent_fd = open("/tmp", O_PATH),
};
landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &path_beneath, 0);
landlock_restrict_self(ruleset_fd, 0);
// 从此刻起只能访问指定路径

// 第二层叠加：进一步禁止读 /tmp/secret
// 即使第一层允许，第二层可以收紧
```

### 底层实现

Landlock 基于 eBPF 实现，但不是让用户直接写 BPF 程序。内核在 `security_file_open()` 等 LSM hook 点检查 Landlock 规则。

调用栈：
```
用户进程 → open() → fd_open() → security_file_open() → landlock_file_open()
                                                              │
                                                              ▼
                                              遍历 inode 的 landlock 规则域
                                              检查 access_mask 是否允许
```

核心数据结构（全新实现，非复用 SELinux/AppArmor）：
```c
struct landlock_object {
    struct landlock_ruleset *ruleset;  // 指向规则域
};

struct landlock_ruleset {
    struct landlock_hierarchy *hierarchy;  // 层级（用于叠加）
    // ... eBPF map 数据
};
```

与 SELinux 的 xattr + AVC 缓存、AppArmor 的路径名匹配不同，Landlock 用 **eBPF maps + inode 层级遍历**，在匹配算法、缓存策略、规则继承模型上完全是独立实现。

### 支持的操作类别

| 类别 | 示例 | 引入版本 |
|---|---|---|
| 文件读/写 | `READ_FILE`, `WRITE_FILE` | 5.13 |
| 文件执行 | `EXECUTE` | 5.13 |
| 目录操作 | `REMOVE_DIR`, `MAKE_DIR`, `MAKE_REG` | 5.13 |
| 文件删除 | `REMOVE_FILE` | 5.13 |
| 符号链接 | `MAKE_SYM`, `REFER` | 5.13 |
| 截断 | `TRUNCATE` | 6.2 |
| ioctl | `IOCTL_DEV` | 5.15 |
| 网络绑定/连接 | TCP 端口控制 | 6.7 |

### 与 seccomp 的互补关系

这是 Linux 安全机制中最重要的一对互补组合——两者都是「零特权自沙箱」，但检查粒度完全不同：

| | seccomp | Landlock |
|---|---|---|
| **检查对象** | 系统调用号 + 寄存器参数值 | inode 层级（文件路径/目录树） |
| **能看到字符串吗** | ❌ args[0] 只是地址值，BPF 拿不到指针内容 | ✅ 直接看文件路径 |
| **盲区** | 无法区分 `open("/etc/passwd")` vs `open("/tmp/safe")` | 不管走的是 `open/openat/openat2` 哪个系统调用 |
| **典型场景** | 「不准调 mount()」 | 「可以调 open()，但只准开 /tmp/safe/ 下的文件」 |

两者组合才能覆盖完整攻击面：seccomp 禁止不必要的系统调用类，Landlock 对允许的系统调用做路径级精准控制。

### 不可绕过

- 被 Landlock 限制后，**`execve()` 不能逃避——子进程继承相同的规则域**
- 规则在内核中执行，进程再怎么被攻破也绕不过去
- 一个被提示注入的 Agent，如果底层跑了 Landlock 限制文件访问路径，想读 `~/.ssh/id_rsa` 也读不到——不是 Agent 代码在做检查，是内核在 `open()` 路径上拦截

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

## 七层机制横向对比

### 谁能启用？（特权门槛）

| 机制 | 需要什么特权 | 谁做决策 |
|------|------------|---------|
| **seccomp** | **不需要特权**（进程自愿） | 进程自身 |
| **Landlock** | **不需要特权** | 进程自身 |
| Namespaces | 部分需要 `CAP_SYS_ADMIN` | 编排器/管理员 |
| Capabilities | 需要 root（设置时） | 编排器/管理员 |
| SELinux/AppArmor | 需要 root | 管理员/发行版 |
| cgroups | 需要 root（cgroup v1） | 编排器/管理员 |
| chroot/pivot_root | 需要 `CAP_SYS_CHROOT` | 进程自身/编排器 |

**Landlock 和 seccomp 是唯二不需要任何特权就能由进程自己开启的机制。** 这改变了安全策略的发起者——从「管理员外部施加」到「进程自愿限制自己」。

### 检查什么？（粒度差异）

| 机制 | 检查粒度 | 匹配引擎 |
|------|---------|---------|
| seccomp | 系统调用号 + 6 个寄存器参数 | cBPF 程序 |
| Landlock | inode 层级（文件路径/目录树） | eBPF maps + inode 遍历 |
| SELinux | 文件/进程安全标签（type/role/user） | xattr + AVC 缓存（哈希表） |
| AppArmor | 路径名模式匹配 | profile 匹配引擎 |
| Capabilities | 能力位掩码 | 位运算 |
| Namespaces | 命名空间 ID 映射 | 内核命名空间结构体 |
| cgroups | 资源使用量 | 层级控制器 |

### 规则语义模型

| 机制 | 规则方向 | 能否撤销 | 能否放行已拒绝操作 |
|------|---------|---------|-------------------|
| seccomp | 只能收紧（叠加取最严格） | ❌ 不可逆 | ❌ |
| Landlock | 只能收紧（叠加取最严格） | ❌ 不可逆 | ❌ |
| SELinux | 管理员可放可收 | ✅ 实时更新 | ✅ |
| AppArmor | 管理员可放可收 | ✅ 实时更新 | ✅ |
| Capabilities | 可增可减 | ✅ | ✅ |

**「单向门」是 Landlock 和 seccomp 独有的安全属性：** 进程一旦进入限制状态，就不可能自行退出。这保证了安全性不会被进程自身的后续行为回滚。

### Landlock vs SELinux/AppArmor：不是同一个内核逻辑的不同入口

虽然三者都挂在 LSM 框架的同一组 hook 点上（`security_file_open()` 等），但底层有本质差异：

| 维度 | SELinux | AppArmor | Landlock |
|------|---------|----------|----------|
| **安全决策发起人** | 管理员（root） | 管理员（root） | **进程自己** |
| **规则来源** | 系统策略文件 | profile 文件 | **进程 API 调用** |
| **需要特权吗** | 需要 | 需要 | **不需要** |
| **规则类型** | 基于标签的 Type Enforcement | 基于路径名模式 | **基于 inode 层级 + 只收不放** |
| **存储机制** | `security.selinux` xattr | profile 文件 | **eBPF maps** |
| **匹配缓存** | AVC（Access Vector Cache）哈希表 | profile 匹配缓存 | **inode 链表遍历** |

结论：LSM 框架提供了统一的 hook 点，但 Landlock 的叠加语义、eBPF 存储、inode 遍历匹配都是**全新实现**，不是「换个更友好的 API 皮」。真正的创新在于**范式转变**——谁有权限制谁。

## 本质洞察

1. **纵深防御，没有银弹：** 每层都有盲区，组合才有意义。Docker 的隔离强不是因为单一技术。
2. **触发顺序反映成本：** seccomp 做粗粒度快速拦截（只看系统调用号和参数寄存器值），LSM 做细粒度策略决策（需要查标签/路径），cgroups 做资源记账——成本递增，精度也递增。
3. **特权不对称与单向门：** seccomp 和 Landlock 是进程自愿放弃能力的沙箱，不是外部强加的限制。进程可以自己进笼子但不能出来——单向门设计保证了安全性不能被回滚。
4. **Landlock 的范式创新：** 真正改变的不是技术实现，而是「谁有权限制谁」——从管理员特权变成了进程自助能力。seccomp 也是自限制，但它看不到文件路径；Landlock 是第一个既能看路径、又不需要特权的机制。
5. **seccomp + Landlock = 完整自沙箱：** seccomp 禁止系统调用类，Landlock 对允许的调用做路径级精准控制，两者组合覆盖了从 syscall 到 file path 的完整攻击面，且都不需要 root。

## 相关概念

[[seccomp]] · [[namespaces]] · [[capabilities]] · [[SELinux]] · [[AppArmor]] · [[Landlock]] · [[cgroups]] · [[chroot]] · [[pivot_root]] · [[线程单独被杀的内核机制]] · [[linux线程信号与进程级退出]] · [[容器安全纵深防御]]
