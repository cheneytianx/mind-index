---
tags: [ebpf, kernel, tracing, concept]
created: 2026-05-23
source: conversation
---

# Kprobe / Tracepoint / Fentry 底层原理

> 三种 [[eBPF]] hook 机制的底层实现剖析——从指令级（int3 trap、NOP/JMP patch、BPF trampoline call）理解它们为什么会表现出不同的性能特征。

## 整体架构

```
用户态 eBPF 程序 (.o)
        │
        ▼ bpf() syscall 加载
┌─────────────────────────────┐
│   eBPF Verifier (安全校验)    │
│   JIT Compiler (编译为机器码) │
└────────────┬────────────────┘
             │ attach 到 hook 点
             ▼
┌─────────────────────────────────────┐
│  内核 Hook 基础设施                   │
│  ┌──────────┬──────────┬──────────┐ │
│  │ kprobe   │tracepoint│ fentry   │ │
│  │(int3断点) │(static   │(BPF      │ │
│  │          │ jump_label)│trampoline)│
│  └──────────┴──────────┴──────────┘ │
└─────────────────────────────────────┘
```

---

## 1. [[kprobe]] — 基于断点指令的动态插桩

> "When a kprobe is registered, Kprobes makes a copy of the probed instruction and replaces the first byte(s) of the probed instruction with a breakpoint instruction (e.g., int3 on i386 and x86_64)."
> — docs.kernel.org/trace/kprobes.html

### 注册：指令改写

```
┌────────────────────────────────────────────────┐
│ 目标函数原始指令:  0x48 0x89 0xe5 (mov rbp,rsp) │
│                          │                      │
│ kprobe 注册后:     0xCC 0x89 0xe5 (int3 + ...)  │
│                     ↑                           │
│              替换第一字节为 int3 (0xCC)            │
│              原始指令保存到 kprobe 结构体           │
└────────────────────────────────────────────────┘
```

### 触发：完整的异常处理路径

```
CPU 执行到 int3
    │
    ▼ 触发 #BP 异常 (trap)
内核 trap handler (do_int3)
    │
    ▼ notifier_call_chain
kprobe 框架接管
    │
    ├─→ 执行 pre_handler（eBPF 程序在此运行）
    │
    ├─→ 单步执行被替换的原始指令（用保存的副本）
    │
    └─→ 执行 post_handler
    │
    ▼ 恢复正常执行流
```

### kretprobe 的补充原理

函数返回探测通过**劫持返回地址**实现：
- 注册时将栈上的返回地址替换为 `kretprobe_trampoline`
- 函数 `ret` 时跳到 trampoline → 执行 handler → 跳回真实返回地址

### 性能开销

| 环节 | 开销 |
|------|------|
| int3 trap（用户态→内核态级别的异常处理） | **高** |
| 保存/恢复寄存器上下文 | 中 |
| 单步执行原始指令 | 中 |
| **总开销** | **~100-200ns per hit** |

---

## 2. [[tracepoint]] — 基于 static_key/[[JUMP_LABEL]] 的静态插桩

> "The advantage of using trace_<tracepoint>_enabled() is that it uses the static_key of the tracepoint to allow the if statement to be implemented with jump labels and avoid conditional branch overhead."
> — kernel.org tracepoints.html

### 内核源码形态（编译时埋入）

```c
// 内核源码中的 tracepoint 调用
trace_sched_switch(prev, next);

// 展开后等价于：
if (static_key_false(&__tracepoint_sched_switch.key)) {
    __DO_TRACE(...)  // 调用注册的 probe 函数
}
```

**这个 `if` 不是普通的条件分支。**

### Jump Label 的 NOP 魔术

```
未激活时（默认）：
┌─────────────────────────────┐
│ ... 正常代码 ...              │
│ NOP (5字节)   ← tracepoint  │  ← CPU 直接滑过，零开销
│ ... 正常代码 ...              │
└─────────────────────────────┘

激活时（有 eBPF 程序 attach）：
┌─────────────────────────────┐
│ ... 正常代码 ...              │
│ JMP <trace_handler>          │  ← text_poke 实时修改指令
│ ... 正常代码 ...              │
└─────────────────────────────┘
```

