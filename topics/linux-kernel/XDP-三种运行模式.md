---
tags: [ebpf, kernel, network]
created: 2026-11-19
source: conversation
---

# XDP 三种运行模式与执行链路

> XDP 的「模式」不是一个维度，而是三个彼此正交的维度：**attach 模式**决定代码插在收包路径的哪个高度，**执行模式**决定字节码怎么变成机器码，**attach 机制**决定生命周期。三者互不影响，混在一起谈必然出错。

## 核心命题

三种 attach 模式的 API **完全一致**（同一个 `struct xdp_md`、同一套 helper、同一批返回码），差异全部来自「插在收包路径的哪个位置」。位置决定了两件事：

1. **你看到的包长什么样**（是否已被 [[GRO]] 聚合、headroom 从哪来）
2. **你的动作有多贵**（`XDP_TX` / `XDP_REDIRECT` 是原生路径还是兜底路径）

一句话：**模式对程序不可见，但对程序的行为后果完全可见。**

---

## 一、三个拦截点

XDP 的本质是「在包进入协议栈之前截住它」。三个模式的区别就是拦截点的高度：

```
收包路径（自上而下）：

  ① 网卡固件
     │  ┌─────────────────────────────┐
     │  │  Offload / HW 模式          │  ← 程序烧在网卡芯片上
     │  │  CPU 完全不参与              │
     │  └─────────────────────────────┘
     ▼
  ② 驱动 NAPI poll
     │  ┌─────────────────────────────┐
     │  │  Native / DRV 模式          │  ← 裸 DMA 缓冲区指针
     │  │  skb 尚未分配                │
     │  └─────────────────────────────┘
     ▼
  ③ 分配 skb
     ▼
  ④ GRO（聚合）
     ▼
  ⑤ __netif_receive_skb_core
     │  ┌─────────────────────────────┐
     │  │  Generic / SKB 模式         │  ← 已有完整 skb
     │  └─────────────────────────────┘
     ▼
  ⑥ tc ingress → 协议栈
```

内核里能直接观察到两条入口（本机 `6.6.117` 实测）：

```
bpf_prog_run_generic_xdp      ← generic 唯一入口
do_xdp_generic                ← generic 调度层
xdp_do_generic_redirect       ← generic 下的重定向兜底
```

Native 模式**没有独立的调度函数**——它被直接内联进各驱动自己的 NAPI poll 里（`bpf_prog_run_xdp()` 内联展开）。这也是为什么 native 需要驱动逐个支持。

---

## 二、规格对比

| | Native / DRV | Generic / SKB | Offload / HW |
|---|---|---|---|
| libbpf flag | `XDP_FLAGS_DRV_MODE` | `XDP_FLAGS_SKB_MODE` | `XDP_FLAGS_HW_MODE` |
| 执行位置 | 驱动收包路径 | 协议栈入口前 | 网卡固件 |
| skb | 尚未分配 | 已分配 | 不存在 |
| 需要驱动支持 | **是** | 否（任何网卡） | **是**（专用 NIC + 厂商工具链） |
| 包长上限 | MTU + L2 头 | 可达 65KB（GRO 聚合） | 硬件缓冲布局 |
| 每包成本 | 最低 | 高（skb + GRO 已付） | 最低，但能力受限于硬件 |
| 可移植性 | 需驱动支持 | 通用 | 极差 |

**本机（`virtio_net`）实测：**

```bash
ip link set dev vx0 xdpgeneric  → 成功
ip link set dev vx0 xdpdrv      → 成功
ip link set dev vx0 xdpoffload  → RTNETLINK answers: Invalid argument
```

`EINVAL` 不是权限问题，是网卡不认——veth 没有任何卸载能力。Offload 模式在现场几乎见不到，主因是程序必须由厂商工具链编译成设备 ISA，helper 集被大幅裁剪。

---

## 三、上下文：`struct xdp_md` 三种模式完全一致

