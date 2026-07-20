---
tags: [linux, kernel, ebpf, debugging]
created: 2026-07-20
source: conversation
---

# proc-kallsyms 与 BTF vmlinux：内核符号与类型基础设施

> kallsyms 是内核的"电话本"（地址→名字），BTF vmlinux 是内核的"LinkedIn 主页"（完整的类型履历）。两者共同构成现代 Linux 可观测性的地基。

## /proc/kallsyms：内核符号表

### 文件内容

每一行 = 一个内核符号：

```
ffffffff81000000 T _text
ffffffff81000110 T secondary_startup_64
ffffffff81800000 D __bss_start
ffffffffc0123456 t nf_conntrack_init  [nf_conntrack]
```

| 列 | 含义 |
|----|------|
| 地址 | 当前运行时的虚拟地址（受 KASLR 影响） |
| 类型 | T=代码, D=已初始化数据, B=BSS, t=本地代码, R=只读数据... |
| 符号名 | 函数名 / 变量名 |
| [模块名] | 动态加载的模块（可选） |

### 与 /boot/System.map 的区别

| | System.map | /proc/kallsyms |
|---|---|---|
| 来源 | 编译时链接器生成 | 运行时内核动态导出 |
| 更新 | 只对应安装时的内核 | 实时反映当前内核 + 动态加载的模块 |
| KASLR | 地址不对（编译时地址） | 真实运行时地址 |
| 存在前提 | 可能被删除 | 内核运行必定存在 |

### KASLR 与权限控制

现代内核默认开启 KASLR，每次重启地址随机化。非 root 用户看到的地址全是 `0000000000000000`，由 `kernel.kptr_restrict` 控制：

| 值 | 效果 |
|:---:|------|
| 0 | 所有人可见真实地址（不推荐） |
| 1 | root 可见，普通用户全 0（**默认**） |
| 2 | 所有人全 0（含 root） |

### 谁在用 kallsyms

- **perf**：地址→函数名解析
- **Oops / panic 回溯**：`Call Trace` 中 `[<ffffffffc0123456>]` 的翻译
- **ftrace / bpftrace / systemtap**：函数名→地址
- **kprobe**：`p:my_probe do_sys_open` 解析
- **`printk("%pS", addr)`**：内核代码中打印函数指针

### 底层实现

编译时由 `scripts/kallsyms.c` 生成排序过的地址数组 + 变长编码压缩的符号名字符串。运行时二分搜索查找。典型内核的 kallsyms 数据约 2-4 MB。

## /sys/kernel/btf/vmlinux：类型元数据

### kallsyms 回答不了的

kallsyms 告诉你 `do_sys_open` 在哪儿，但回答不了：

```
do_sys_open 的参数类型和个数？
struct task_struct 里有哪些字段？偏移多少？
这个函数指针的完整签名？
```

**BTF（BPF Type Format）就是为此而生。**

### 粒度对比

| 维度 | kallsyms | BTF vmlinux |
|------|:---:|:---:|
| 信息 | 符号名 + 地址 + 类型标记 | 完整类型图：struct/union/enum/typedef/函数签名 |
| 格式 | 纯文本 | BTF 二进制，需工具解析 |
| 编译要求 | 默认 | 需 `CONFIG_DEBUG_INFO_BTF=y` |
| 大小 | ~2-4 MB | ~2-5 MB |

### 实际能力

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format raw | grep -A10 do_sys_open

[100234] FUNC_PROTO '(anon)' ret_type_id=100212 vlen=4
        'dfd' type_id=100235
        'filename' type_id=100203
        'flags' type_id=100235
        'mode' type_id=100209
[100235] FUNC 'do_sys_open' type_id=100234
```

有了类型信息，bpftrace 才能做到：

```bpftrace
kprobe:do_sys_open
{
    printf("%s opened by pid %d", str(arg1), pid);
    // arg1 对应 filename —— 靠 BTF 知道的
}
```

## 历史演进

```
2002 年前：只有 System.map（静态编译时符号表）
    ↓
