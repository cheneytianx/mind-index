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
Kprobe (int3 trap)  ████████████████  ~100-200ns
                         ↓
Tracepoint (JMP)    ████████          ~30-50ns
                         ↓
Fentry (call)       ████              ~10-30ns
```

### 机制全景

| | Kprobe | Tracepoint | Fentry |
|---|--------|------------|--------|
| 机制 | int3 断点 + trap | static_key + jump_label NOP/JMP | ftrace NOP → call trampoline |
| 动态/静态 | 动态（任意地址） | 静态（预定义位置） | 动态（任意 ftrace 可用函数） |
| 参数类型安全 | ❌ (pt_regs) | ✅ (TRACE_EVENT 定义) | ✅ (BTF) |
| 未激活开销 | N/A | 零 | 零（NOP） |
| ABI 稳定 | ❌ | ✅ | ❌ |
| 适用场景 | 任意位置调试 | 生产环境标准观测 | 高性能函数级追踪 |

### 选择建议

- **生产环境优先 tracepoint**（ABI 稳定，零激活前的开销）
- **需要更多灵活性用 fentry**（性能最优，类型安全）
- **特殊位置/旧内核用 kprobe**（万能但慢）

---

## 相关笔记

- [[ebpf-hook-points]] — 更全面的 hook 点概览（含 LSM BPF、XDP、TC、struct_ops 等）
- [[kprobe]] · [[tracepoint]] · [[fentry]] · [[BPF trampoline]] · [[BTF]] · [[JUMP_LABEL]] · [[ftrace]] · [[JIT]] · [[text_poke]]