```c
struct xdp_md {
    __u32 data;            /* 包起始虚拟地址（低 32 位） */
    __u32 data_end;        /* 包结束 */
    __u32 data_meta;       /* 元数据区起始，用于跨程序传信息 */
    __u32 ingress_ifindex; /* 入接口 ifindex */
    __u32 rx_queue_index;  /* 接收队列号 */
    __u32 egress_ifindex;  /* 仅重定向场景有意义 */
};
```

**注意：没有 `flags` 字段。** 编译时写 `ctx->flags` 直接报 `no member named 'flags' in 'struct xdp_md'`——这是实证，不是文档推测。

**推论：程序内部无法得知自己处于哪种模式。** 没有字段、没有 helper、没有 kfunc 可以查询。模式是外部环境属性，不是程序可见状态。

字段读取的实测差异：

| 字段 | Native | Generic |
|---|---|---|
| `ingress_ifindex` | 真实 ifindex | 真实 ifindex |
| `rx_queue_index` | 真实硬件队列号 | 常为 0，无意义 |
| `data_meta` 偏移 | 0 | 0 |

`rx_queue_index` 是唯一有实际影响的小差异：native 下可用来做 RSS 感知的 per-queue map 分片；generic 下不要依赖。

---

## 四、数据可见性：唯一真正改变程序行为的地方

这是整篇最关键的实测结论。

### 4.1 TCP：Generic 会看到 GRO 拼出来的超级包

20MB 文件传输，veth MTU 1500，同一个程序对象分别挂两种模式：

```
Native  : pkts=14783  min_len=42  max_len=1514    ← 单帧
Generic : pkts=15236  min_len=42  max_len=65226   ← GRO 聚合
```

那个 65226 字节的「包」**在链路上从未存在过**——它是 CPU 在 `napi_gro_receive` 里拼出来的 skb。一个 skb 内部可能包含十几个连续 IP 段。

### 4.2 UDP：两种模式一致

500 个 1200B datagram：

```
Native  : max_len=1242   ← 单帧
Generic : max_len=1242   ← 也是单帧
```

原因：UDP GRO 需要显式开启（`ethtool -K eth0 rx-udp-gro-forwarding on`），默认关闭。所以纯 UDP 流量不受影响。

### 4.3 后果

**写解析逻辑时不能假设「一个 ctx = 一个网络帧」。** 如果要按 L3/L4 四元组做统计，generic 模式下必须自己拆 GRO 段，否则：
- 包计数偏小
- 包长分布被 65KB 巨包污染
- 按「帧内偏移」解析第二个段会直接崩

---

## 五、headroom 与 `bpf_xdp_adjust_head`

`bpf_xdp_adjust_head(ctx, delta)` 是 XDP 最重要的 helper 之一（加/删包头、做封装、做 [[VXLAN]] 类隧道）。能往前推多远取决于 headroom 来源：

| 模式 | headroom 来源 | 可预期性 |
|---|---|---|
| Native | 驱动预留 `XDP_PACKET_HEADROOM`（通常 256B） | 稳定可预期 |
| Generic | skb 的 `head` 余量，取决于 skb 分配路径 | 不可预期 |
| Offload | 硬件描述符布局 | 通常几乎为 0 |

实测 `bpf_xdp_adjust_head(ctx, -64)` 在两种模式下都返回 0（成功）。但更深的插入在 generic 下会先失败，**阈值不可预知**。

因此必须检查返回值，不要假设成功：

```c
if (bpf_xdp_adjust_head(ctx, -MY_HDR_LEN) < 0)
    return XDP_PASS;   /* 优雅退化，不要硬来 */
```

---

## 六、返回码：语义相同，代价不同

XDP 有五个返回码（`enum xdp_action`），两种模式都支持，但实现路径不同：

| 返回码 | Native | Generic |
|---|---|---|
| `XDP_PASS` | 继续协议栈 | 继续协议栈 |
| `XDP_DROP` | 丢弃，**省掉 skb 分配** | 丢弃，但 skb 已分配，省不了 |
| `XDP_TX` | 网卡直接发回，零拷贝 | 走 `xdp_do_generic_redirect`，需重建 skb |
| `XDP_REDIRECT` | cpumap / devmap 零拷贝 | 兜底为普通转发，有拷贝 |
| `XDP_ABORTED` | 丢弃 + tracepoint 报错 | 同左 |