2002：/proc/kallsyms（运行时符号表，解决"有这个函数"）
    问题：只知道"有什么"，不知道"长什么样"
    ↓
2018（Linux 4.18）：BTF 引入
    从 DWARF 调试信息提取类型数据嵌入 vmlinux
    通过 /sys/kernel/btf/vmlinux 暴露
    ↓
现在：kallsyms + BTF 双剑合璧
```

## 协作核心：CO-RE

现代 eBPF 的 **CO-RE（Compile Once, Run Everywhere）** 靠两者联合：

```
# 普通 kprobe → 只依赖 kallsyms
bpftrace -e 'kprobe:do_sys_open { printf("called\n"); }'

# CO-RE → 同时依赖两者
bpftrace -e 'kprobe:do_sys_open {
    printf("file: %s\n", str(args->filename));
}'
```

- **kallsyms** 提供地址：`do_sys_open` → 真实内存地址
- **BTF** 提供类型：`args->filename` 在 `struct tracepoint__...` 中的偏移——不同内核版本偏移不同，BTF 让 bpftrace 运行时自适应

**没有 BTF：** 每个内核版本都得重新编译 BPF 程序（字段偏移硬编码）。

**有了 BTF（CO-RE）：** 编译一次，运行时通过 BTF 自动适配，跨内核版本通吃。

### 具体例子：访问 task_struct.pid

```c
// 没有 BTF —— 硬编码偏移，换内核就炸
bpf_probe_read(&pid, sizeof(pid), (void *)task + 1184);

// 有了 BTF —— 运行时重定位，跨版本兼容
pid = task->pid;
// BPF 加载器自动：
//   1. 查 BTF 找到 struct task_struct 定义
//   2. 找到 pid 字段偏移（如 1184）
//   3. 把 BPF 字节码中 task->pid 重定位为 task + 1184
```

## 关系全景

```
             ┌────────────────────────────────────────┐
             │          内核二进制 vmlinux              │
             │                                        │
             │  ┌─────────────┐  ┌──────────────────┐ │
             │  │  kallsyms    │  │  BTF 类型数据     │ │
             │  │  压缩符号表   │  │  (从 DWARF 提取)  │ │
             │  │              │  │                  │ │
             │  │ "do_sys_open │  │  函数原型、参数、 │ │
             │  │  → 0xffff... │  │  结构体字段偏移    │ │
             │  │   type: FUNC │  │                  │ │
             │  └──────┬───────┘  └────────┬─────────┘ │
             └─────────┼───────────────────┼───────────┘
                       │                   │
              ┌────────▼───────┐   ┌───────▼──────────┐
              │ /proc/kallsyms │   │ /sys/kernel/btf/ │
              │  (文本, 按行)   │   │   vmlinux        │
              │                │   │  (二进制 BTF)     │
              └────────┬───────┘   └───────┬──────────┘
                       │                   │
                       ▼                   ▼
                 ┌──────────────────────────────┐
                 │         eBPF / 调试工具        │
                 │                              │
                 │  kallsyms → 名字→地址          │
                 │  BTF      → 类型→字段偏移      │
                 │                              │
                 │  合起来 → CO-RE、可移植追踪    │
                 └──────────────────────────────┘
```

## 关键结论

**kallsyms 解决"内核里有什么"，BTF 解决"它们长什么样"。** kallsyms 是地图上的标记点，BTF 是点开标记后的完整文档。没有 kallsyms，内核是黑盒；没有 BTF，黑盒里的东西只能靠猜偏移量。

## 相关概念

[[BTF (BPF Type Format) 详解]] · [[kprobe-tracepoint-fentry-底层原理]] · [[ebpf-hook-points]] · [[linux内核观测技术对比]]

## 参考

- 对话讨论 (2026-07-20)
