
**BTF 是内核数据结构的"类型字典"——告诉 eBPF 程序"struct task_struct 的第 N 个字段是什么类型、叫什么名字、在偏移量多少处"。**

---

## 为什么需要 BTF？——从一个痛点说起

### 没有 BTF 的年代（kprobe 时代）

假设你要读取当前进程的 PID：

```c
// 内核中 task_struct 定义（简化）
struct task_struct {
    // ... 几百个字段 ...
    pid_t pid;        // 在 5.4 内核中 offset = 1208
    // ... 更多字段 ...
};
```

在 kprobe eBPF 程序中，你必须硬编码偏移量：

```c
// 5.4 内核上能跑
pid_t pid;
bpf_probe_read(&pid, sizeof(pid), (void *)task + 1208);
```

**问题**：
- 5.10 内核里 `task_struct` 加了新字段，`pid` 的 offset 变成了 1216
- 换个内核编译选项（开了 CONFIG_XXX），offset 又变了
- **你的 eBPF 程序换个内核就挂了**

### 有 BTF 之后

```c
// CO-RE 方式，用 BTF 自动适配
pid_t pid = BPF_CORE_READ(task, pid);
// 编译时记录："我要读 task_struct 的 pid 字段"
// 加载时 libbpf 查 BTF，自动填入当前内核的正确 offset
```

**不管什么内核版本，只要有 BTF 信息，自动适配。**

---

## BTF 的本质是什么

### 来源（Linux 内核文档 linuxkernel.org.cn/doc/html/latest/bpf/btf.html）

> "BTF (BPF 类型格式) 是一种元数据格式，用于编码与 BPF 程序/映射相关的调试信息。BTF 最初用于描述数据类型。后来，BTF 扩展到包含已定义子例程的函数信息。"

### 类比

| 传统世界 | eBPF 世界 |
|----------|-----------|
| DWARF 调试信息 | BTF |
| Java 的 .class 文件元数据 | BTF |
| 反射 (Reflection) | BTF |
| C 头文件 (.h) | BTF（但是二进制格式，内嵌在 vmlinux 中） |

### BTF vs DWARF

```
DWARF:  完整的调试信息，几十 MB，开发/调试用
BTF:    精简的类型信息，1-5 MB，生产环境可用
```

BTF 是 DWARF 的**极致压缩版**（灵感来自 Sun 的 CTF — Compact C Type Format），只保留类型布局信息，去掉行号、变量生命周期等调试细节。

---

## BTF 记录了什么

### 数据格式

```
BTF Header
    │
    ├── Type Section（类型描述）
    │   ├── BTF_KIND_INT      → int, u32, pid_t 等基本类型
    │   ├── BTF_KIND_STRUCT   → struct task_struct { ... }
    │   ├── BTF_KIND_UNION    → union { ... }
    │   ├── BTF_KIND_ENUM     → enum { ... }
    │   ├── BTF_KIND_PTR      → 指针类型
    │   ├── BTF_KIND_ARRAY    → 数组
    │   ├── BTF_KIND_TYPEDEF  → typedef 别名
    │   ├── BTF_KIND_FUNC     → 函数签名
    │   └── BTF_KIND_FUNC_PROTO → 函数原型（参数类型列表）
    │
    └── String Section（字符串池）
        → 所有类型名、字段名的字符串
```

### 具体例子

对于内核中的：
```c
struct sock {
    struct sock_common  __sk_common;
    // ...
    u16  sk_type;      // offset = 36, size = 2
    u16  sk_protocol;  // offset = 38, size = 2
    // ...
};
```

BTF 会记录：
```
Type #42: BTF_KIND_STRUCT "sock"
  - member "sk_type",    type=u16, offset=288 (bits)  → 36 bytes
  - member "sk_protocol", type=u16, offset=304 (bits) → 38 bytes
  - ...
```

---

## BTF 在哪里？

### 内核侧

```bash
# 现代内核（5.2+, CONFIG_DEBUG_INFO_BTF=y）
# BTF 编译进 vmlinux，运行时通过 sysfs 暴露
$ ls -la /sys/kernel/btf/vmlinux
-r--r--r-- 1 root root 4.5M  ...  /sys/kernel/btf/vmlinux

# 也可以用 bpftool 查看
$ bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
# 生成一个包含所有内核类型定义的头文件（CO-RE 编程用）
```

### eBPF 程序侧

编译 eBPF 程序时，clang 也会在 `.o` 文件中嵌入 BTF section：

```bash
$ llvm-objdump --section-headers my_prog.bpf.o
...
  .BTF           000005a3  ...
  .BTF.ext       000001c8  ...   # 包含 relocation 信息
```

---

## BTF 的核心用途

### 1. CO-RE (Compile Once, Run Everywhere)

这是 BTF 最重要的应用（来源：docs.ebpf.io/concepts/core/）：