**关键：`XDP_TX` 和 `XDP_REDIRECT` 在 generic 下基本不可用**——需要构造完整 skb 再重新入栈，性能差一个量级。如果程序核心逻辑是「回包」或「重定向」，别用 generic。

### 硬件配置也会拖垮 Native

本机 dmesg 实录：

```
virtio_net: XDP request 4 queues but max is 2.
XDP_TX and XDP_REDIRECT will operate in a slower locked tx mode.
```

即便 native 模式，**队列数不匹配时 TX/REDIRECT 会退化到带锁路径**。结论：native ≠ 一定快，要看硬件配置。

---

## 七、helper 可用性

`bpftool feature probe kernel` 输出的 helper 列表是**按 prog type 划分，不按 mode 划分**。所以理论上三种模式 helper 集相同：

```
bpf_xdp_adjust_head    bpf_xdp_adjust_meta
bpf_xdp_adjust_tail    bpf_xdp_get_buff_len
bpf_xdp_load_bytes     bpf_xdp_store_bytes
bpf_xdp_output
```

外加全局可用：`bpf_tail_call`、`bpf_ktime_get_ns`、`bpf_get_prandom_u32`、`bpf_map_lookup_elem` 等。

**Offload 是例外**：程序编译成设备 ISA，上面这些基本都不可用。这是 HW 模式可移植性差的根因。

---

## 八、执行模式：字节码怎么变成机器码

这是独立于 attach 模式的第二个维度，很多人忽略。

### 8.1 本机实测：解释器已被编译剔除

```bash
$ grep -E 'CONFIG_BPF_JIT' /boot/config-$(uname -r)
CONFIG_BPF_JIT=y
CONFIG_BPF_JIT_ALWAYS_ON=y      ← 关键
CONFIG_BPF_JIT_DEFAULT_ON=y

$ grep -cE ' (___bpf_prog_run|__bpf_prog_run)$' /proc/kallsyms
0
```

`___bpf_prog_run` 是 eBPF 解释器主循环。`BPF_JIT_ALWAYS_ON=y` 在编译期把它整个 `#ifdef` 掉了。

**结论：内核里没有解释器可用，`net.core.bpf_jit_enable` 只是占位开关，设成 0 也会被忽略。** 所有 XDP 程序必定走 JIT。

### 8.2 证明它真的编译成了原生机器码

```bash
$ bpftool prog show pinned /sys/fs/bpf/probe
1098: xdp  name probe2  tag b484e636e74399e4  gpl
	xlated 592B  jited 321B  memlock 4096B  map_ids 138
```

`xlated` = 验证器输出的规范 eBPF 字节码；`jited` = JIT 生成的 x86-64 机器码。两个数字都在，说明完整走了「验证 → 编译」链路。

`bpftool prog dump jited` 能看到真正的汇编：

```asm
bpf_prog_b484e636e74399e4_probe2:
   0:	nopl	(%rax,%rax)
   7:	pushq	%rbp
   8:	movq	%rsp, %rbp
   b:	subq	$8, %rsp
  15:	movq	%rdi, %rbx            ← rdi = ctx，符合 x86-64 SysV ABI
  24:	movabsq	$-127527195557376, %rdi  ← map 指针被烘成立即数
```

`movabsq $..., %rdi` 这一行是决定性证据：**JIT 在编译期就把 map 地址固化进指令**。解释器做不到这件事。

### 8.3 JIT 后的程序是内核符号

```bash
$ grep bpf_prog_b484e636e74399e4 /proc/kallsyms
ffffffffc011b850 t bpf_prog_b484e636e74399e4_probe2	[bpf]
```

`net.core.bpf_jit_kallsyms=1` 时，JIT 产物作为内核符号挂在可执行内存里。这就是 [[bpftrace]] / perf / ftrace 能直接对它下断点的原因。

