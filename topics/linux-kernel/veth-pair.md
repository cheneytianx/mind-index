---
tags: [infra, networking, kernel]
created: 2026-05-13
source: conversation
---

# veth pair

> 一对虚拟以太网卡，内核保证从一端发出的数据帧直接被另一端收到，像一根网线的两端。本质上是连接两个 [[Network Namespace]] 的管道。

## 为什么需要 veth pair

[[Network Namespace|网络命名空间]] 将容器与宿主机的网络栈完全隔离（独立的路由表、iptables、网卡设备）。但容器要访问外界，数据包必须跨越这个边界——这正是 veth pair 解决的问题：**它为相邻 netns 提供了一条纯内存的数据通道**。

没有 veth pair 的话，要让一个 netns 里的数据包到达另一个 netns，只能走物理网卡或软件隧道——要么浪费硬件资源，要么协议栈开销过大。

## 内核实现原理

### 数据结构

veth pair 由两个 `struct net_device` 组成，它们之间通过指针互链：

```c
struct veth_priv {
    struct net_device *peer;   // 指向对端网卡
    ...
};
```

每个 `net_device` 的 `priv` 成员指向一个 `veth_priv` 结构体。创建 pair 时内核分配两个设备，互填对方指针。

### 发送路径

当数据包从一端发出时：

```
ndo_start_xmit(skb, dev)
    ↓
peer = veth_priv(dev)->peer  // 拿到对端网卡
    ↓
skb->dev = peer              // skb 设备换到对端
dev_net_set(skb, peer_net)   // netns 切换到对端所在命名空间
    ↓
netif_rx(skb)                // 直接注入对端接收队列
```

关键细节：

1. **纯内存操作**：没有 DMA、没有硬件中断、没有总线传输。一次 `memcpy`（甚至只是指针重映射）的代价。
2. **跨 netns 切换**：`dev_net_set` 是核心——sock buffer 的所属 [[Network Namespace]] 从容器 netns 切换到宿主机 netns，后续所有协议栈处理（路由、iptables）都在新 netns 下进行。
3. **直接入队**：`netif_rx` 将 skb 放入对端的 backlog 队列，触发软中断，由 `NET_RX_SOFTIRQ` 处理。和物理网卡收到的数据包走同一个处理流程。

### 接收端视角

从对端的角度来看，收到的 skb 没有物理层处理，直接从 **链路层（L2）** 开始：

```
netif_rx → enqueue_to_backlog → softirq (NET_RX_SOFTIRQ)
    ↓
__netif_receive_skb → ip_rcv (IPv4) / br_handle_frame (桥接)
    ↓
继续协议栈处理...
```

因此从 veth 接收到的包和物理网卡收到的包在内核中的后续处理路径 **完全一致**——都经过相同的协议栈、防火墙规则、路由决策。这使得容器网络行为和原生网络行为在语义上等价。

### 创建过程

用户态通过 `ip link add` 命令触发：

```bash
ip link add veth0 type veth peer name veth1
```

内核处理路径：

```
socket(AF_NETLINK, ...)
    ↓
rtnetlink_rcv → do_setlink → rtnl_link_create
    ↓
veth_newlink():
    1. alloc_netdev_mqs() × 2     ← 分配两个 net_device
    2. veth_alloc_queues(dev)     ← 分配 tx/rx 队列
    3. peer = 对端
    4. veth_priv(dev1)->peer = dev2
    5. veth_priv(dev2)->peer = dev1
    6. register_netdevice(dev1)   ← 注册设备
    7. register_netdevice(dev2)
```

## 核心特性

### 1. 一对一的对称关系

- 创建时成对出现，销毁时同时销毁
- 两端无主从之分——任何一端都可以发收
- 一端 down 后，对端可以检测到 carrier off（`netif_carrier_off`）

### 2. 无 NAT / 无隧道

- 不修改数据包内容，不封装 L2/L3 头部
- 数据包经过 veth 后保持语义不变（源/目的 MAC、IP 全部保留）

### 3. 延迟接近于零

- 微秒级差异（纯内存拷贝 + 软中断）
- 相比物理网卡（几十微秒硬件中断 + DMA + 驱动），veth 省去了整个硬件链路

## 常见部署模式

### 模式一：容器 bridge 网络

```
容器 netns                宿主机 root netns
┌────────┐               ┌───────────────┐
│  eth0  │ ← veth pair → │  vethXXX      │
│        │               │     ↓         │
└────────┘               │  docker0      │
                          │  (bridge)     │
                          └───────────────┘
```

Docker 为每个容器创建一对 veth，一端放入容器 netns 并重命名为 `eth0`，另一端挂到宿主机侧的 `docker0` [[Linux Bridge]]。这是 [[container-bridge-mode|容器 Bridge 模式]] 的核心拓扑。

### 模式二：容器 macvlan / ipvlan 辅助

某些场景下 veth 用于将容器连接到更复杂的虚拟网络拓扑中，如结合 [[OVS]]（Open vSwitch）。

### 模式三：CNI 插件的基础设备

Kubernetes 集群中，Flannel、Calico、Weave 等 [[CNI]] 插件的底层通常都依赖 veth pair 作为容器与宿主机网络的连接点。

## 与 tun/tap 的对比

| 特性 | veth pair | tun/tap |
|------|-----------|---------|
| 工作层级 | L2（以太网帧） | L3（IP 包）/ L2 |
| 连接对象 | 另一端 veth | 用户态应用程序 |
| 跨 netns | 原生支持 | 手动配置 |
| 性能开销 | 极低（纯内核转发） | 较高（内核↔用户态切换） |
| 典型用途 | 容器网络 | VPN、隧道、用户态协议栈 |

tun/tap 需要在内核态和用户态之间拷贝数据（上下文切换成本高），而 veth 两端都在内核态，转发路径更短。

## 性能特征

- **延迟增加**：~1-5μs（相比 loopback 可忽略，相比物理网卡节省几十 μs 硬件延迟）
- **吞吐量上限**：受 CPU 软中断处理能力限制，通常可达到接近线速（~10Gbps+ 取决于核数）
- **PPS（每秒包数）** 能力远高于物理网卡，因为绕过了 PCIe 和 DMA

## 相关概念

[[Network Namespace]] · [[Linux Bridge]] · [[容器网络-bridge模式数据包路径]] · [[Docker]] · [[CNI]] · [[tun-tap]]