> "BPF CO-RE brings together BTF type information, compiler (Clang) support, and BPF loader (libbpf) to allow building cross-version kernel eBPF applications in a single binary."

#### 工作流程

```
编译时（你的开发机，内核 5.10）：
┌──────────────────────────────────────────────┐
│ BPF_CORE_READ(task, pid)                     │
│         │                                     │
│         ▼ Clang                              │
│ 生成 .BTF.ext relocation 记录：               │
│   "需要读取 struct task_struct 的 pid 字段"    │
│   "编译时 offset = 1208"                      │
└──────────────────────────────────────────────┘

加载时（目标机器，内核 5.15）：
┌──────────────────────────────────────────────┐
│ libbpf 加载 .o 文件                           │
│         │                                     │
│         ▼ 读取目标内核的 /sys/kernel/btf/vmlinux│
│         │                                     │
│         ▼ 查到 5.15 上 task_struct.pid offset = 1216│
│         │                                     │
│         ▼ 修正 eBPF 字节码中的 offset：         │
│           1208 → 1216                         │
│         │                                     │
│         ▼ bpf() syscall 加载修正后的程序        │
└──────────────────────────────────────────────┘
```

**一次编译的 .o 文件，在任何带 BTF 的内核上都能正确运行。**

### 2. Fentry/Fexit 的类型安全

回到我们之前的讨论——fentry 为什么能直接访问函数参数且类型安全？

```c
SEC("fentry/tcp_sendmsg")
int BPF_PROG(my_prog, struct sock *sk, struct msghdr *msg, size_t size)
```

**Verifier 的工作**：
1. 通过 BTF 查到 `tcp_sendmsg` 的函数签名：`(struct sock *, struct msghdr *, size_t)`
2. 验证你的 eBPF 程序声明的参数类型是否匹配
3. 当你访问 `sk->sk_protocol` 时，通过 BTF 知道正确的 offset
4. 拒绝越界访问或类型不匹配

**没有 BTF → 没有类型信息 → fentry 无法工作。** 这就是为什么 fentry 要求内核 5.5+ 且 `CONFIG_DEBUG_INFO_BTF=y`。

### 3. BPF Map 的可读性

```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u32);
    __type(value, struct event);
} events SEC(".maps");
```

有了 BTF，`bpftool map dump` 可以直接显示结构化数据，而非裸字节：

```bash
# 没有 BTF
$ bpftool map dump id 5
key: 01 00 00 00  value: c8 00 00 00 48 65 6c 6c ...

# 有 BTF
$ bpftool map dump id 5
[{
    "key": 1,
    "value": {
        "pid": 200,
        "comm": "Hello",
        "timestamp": 1685012345678
    }
}]
```

### 4. 内核函数签名描述（用于 kfunc 调用）

BTF 还描述了哪些内核函数可以被 eBPF 程序直接调用（kfunc），以及它们的参数和返回值类型。

---

## BTF 生态全景

```
┌─────────────────────────────────────────────────────┐
│                    编译时                             │
│  vmlinux.h (由 bpftool 从 BTF 生成)                  │
│      + Clang + __attribute__((preserve_access_index))│
│      → 生成带 .BTF.ext relocation 的 .o 文件         │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                    加载时 (libbpf)                    │
│  读 /sys/kernel/btf/vmlinux                          │
│  对比编译时 BTF vs 运行时 BTF                         │
│  执行 field relocation（修正 offset）                 │
│  执行 type/enum relocation（适配类型变化）             │
│  → 修正后的 eBPF 字节码                              │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                    验证时 (Verifier)                  │
│  用 BTF 做类型检查：                                  │
│  - fentry 参数类型是否匹配函数签名                     │
│  - 结构体字段访问是否越界                             │
│  - map key/value 类型是否一致                         │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                    运行时                             │
│  BPF trampoline 生成时参考 BTF 确定寄存器映射          │
│  bpftool 用 BTF 做人类可读的 dump                    │
└─────────────────────────────────────────────────────┘
```

---

## 总结

| 维度 | 说明 |
|------|------|
| 是什么 | 内核/eBPF 程序的**类型元数据**（精简版 DWARF） |
| 记录什么 | struct 布局、字段名/类型/offset、函数签名 |
| 大小 | 1-5 MB（vs DWARF 几十 MB） |
| 存放位置 | `/sys/kernel/btf/vmlinux`（内核侧）；`.BTF` section（eBPF .o 文件） |
| 核心作用 | ① CO-RE 跨内核版本适配 ② fentry 类型安全 ③ verifier 校验 ④ 可读性 |
| 要求 | 内核 5.2+，`CONFIG_DEBUG_INFO_BTF=y` |
| 类比 | Java 的反射元数据 / TypeScript 的 .d.ts 类型声明 |

**本质上，BTF 让 eBPF 从"写死偏移量的内存操作"进化到了"有类型系统保护的结构化编程"。** 这是 eBPF 从调试工具走向生产级基础设施的关键一步。