### 8.4 JIT 硬化

```bash
net.core.bpf_jit_harden = 0
```

关闭 constant blinding。性能最好；代价是理论上可被用于 JIT spraying。自管服务器上这是合理取舍，共享宿主环境要重新评估。

---

## 九、attach 机制：第三个维度

不是「模式」，但决定生命周期，必须单独说。

```bash
$ grep bpf_xdp_link /proc/kallsyms
ffffffffbcb19b00 T bpf_xdp_link_attach
ffffffffbcb0b7a0 t bpf_xdp_link_update
ffffffffbcb0be80 t bpf_xdp_link_release
```

`BPF_LINK_TYPE_XDP = 6`（`linux/bpf.h`），说明支持 [[bpf_link]] 型 attach。

| 机制 | 生命周期 | 特点 |
|---|---|---|
| `NETLINK_SETLINK`（`ip link set xdp`） | 绑在网卡上，**进程退出后依然挂着** | 传统方式，容易留孤儿程序 |
| `BPF_LINK_CREATE`（`bpf_link`） | 绑在 fd 上，**fd 关闭自动摘除** | 现代方式，libbpf 默认 |

孤儿程序是很实际的问题：调试时反复 attach 会报 `XDP program already attached`，必须先显式 `xdp off` 才能重挂。用 bpf_link 就没这个问题。

---

## 十、libbpf 怎么写

本机 libbpf 1.2.2，**没有** `/usr/include/bpf/xdp.h`（`xdp_program__attach()` 那套高层 API 需要更新版本），要用底层接口：

```c
#include <bpf/bpf.h>
#include <linux/if_link.h>

/* attach —— 模式就是第三个参数 */
int err = bpf_xdp_attach(ifindex, prog_fd,
                         XDP_FLAGS_DRV_MODE,   /* 或 XDP_FLAGS_SKB_MODE */
                         NULL);

/* detach */
bpf_xdp_attach(ifindex, -1, XDP_FLAGS_DRV_MODE, NULL);
```

**不指定 flag 时**：内核先试 native，驱动不支持才回退 generic。

实测 `ip link set eth0 xdp ...`（不带前缀）就是这样——挂上去后 `ip -d` 里没有 `xdpgeneric` 标记，反推走的是 native。

> **建议永远显式指定。** 静默回退会让程序行为在换机器/换驱动时悄悄变化（从「看到单帧」变成「看到 65KB 巨包」），而不是在 attach 时报错。

---

## 十一、怎么判断当前是什么模式

程序内部查不到，只能看外部：

```bash
ip -d link show eth0
```

| 输出中出现 | 模式 |
|---|---|
| 裸 `xdp` | Native / DRV |
| `xdpgeneric` | Generic / SKB |
| `xdpoffload` | Offload / HW |
| 什么都没有 | 未挂载 |

---

## 十二、运行时硬约束（与模式无关）

这些是 eBPF 本身的环境限制，三种模式（除 offload）都适用：

**① softirq 上下文，不可睡眠**

Native 跑在 NAPI poll 里，Generic 跑在 `netif_receive_skb` 里，都是 BH disabled 的软中断上下文。只能用 atomic 语义 helper，不能做任何会 block 的操作。

**② 栈只有 512 字节**

稍大的结构体必须放 map，不能放栈。这是写 XDP 最常踩的坑——解析变长协议头时极易爆栈。

**③ 循环必须有界**

验证器要求可证明有界。用 `#pragma unroll` 或明确上界。

**④ tail call 是唯一的模块化手段**

`bpf_tail_call` 通过 `BPF_MAP_TYPE_PROG_ARRAY` 串起多个程序，每个独立验证、独立 512B 栈。链长上限 33（`MAX_TAIL_CALL_CNT`，内核宏，用户态不可配）。

---

## 十三、写代码的四条硬规则

**① 不要假设包长 ≤ MTU**

```c
__u64 len = ctx->data_end - ctx->data;
/* generic 下可能是 65226，别写死 1514 */
```