内核通过 [[text_poke]]（x86）在运行时**原子地将 NOP 替换为 JMP**（或反之）。整个过程：
1. `static_key_enable()` → 遍历所有使用该 key 的 `jump_entry`
2. 调用 `arch_jump_label_transform()` → 计算相对偏移 → `text_poke_bp` 原子改写指令
3. 利用 `int3` 作为中间态保证改写过程中的 CPU 一致性（其他核心不会执行到半改写的指令）

### 性能开销

| 状态 | 开销 |
|------|------|
| 未激活 | **零**（NOP，CPU pipeline 直接跳过） |
| 激活后 | 一次 JMP + 回调函数调用（~30-50ns） |

### 与 kprobe 的本质区别

| 维度 | Kprobe | Tracepoint |
|------|--------|------------|
| 插桩时机 | **运行时动态** | **编译时静态**（代码里写好了位置） |
| 可 hook 位置 | 任意内核地址 | 仅内核开发者预定义的点 |
| 稳定性 | 内核升级可能函数改名/消失 | 被视为 ABI，较稳定 |
| 未激活开销 | 不存在此状态（注册=激活） | 零（NOP） |

### 2.1 tracepoint 的三条挂载路径：tracepoint / raw_tracepoint / tp_btf

上面讲的 `trace_<name>()` 后的 JMP 目标是「tracepoint 的 probe 链」。**往这条链上挂 BPF 程序有三种方式**，埋点本身完全不变，变的是挂载方式和上下文形态：

```
TRACE_EVENT() 生成的 trace_<name>()
        │
        │  static_key → JMP
        ▼
  tracepoint probe 链  ←── ① perf_event_open + PERF_EVENT_IOC_SET_BPF   （4.x）
                       ←── ② bpf(BPF_RAW_TRACEPOINT_OPEN)              （4.17+）
                       ←── ③ BPF_PROG_TYPE_TRACING + attach_btf_id     （5.5+）
```

**① tracepoint（perf 方式）**

```c
SEC("tracepoint/sched/sched_switch")
int prog(void *ctx)   // ctx = struct trace_event_raw_sched_switch *
```

- 走 perf 子系统：打开 tracepoint id → `PERF_EVENT_IOC_SET_BPF` 挂程序
- 上下文是**格式化记录**：开头 `common_type/common_flags/common_preempt_count/common_pid` 头，后面字段按 `/sys/kernel/tracing/events/sched/sched_switch/format` 里写的**固定偏移**排列
- 拿到的 `prev_comm`、`prev_pid` 是内核 `TP_fast_assign` 预先算好的派生值
- 代价：字段偏移随内核版本/编译配置变化（bpftrace 得运行时解析 format 文件）、多一层 perf 注册、受 `perf_event_paranoid` 约束

**② raw_tracepoint**

```c
SEC("raw_tracepoint/sched_switch")
int prog(struct bpf_raw_tracepoint_args *ctx)
{
    bool preempt       = ctx->args[0];
    struct task_struct *prev = (void *)ctx->args[1];
    struct task_struct *next = (void *)ctx->args[2];
    unsigned int prev_state  = ctx->args[3];
}
```

- 直接挂到 tracepoint 的 probe 链上，**不经过 perf**
- 上下文是 `u64 args[]`，**按 `TP_PROTO` 原型顺序原样给指针**：没有头、没有偏移、没有派生值
- 白送的自由度：format 里被裁掉的参数（整个 `task_struct*`、`pt_regs*`）在这里拿得到
- 典型例子 `raw_syscalls:sys_enter`：tracepoint 给 `id, args[6]`（内核解码好的），raw_tp 给 `(struct pt_regs *regs, long id)`（你自己按架构解任意寄存器）
- **开销低于 perf 方式的 tracepoint**——名字里的 raw 指「上下文未格式化」，不是「性能更原始」

