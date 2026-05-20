---
tags: [ebpf, kernel, tracing, concept]
created: 2026-05-20
source: conversation
---

# eBPF Hook 点原理与对比

> 所有 [[eBPF]] hook 点的本质是将 BPF 程序注入到内核事件执行路径中。区别在于注入点距离事件的远近、注入机制的开销、对内核的侵入性、以及可获得上下文信息的完整度。

## 核心命题

理解 eBPF hook 点，关键在于理解**四个维度**：
1. **注入点离事件有多近**——延迟 vs 信息完整性
2. **注入机制的开销**——中断 (trap) vs 纯函数调用
3. **对内核的侵入性**——动态改写指令 vs 静态预留调用点
4. **可获取的上下文**——寄存器级 vs 结构化参数

---

## [[kprobe]] / kretprobe — 动态插桩

**原理：**
- 在目标函数入口处动态改写指令：x86 上替换首字节为 `int3`（0xCC）
- 触发 `#BP` 异常 → `do_int3` → `kprobe_handler`
- 单步执行原指令后恢复执行流
- 优化变体 `optprobe`：指令间距够大时用 `jmp` 代替 `int3` 减少开销
- kretprobe 劫持返回地址 → 替换为 trampoline → 通过 reserved 的 `kretprobe_instance` 保存上下文

**特性：**
- ✅ 可以探测**任意**内核函数（包括非导出符号）
- ❌ 不能探测 `__kprobes` / `NOKPROBE_SYMBOL` 标注的函数
- ⚠️ 对内联函数不可靠（符号可能不存在）
- 📉 延迟较高：trap → 内核异常处理 → 上下文切换 → handler

**典型场景：** 调试、临时监控、没有 BTF 的老内核

---

## [[fentry]] / fexit — BPF Trampoline

**原理：**
- 依赖 [[BTF]]（BPF Type Format）获取函数签名和参数类型
- 通过 [[BPF trampoline]]（动态生成的机器码）直接调用 BPF 程序
- 本质是**函数调用**而非 trap/异常
- `bpf_tramp_run` 在运行时生成调用链

**特性：**
- 🟢 **延迟最低**——纯函数调用，无 trap 开销
- 🟢 参数类型**自动推导**（BTF 提供）
- 🟢 fexit 可同时访问**参数和返回值**（kretprobe 做不到）
- ❌ 要求函数有 BTF 信息（编译时 `-g` + `CONFIG_DEBUG_INFO_BTF`）
- ❌ 只能用于 BTF 中可见的函数

```c
// fentry 示例：参数名和类型自动可用
SEC("fentry/do_unlinkat")
int BPF_PROG(do_unlinkat, int dfd, struct filename *name)
{
    // dfd 和 name 直接可用
}
```

---

## [[tracepoint]] — 静态插桩

**原理：**
- 内核源码中用 `TRACE_EVENT()` 宏**静态定义**的 hook 点
- 编译时自动生成：结构体定义、`trace_<name>()` 调用函数、导出到 `/sys/kernel/tracing/events/` 的 format 文件
- 运行时通过 RCU 回调机制调用 BPF 程序
- 使用 **[[JUMP_LABEL]]** 优化：未激活时是 NOP（零开销），激活时 patching 成 `jmp callback`

**特性：**
- 🟢 参数稳定、文档完整（format 文件可读）
- 🟢 开销低于 kprobe（无 int3 trap）
- 🟢 JUMP_LABEL：不激活时零开销
- ❌ 只能探测**预定义**的 hook 点（约 1000+ 个）
- ⚠️ 参数是事件结构字段，不是被 hook 函数参数

**补充：raw_tracepoint**
- 跳过格式验证，直接访问原始参数指针
- 性能更好，但需要自己解析参数

---

## [[LSM BPF]] — 安全钩子

**原理：**
- 基于 [[BPF trampoline]]（与 fentry 共享底层机制）
- 内核编译时启用 `CONFIG_BPF_LSM=y`
- 在 LSM 框架的 `security_<hook>()` 位置预留 BPF 调用点
- 注册为**独立的 LSM 模块**（`bpf_lsm`），优先级在 selinux / apparmor **之前**
- 返回 `-EPERM` 拒绝操作——可以在 selinux 之前先达先拒

**特性：**
- 🟢 实现**细粒度的 MAC**（自主访问控制策略）
- 🟢 优先级**高于传统 LSM**
- ❌ 需要 `CONFIG_BPF_LSM=y` + BTF
- ⚠️ 不是所有 LSM hook 点都有对应 BTF 函数

```c
SEC("lsm/file_open")
int BPF_PROG(file_open, struct file *file)
{
    // 返回 0 放行，-EPERM 拒绝
}
```

---

## 其他 Hook 点

| Hook 类型 | 原理简述 | 延迟 | 灵活性 |
|-----------|----------|------|--------|
| [[XDP]] | 网卡驱动层，SKB 分配前直接操作 DMA 页 | 🟢 最低 | 🟡 仅网络 |
| TC | 内核网络栈 ingress/egress classifier | 🟢 低 | 🟡 仅网络 |
| cgroup hooks | cgroup BPF attach 点（sock_ops, sysctl 等） | 🟢 低 | 🟡 cgroup 范围 |
| perf_event | 绑定到硬件 PMU / 软件事件 | 🟡 中 | 🟢 通用 |
| struct_ops | 替换内核结构体函数指针（如 tcp_congestion_ops） | 🟢 最低 | 🔴 特定结构 |

---

## 对比总结

**按抽象层面排序（离原始事件由近到远）：**
```
[XDP] → [fentry] → [kprobe] → [tracepoint/raw_tp] → [LSM/tc/cgroup]
最近                      最灵活                     最稳定
```

**按开销排序（由低到高）：**
```
struct_ops < fentry < tracepoint(不激活=NOP) < kprobe < raw_tracepoint
最快                                              开销高
```

**按可探测范围排序（由广到窄）：**
```
kprobe(任意符号) > fentry(BTF函数) > tracepoint(静态定义) > LSM(security_*)
最多                                                   特定
```

---

## 面试本质洞察

1. **"kprobe 是动态插桩，fentry 是静态编译后通过 BTF 的轻量调用"**——点出底层机制差异的本质

2. **"fentry 的出现标志着 eBPF 的成熟：依赖内核 BTF 元数据和 BPF trampoline 动态生成的直调路径，把 BPF 从一个外挂探针变成了内核的原生扩展机制"**——画出技术演进路径

3. **"tracepoint 的 JUMP_LABEL 优化是内核的 clever trick：未激活时是一个 NOP 指令，运行时 patching 成 jmp callback，生产环境零 overhead"**——展现对内核编译/链接的理解深度

4. **"LSM BPF 本质上是一组安全相关的 fentry hook，但它不是 add-on——它注册为一个独立的 LSM 模块，有自己的 hook order，能在 selinux 之前先拒绝"**——点出架构设计

5. **"kprobe 不适合做生产监控——int3 对 icache 和分支预测有影响，且依赖内部实现细节容易因内核升级失效。宁愿用 tracepoint 或 fentry 替代。"**——体现工程判断力

## 相关概念

[[eBPF]] · [[kprobe]] · [[fentry]] · [[tracepoint]] · [[LSM BPF]] · [[BTF]] · [[BPF trampoline]] · [[XDP]] · [[JUMP_LABEL]]