**② 每次读数据前都验边界**

```c
if (p + 14 > e) return XDP_PASS;
```

尤其 generic 模式，GRO 段内部偏移与「一帧一包」的假设完全不同。

**③ `adjust_head` 必须检查返回值**

不同模式、不同驱动能给的空间不同，失败要优雅退化。

**④ 别依赖 `rx_queue_index`**

native 下有意义，generic 下无意义。

---

## 十四、一个未验证的观察

实测中 native 累计 ~502 包、generic ~1002 包，稳定差 2 倍，重复两次均复现。

推测是 generic 路径对同一 skb 调用了程序两次（GRO 前后各一次），但**未验证到底**。

**实践影响：如果 XDP 程序用于精确计数（丢包率、流量统计），先用两种模式跑同一份流量校准倍数，不要直接信计数器绝对值。**

---

## 十五、完整执行链路

把三个维度串起来看：

```
① C 源码
        │ clang -target bpf
        ▼
② eBPF 目标文件（ELF，含 .text / BTF / .maps 段）
        │ bpf(BPF_PROG_LOAD)
        ▼
③ 内核验证器
   - 指针安全、循环有界、栈不越界
   - map 引用解析为实际指针
   - 输出规范字节码 → xlated 592B
        │ bpf_prog_select_runtime()
        ▼
④ JIT 编译（本机强制，无解释器）
   - x86-64 后端生成机器码 → jited 321B
   - 分配可执行内存（jit_limit 池）
   - 注册 kallsyms 符号
        │ attach（netlink 或 bpf_link）
        ▼
⑤ 挂到网卡
   ip -d link show eth0 → prog/xdp id 1098
        │ 数据包到达
        ▼
⑥ 运行时：直接 call 到 JIT 代码，无解释开销
```

**从 ⑥ 往回看：执行开销等于一次函数调用。** XDP 能线速的原因不是「特殊」，而是它在 skb 分配之前拿到裸指针，跑的是原生机器码。

---

## 面试本质洞察

1. **「XDP 的三种模式不是三种技术，而是同一个抽象插在收包路径的三个高度上。API 完全一致，差异全在下游：你看到什么样的包，你的动作有多贵。」**——一句话掐断「XDP 有三种实现」的误解。

2. **「Generic 模式看到的是 CPU 拼出来的包，不是链路真实存在的包。一个 65KB 的 skb 里可能包含十几个 IP 段，按帧解析必崩。」**——这是实测出来的、绝大多数资料不会提的坑。

3. **「模式对程序不可见——`struct xdp_md` 里根本没有 mode 字段。所以必须在 attach 时显式指定，不能指望程序自己适配。」**——体现对 API 边界的精确掌握。

4. **「Native 在 skb 分配之前运行，所以 XDP_DROP 连内存分配都省了。Generic 在 skb 和 GRO 之后运行，该付的成本已经付完，只省了后续协议栈。这才是两者性能差的根源，不是『一个快一个慢』这么模糊。」**——把性能差异归因到具体的内存分配时机。

5. **「`BPF_JIT_ALWAYS_ON=y` 时解释器在编译期就被 `#ifdef` 掉了，kallsyms 里查不到 `___bpf_prog_run`。所谓『关闭 JIT』在现代内核上是个不存在的选项。」**——用一条符号查询证伪常见误解。

## 相关笔记

- [[ebpf-hook-points]] — 所有 eBPF hook 点的横向对比，本篇是其 XDP 部分的纵向展开
- [[kprobe-tracepoint-fentry-底层原理]] — 另一种 hook 机制的指令级剖析，可对照看「插桩方式」如何决定能力边界
- [[linux内核观测技术对比]] — 观测/拦截技术的全景定位
- [[veth-pair]] — 本篇实验环境用的网络设备对

## 相关概念

[[eBPF]] · [[XDP]] · [[GRO]] · [[BTF]] · [[bpf_link]] · [[BPF trampoline]] · [[JIT]] · [[softirq]] · [[skb]] · [[NAPI]]