**③ tp_btf**

```c
SEC("tp_btf/sched_switch")
int BPF_PROG(prog, bool preempt, struct task_struct *prev,
             struct task_struct *next, unsigned int prev_state)
{
    bpf_printk("%s -> %s", prev->comm, next->comm);
}
```

- `BPF_PROG_TYPE_TRACING` + `attach_btf_id` 定位到 tracepoint 的 BTF 描述
- 参数形态和 raw_tp 一样是原始指针，但 **verifier 知道类型**：写 `prev->comm` 会按 `struct task_struct` 校验；字段偏移由 libbpf/CO-RE 在**加载时重定位**
- 不需要 format 文件，也不需要手写偏移 → raw_tp 的自由度 + tracepoint 的类型安全
- 门槛：5.5+ 且 `CONFIG_DEBUG_INFO_BTF`

**三者对比**

| | tracepoint (perf) | raw_tracepoint | tp_btf |
|---|---|---|---|
| 挂载 | `perf_event_open` | `BPF_RAW_TRACEPOINT_OPEN` | TRACING + attach_btf_id |
| ctx 类型 | `trace_event_raw_*`（带 common 头） | `bpf_raw_tracepoint_args`（裸 `args[]`） | BTF 类型化具名参数 |
| 字段 | 派生值 + 固定偏移 | 原始指针 | 原始指针 + 已知类型 |
| 偏移解析 | 读 format 文件 | 无 | CO-RE 重定位 |
| verifier 类型校验 | 无（当字节偏移） | 无 | **有** |
| 额外限制 | `perf_event_paranoid` | 只要 BPF 权限 | 只要 BPF 权限 |
| 最低内核 | 4.x | 4.17 | 5.5 + BTF |
| 单次开销 | 中 | 低 | 低 |

**本质概括：** tracepoint 是「埋点 + perf 中间层 + 格式化视图」，raw_tp 是「埋点 + BPF 直连 + ABI 原貌」，tp_btf 是「ABI 原貌 + 类型系统」。多一层 perf 换兼容性（不依赖 BTF），少一层换性能与自由度，tp_btf 是两个都要——代价是内核版本门槛。

---

## 3. [[fentry]] / fexit — 基于 [[BPF trampoline]] 的直接调用

> "Ftrace (function trace) is a mechanism in the kernel for observing function execution... BPF trampoline implements zero-overhead tracing."
> — docs.ebpf.io trampolines

### 背景：`__fentry__` 和 [[ftrace]]

GCC 编译内核时，**每个函数入口**都插入一个 5 字节的 `call __fentry__`：

```asm
<some_kernel_function>:
    call __fentry__      ; 5 字节，编译时插入
    push rbp
    mov  rbp, rsp
    ...
```

**默认状态**：boot 阶段这 5 字节被 patch 为 `NOP`（没人在追踪时零开销）。

### BPF Trampoline 机制

当 fentry eBPF 程序 attach 时，分两步：

**Step 1：内核 [[JIT]] 生成 BPF Trampoline**

```
┌────────────────────────────────┐
│ 保存调用者保存的寄存器（保留函数参数）│
│ 调用 eBPF JIT 编译后的机器码       │
│ 恢复寄存器                        │
│ 返回到原函数继续执行               │
└────────────────────────────────┘
```

**Step 2：将函数入口的 NOP patch 为 call**

```
┌─────────────────────────────────────┐
│ <some_kernel_function>:              │
│     call <BPF_trampoline>   ← patch │
│     push rbp                         │
│     mov  rbp, rsp                    │
└─────────────────────────────────────┘
```

### 与 kprobe 的执行路径对比

