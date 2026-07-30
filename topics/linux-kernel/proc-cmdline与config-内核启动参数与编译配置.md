---
tags: [linux-kernel, procfs, boot]
created: 2026-07-30
source: conversation
---

# /proc/cmdline 与 /proc/config.gz — 内核启动参数与编译配置

> `/proc/cmdline` 定义运行时行为（bootloader 传入），`/proc/config.gz` 定义编译时能力（`.config` 内嵌），`/proc/version` 定义编译时身份。三者完整描述内核的「能力 + 行为 + 身份」。

---

## /proc/cmdline — 启动参数

**本质**：bootloader（GRUB/systemd-boot/UEFI）加载内核时传入的命令行参数字符串，内核原样保留在内存中并通过 procfs 暴露。

**底层实现**：简单的静态 buffer 读出，只读不可写。

```c
// fs/proc/cmdline.c
static int cmdline_proc_show(struct seq_file *m, void *v) {
    seq_puts(m, saved_command_line);
    seq_putc(m, '\n');
    return 0;
}
```

**常见参数**：

| 参数 | 作用 |
|---|---|
| `root=UUID=xxx` | 根文件系统位置 |
| `ro` / `rw` | 根文件系统挂载为只读/读写 |
| `init=/sbin/myinit` | 指定 1 号进程 |
| `console=ttyS0,115200` | 串口控制台 |
| `quiet` | 抑制非关键日志 |
| `maxcpus=4` | 限制 CPU 数 |
| `mem=4G` | 限制可用内存 |
| `lsm=lockdown,yama,apparmor` | 启用哪些 LSM |
| `selinux=0` | 控制 SELinux 模式 |

**读取者**：内核初始化阶段、systemd、initramfs 脚本、监控诊断工具、安全审计工具。

---

## /proc/config.gz — 编译配置

**本质**：编译内核时用的 `.config` 文件，gzip 压缩后作为 ELF section 嵌入 vmlinux，运行时通过 procfs 暴露。

**前提条件**：需要内核开启 `CONFIG_IKCONFIG_PROC=y`，否则不存在。

```bash
zcat /proc/config.gz | grep CONFIG_CGROUP
```

**用途**：
- 判断内核是否支持某个特性（eBPF CO-RE、Landlock、user namespaces）
- 容器/沙箱能力探测
- 内核 CVE 影响范围排查
- 可移植二进制/内核模块兼容性验证

---

## /proc/version — 编译身份

编译时嵌入的内核版本 + 编译器信息，用于确定运行的到底是哪个内核镜像。

---

## 三者对比

| | `/proc/cmdline` | `/proc/config.gz` | `/proc/version` |
|---|---|---|---|
| **定义什么** | 运行时行为 | 编译时能力 | 编译时身份 |
| **来源** | bootloader 传入 | `.config` 内嵌 | 编译时嵌入 |
| **能否改变** | 能（改 GRUB 重启） | 不能（需重新编译） | 不能 |
| **是否总是存在** | 是 | 不一定 | 是 |
| **格式** | 原始字符串，几 KB | gzip 压缩，20-50 KB | 一行字符串 |

---

## /boot/config-* vs /proc/config.gz

**同一份 source 的两份拷贝，存在条件独立：**

```
编译时生成的 .config
    ├──→ 发行版惯例：复制到 /boot/config-$(uname -r)  （纯文本，总是可用）
    └──→ 需内核选项：嵌入 vmlinux → /proc/config.gz   （gzip，需 CONFIG_IKCONFIG_PROC）
```

| | `/boot/config-*` | `/proc/config.gz` |
|---|---|---|
| **本质** | `.config` 的纯文本副本 | `.config` 的 gzip 内嵌版 |
| **存储** | `/boot` 目录，普通文件 | 内核镜像内部，procfs 暴露 |
| **是否需要内核选项** | ❌ 不需要 | ✅ `CONFIG_IKCONFIG_PROC=y` |
| **格式** | 纯文本 | gzip 压缩（需 `zcat`） |
| **能否被删除** | ✅ 能（但包管理器会重建） | ❌ 不能（在内核镜像里） |
| **容器内可读** | ❌ 通常不可（/boot 不挂载） | ✅ 可（宿主 procfs） |

**为什么发行版倾向 `/boot/config-*` 而非开启 `IKCONFIG_PROC`**：
1. 不依赖内核编译选项，总是可用
2. 省内存（不把配置嵌入内核镜像）
3. 绝大多数排查场景够用

**何时必须用 `/proc/config.gz`**：
- 容器内无法访问宿主机 `/boot`
- 自编译内核需确保版本匹配

---

## 相关概念

[[seccomp与Linux安全机制全景]] · [[BTF (BPF Type Format) 详解]] · [[ebpf-hook-points]] · [[容器安全纵深防御]]
