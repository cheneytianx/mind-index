---
tags: [infra, container, networking]
created: 2026-05-13
source: conversation
---

# 容器网络 Bridge 模式：数据包完整路径

## 场景

容器使用 [[Docker]] 默认的 [[bridge网络模式]]，容器 IP `172.17.0.2` 访问外网 `8.8.8.8`。按数据包在每一层的流转，拆解全部步骤。

---

## Step 1：容器进程发出数据（Socket 层）

进程（如 curl）调用 `write()/send()`，内核 [[TCP-IP]] 协议栈接管：

- **传输层**：封装 TCP/UDP 头部（源端口随机 + 目的端口 53/80/443）
- **网络层**：构造 IP 数据包

关键检查发生在 **路由查找**：

```
# 容器内路由表
default via 172.17.0.1 dev eth0        ← 默认路由，下一跳是 docker0 bridge 的 IP
172.17.0.0/16 dev eth0 proto kernel
```

目标 `8.8.8.8` 不在直连网段，匹配默认路由 → 下一跳 `172.17.0.1`，出口 `eth0`。

---

## Step 2：ARP + L2 封装

内核发现下一跳 `172.17.0.1` 需要通过 `eth0` 出去：
- 查 [[ARP]] 缓存，看 `172.17.0.1` 的 MAC 地址
- 未命中则发 ARP 广播："Who has 172.17.0.1?"

这个 ARP 广播依靠 [[veth pair]] 到达宿主机。

---

## Step 3：veth pair 跨 netns 传递（最核心）

容器内的 `eth0` **不是物理网卡**，而是 veth pair 的一端。[[Network Namespace|网络命名空间]] 隔离了容器与宿主机的网络栈。

### 内核实现原理

veth pair 是一对虚拟网卡，内核保证：**从一端发出的数据帧，直接被另一端收到**，纯内存拷贝，不走物理链路。

```
容器 netns                         宿主机 root netns
┌─────────────────┐              ┌──────────────────┐
│  eth0 (veth0@if5)│ ← veth pair →  veth12345        │
│  172.17.0.2/16  │              │  连接到 docker0   │
│  02:42:ac:11:.. │              │                   │
└─────────────────┘              └──────────────────┘
```

关键代码路径：
1. 容器端调用 `dev_queue_xmit(skb)` 发送
2. veth 驱动 `ndo_start_xmit` 找到 peer，直接把 skb **指针**塞到对端接收队列
3. 对端 `netif_rx(skb)` 前进行 **跨 netns 切换**：
   - `skb->dev = peer` — 换到对端网卡设备
   - `dev_net_set(skb->dev, peer_net)` — sock buffer 的所属 netns 切换到宿主机
4. 此后所有协议栈处理都在宿主机 root netns 下执行

---

## Step 4：Linux Bridge 处理（L2 交换）

对端的 veth 插在 `docker0` bridge 上：

```
docker0: 172.17.0.1/16
        ┌────── Linux Bridge ───────────────┐
        │  FDB 表（MAC 地址学习）             │
        │  veth12345  → 02:42:ac:11:00:02   │
        │  eth0       → 宿主机物理 MAC        │
        └───────────────────────────────────┘
```

[[Linux Bridge]] 处理流程：

1. **MAC 学习**：源 MAC（容器 eth0）记录到 FDB 表，关联端口 `veth12345`
2. **FDB 查表**：目的 MAC 是 `docker0` 自身（ARP 解析的下一跳地址）
3. **Bridge 自收**：帧被 bridge 自身的协议栈接收（类似交换机把包送到了自己的上行端口）

---

## Step 5：宿主机路由 + [[iptables]] [[NAT]]

数据包进入宿主机网络栈，再次路由查找：

```
# 宿主机路由表
default via 10.0.0.1 dev eth0     ← 匹配
172.17.0.0/16 dev docker0
10.0.0.0/24 dev eth0
```

目标 `8.8.8.8` 匹配默认路由 → 出口 `eth0`（物理网卡）。

**问题：源 IP 是 172.17.0.2（私有 IP），直接发出路由器会丢包。**

### MASQUERADE（动态 SNAT）

```
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

- 挂在 `nat` 表的 `POSTROUTING` 链
- 触发条件：源 IP 在 `172.17.0.0/16` **且** 出口不是 docker0（即发往外部）
- 动作：将源 IP 从 `172.17.0.2` 改为宿主机物理网卡 IP（如 `10.0.0.2`）
- [[conntrack]] 记录映射关系：
  ```
  orig:  172.17.0.2:54321 → 8.8.8.8:80
  reply: 8.8.8.8:80       → 10.0.0.2:54321
  ```

---

## Step 6：物理 NIC 发送

- 封装 L2 帧：目的 MAC 是网关 `10.0.0.1` 的 MAC
- 物理网卡发出电/光信号

---

## Step 7：回程路径（反向 [[NAT]] + conntrack）

响应从 `8.8.8.8` 返回，目的 IP 是 `10.0.0.2:54321`：

1. 物理网卡接收 → 宿主机协议栈
2. **conntrack 查表**：找到匹配的 `reply` 条目
3. **DNAT**：`10.0.0.2:54321` → `172.17.0.2:54321`
4. 路由查找：`172.17.0.2` 在 docker0 直连网段
5. **Bridge 转发**：查 FDB 表匹配端口 → `veth12345`
6. **veth pair 反向**：veth12345 → 容器内 eth0
7. 容器内协议栈收到，socket 匹配到进程

---

## 完整 Flow 总图

```
容器进程
   │ 系统调用 write/send
   ▼
协议栈 TCP/IP（容器 netns）
   │ 路由 → 下一跳 172.17.0.1 → dev eth0
   ▼
ARP → 找 docker0 MAC
   │
   ▼
veth pair（容器端 eth0）
   │ skb 跨 netns 传递（纯内存，dev_net_set）
   ▼
veth pair（宿主机端 vethXXX）
   │
   ▼
Linux Bridge (docker0)
   │ MAC 学习 + FDB 查表 → 自收
   ▼
宿主机协议栈（root netns）
   │ 路由 → 目标外网 → 走 eth0
   ▼
iptables POSTROUTING MASQUERADE
   │ SNAT: 172.17.0.2 → 10.0.0.2
   │ conntrack 记录映射
   ▼
物理网卡 eth0 → 网关 → 互联网

          ←── 回程 ──→

互联网 → 网关 → 物理网卡 eth0
   │
   ▼
conntrack 查表 → DNAT: 10.0.0.2 → 172.17.0.2
   │
   ▼
路由 → docker0 → Bridge FDB → veth pair
   │
   ▼
容器 eth0 → 协议栈 → 进程 socket
```

## 关键设计点

- **跨 netns 边界靠 veth 驱动**：没有协议栈开销，纯 skb 指针传递 + netns 切换。这也是容器网络延迟接近原生（微秒级差异）的根本原因
- **额外开销来自**：Bridge FDB 查表 + iptables conntrack 记录查询
- **NAT 是必需的**：因为私有 IP 无法在互联网路由，MASQUERADE 是 bridge 模式的标配

## 相关概念

[[Docker]] · [[Linux Bridge]] · [[veth pair]] · [[Network Namespace]] · [[NAT]] · [[iptables]] · [[conntrack]] · [[ARP]] · [[TCP-IP]]