```
Kprobe 路径（经过完整的异常处理框架）：
  int3 → trap handler → save ctx → kprobe dispatch
  → find handler → call eBPF → single-step original insn
  → restore → iret

Fentry 路径（直接函数调用）：
  call <trampoline> → save regs → call eBPF
  → restore regs → ret
```

> 注意：fentry/tp_btf/LSM 共用 **BPF trampoline** 这套底层机制（动态生成调用桩 + text_poke 改写入口 call），区别只在「挂在哪个函数原型上」。fentry 挂内核普通函数，tp_btf 挂 tracepoint 的 probe 函数，LSM 挂 `security_*` 钩子。

| 维度 | Kprobe | Fentry |
|------|--------|--------|
| 触发机制 | int3 异常（trap） | 直接 call（无异常） |
| 上下文切换 | 需要完整 trap frame | 仅 callee-save 寄存器 |
| 参数访问 | 从 pt_regs 手动解析 | 直接访问函数参数（类型安全） |
| 需要 [[BTF]] | 否 | **是** |
| 最低内核版本 | 2.6+ | 5.5+ |
| 性能 | ~100-200ns | ~10-30ns |

---

## 总结对比

### 性能从高开销到低开销

```
Kprobe (int3 trap)      ████████████████  ~100-200ns
                             ↓
Tracepoint (perf 通路)   ████████          ~30-50ns
                             ↓
raw_tp / tp_btf         █████            ~20-40ns
                             ↓
Fentry (call)           ████              ~10-30ns
```

> raw_tp 与 perf tracepoint 命中同一个 `trace_<name>()`，差的只是 perf 那层注册与格式化；省掉它就能追到接近 fentry 的水平。

### 机制全景

| | Kprobe | Tracepoint | raw_tp | tp_btf | Fentry |
|---|--------|------------|--------|--------|--------|
| 机制 | int3 断点 + trap | static_key + jump_label NOP/JMP | 直挂 probe 链（无 perf） | TRACING + attach_btf_id | ftrace NOP → call trampoline |
| 动态/静态 | 动态（任意地址） | 静态（预定义位置） | 静态（预定义位置） | 静态（预定义位置） | 动态（任意 ftrace 可用函数） |
| ctx 形态 | `pt_regs` 手动解 | 格式化记录（固定偏移） | 裸 `args[]` 指针 | 裸指针 + BTF 类型 | 具名参数 + BTF 类型 |
| 参数类型安全 | ❌ | 🟡（结构字段稳定，偏移靠 format） | ❌ | ✅ | ✅ |
| 需要 BTF | 否 | 否 | 否 | **是** | **是** |
| 未激活开销 | N/A | 零 | 零 | 零 | 零（NOP） |
| ABI 稳定 | ❌ | ✅ | ✅ | ✅ | ❌ |
| 最低内核 | 2.6+ | 4.x | 4.17 | 5.5 | 5.5 |
| 适用场景 | 任意位置调试 | 老内核/需现成派生字段 | 要原始参数 / 极致开销 | 新内核生产首选 | 高性能函数级追踪 |

### 选择建议

- **生产环境优先 tp_btf**（ABI 稳定 + 类型安全 + 开销低，5.5+ 默认选它）
- **没有 BTF 或老内核用 tracepoint**（ABI 稳定，但字段偏移靠 format 解析）
- **要 format 里被裁掉的原始参数、或想省掉 perf 层用 raw_tracepoint**（`task_struct*`/`pt_regs*` 直接在手上）
- **需要追踪非 tracepoint 的任意函数用 fentry**（性能最优，类型安全）
- **特殊位置/极老内核用 kprobe**（万能但慢）

---

## 相关笔记

- [[ebpf-hook-points]] — 更全面的 hook 点概览（含 LSM BPF、XDP、TC、struct_ops 等）
- [[kprobe]] · [[tracepoint]] · [[raw_tracepoint]] · [[tp_btf]] · [[fentry]] · [[BPF trampoline]] · [[BTF]] · [[JUMP_LABEL]] · [[ftrace]] · [[JIT]] · [[text_poke]]