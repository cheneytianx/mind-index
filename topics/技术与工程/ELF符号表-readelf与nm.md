---
tags: [linux, elf, linker, binary, toolchain, concept]
created: 2026-07-21
source: conversation
---

# ELF 符号表：readelf、nm 与符号类型全景

> 符号表是 ELF 文件中连接"人类可读的代码"和"机器执行的二进制"的翻译层。理解了符号表，就理解了链接器与加载器的工作原理。

## 一、先建立坐标系：符号表在整个 ELF 中的位置

### 1.1 ELF 文件的两种面貌

ELF 文件不是一成不变的格式——同一个结构，两种视角：

```
        ┌──────────┐          ┌──────────┐
        │  Linking │          │Execution │
        │  View    │          │  View    │
        └──────────┘          └──────────┘
        (链接视图)             (执行视图)

   ELF Header              ELF Header
   ├─ Sections ──┐         ├─ Program Header Table
   │  .text      │         ├─ Segment 1 (LOAD)
   │  .data      │         │   ├─ .text
   │  .rodata    │         │   └─ .rodata
   │  .symtab    │         ├─ Segment 2 (LOAD)
   │  .strtab    │         │   ├─ .data
   │  .dynsym    │         │   └─ .bss
   │  ...        │         ├─ Segment 3 (DYNAMIC)
   └─────────────┘         │   ├─ .dynamic
                           │   ├─ .dynsym
   Section Header Table    │   └─ .dynstr
                           └─ Section Header Table (optional)
```

- **链接视图**：编译器和链接器关心，按 Section 组织。`.symtab` 就在这。
- **执行视图**：内核和动态链接器关心，按 Segment 组织。`.dynsym` 必须被 LOAD。

**这是第一个关键认知：你分析 ELF 时其实同时在和两套不同的"地图"打交道，readelf 的 `-S`（Sections）和 `-l`（Segments）就是打开这两张不同的地图。**

### 1.2 符号表在文件中的物理位置

```
ELF File:
┌─────────────────────┐
│   ELF Header        │  e_shoff  → Section Header Table 的偏移
├─────────────────────┤  e_shnum  → Section 数量
│                     │  e_shstrndx → Section Name String Table 的索引
│   Program Headers   │
│                     │
├─────────────────────┤
│   .text             │  代码
│   .rodata           │  只读数据
│   .data             │  已初始化全局变量
│   .bss              │  未初始化全局变量（不占文件空间）
│   .symtab           │  ★ 符号表（完整版，包含所有符号）
│   .strtab           │  ★ 符号名字符串池（.symtab 引用它）
│   .dynsym           │  ★ 动态符号表（只有动态链接需要的符号）
│   .dynstr           │  ★ 动态字符串池
│   .rela.text        │  重定位表
│   ...               │
├─────────────────────┤
│ Section Header Table│  描述每个 Section 的元数据
└─────────────────────┘
```

**注意：.symtab 和 .strtab 可以被 strip 掉——发行版的二进制通常就没有它们。但 .dynsym 和 .dynstr 不行，因为运行时动态链接需要。**

---

## 二、符号表本身的结构——一行符号究竟是什么

### 2.1 符号在 ELF 中的定义

每一条符号在 ELF 规范中是一个结构体（64 位版）：

```c
typedef struct {
    uint32_t    st_name;    // 符号名在字符串表中的索引（不是字符串本身！）
    uint8_t     st_info;    // 高 4 位=BIND, 低 4 位=TYPE
    uint8_t     st_other;   // 可见性（visibility）
    uint16_t    st_shndx;   // 所在 Section 的索引
    Elf64_Addr  st_value;   // 符号的地址/值
    uint64_t    st_size;    // 符号的大小（对函数是代码段长度，对变量是 sizeof）
} Elf64_Sym;
```

**每个字段都不是随便设计的——理解它们的含义才能理解后面所有的符号类型标记。**

### 2.2 st_name：为什么符号名另存为独立字符串表

```
.symtab                           .strtab
┌──────────────────┐         ┌─────────────────────┐
│ st_name = 1 ──────────────→ │ \0 m a i n \0 f o o \0 ... │
│ st_info = FUNC    │         └─────────────────────┘
│ st_value = 0x1140 │
└──────────────────┘
```

符号表条目本身是定长的（64 位下每个 24 字节），所有符号名集中存在 `.strtab` 中。这样做的好处：

1. **符号条目定长，二分搜索可行**——`readelf -s` 遍历时 O(n)，但链接器可以用二分 O(log n)
2. **节省空间**——多个符号可能共享字符串后缀，且不用每个条目预留长名字字段
3. **字符串表可以独立去重优化**——链接器可以合并相同的字符串引用

### 2.3 st_info：一个字节里的双重信息

```
st_info (1 字节)
┌─────────────────────┬─────────────────────┐
│  高 4 位：STB_xxx   │  低 4 位：STT_xxx   │
│  (BIND / 绑定)       │  (TYPE / 类型)       │
└─────────────────────┴─────────────────────┘
```

**BIND（高 4 位）：这个符号"归谁"**

| 值 | 宏 | 含义 |
|:---:|------|------|
| 0 | `STB_LOCAL` | Local，**对外不可见**，链接器不导出。`static` 函数/变量 |
| 1 | `STB_GLOBAL` | Global，**对外可见**。其他 .o 和 .so 都能引用 |
| 2 | `STB_WEAK` | Weak，**对外可见但弱**。可以被同名的 Global 覆盖 |

**TYPE（低 4 位）：这个符号"是什么"**

| 值 | 宏 | 含义 |
|:---:|------|------|
| 0 | `STT_NOTYPE` | 未指定类型，nm 显示为 U |
| 1 | `STT_OBJECT` | 数据对象（变量） |
| 2 | `STT_FUNC` | 函数 |
| 3 | `STT_SECTION` | Section 符号（内部用） |
| 4 | `STT_FILE` | 源文件名符号（调试用） |
| 5 | `STT_COMMON` | COMMON 块（未初始化的全局变量，链接前） |
| 6 | `STT_TLS` | 线程局部存储变量 |

---

## 三、工具实战：readelf 和 nm

### 3.1 readelf：看到原始结构

```bash
# 看所有符号（最全）
readelf -s /bin/ls

# 动态符号表（运行时需要的）
readelf --dyn-syms /bin/ls

# 只看 .dynsym 的原始数据
readelf -x .dynsym /bin/ls
```

readelf 的输出格式：

```
Num:    Value          Size Type    Bind   Vis      Ndx Name
 37: 0000000000003e80   112 FUNC    GLOBAL DEFAULT  15 main
 45: 0000000000003d70    51 FUNC    LOCAL  DEFAULT  15 usage
 52: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
```

**readelf 的优势：不依赖 libbfd，直接解析 ELF，裸机交叉编译场景下也能用。且它按原始字段拆开显示 BIND 和 TYPE，不会像 nm 那样用单字母混在一起。**

### 3.2 nm：简洁的符号清单

```bash
nm /bin/ls                    # 查看所有符号
nm -D /bin/ls                 # 只看动态符号表（等同于 readelf --dyn-syms）
nm -u /bin/ls                 # 只看未定义符号（外部依赖）
nm -g /bin/ls                 # 只看全局符号
nm -n /bin/ls                 # 按地址排序
nm --defined-only /bin/ls     # 只看已定义的（自己提供的）
nm -C /bin/ls                 # demangle C++ 符号名
```

nm 输出格式：**`地址 类型字母 符号名`**

```
0000000000003e80 T main
0000000000003d70 t usage
                 U printf
0000000000012600 D optarg
0000000000012620 B stdout
0000000000003e40 W __cxa_finalize
```

**nm 用单字母同时编码了 BIND + TYPE + Section 归属**。理解这些字母是理解符号表的关键。

---

## 四、符号类型字母全解——这才是你要的核心

nm 用一个字母表达了 readelf 需要三列才能说明的信息。这个字母的含义来自三条轴的交集：**BIND（local/global/weak）× TYPE（func/object/notype）× Section 归属**。

### 4.1 大写 = Global / Weak，小写 = Local

这是最重要的记忆规律：

```
大写字母 → 对外可见
小写字母 → 本文件内部可见
```

### 4.2 完整对照表

| nm | BIND | TYPE | Section | 含义 | 例 |
|:---:|------|------|---------|------|-----|
| **T** | GLOBAL | FUNC | .text | 全局函数，在代码段 | `main`, `printf`(如果本文件定义) |
| **t** | LOCAL | FUNC | .text | 静态函数，只在当前文件可见 | `static void helper()` |
| **D** | GLOBAL | OBJECT | .data | 已初始化的全局变量 | `int x = 5;` |
| **d** | LOCAL | OBJECT | .data | 已初始化的静态变量 | `static int y = 5;` |
| **B** | GLOBAL | OBJECT | .bss | 未初始化的全局变量 | `int z;`（程序启动时清零） |
| **b** | LOCAL | OBJECT | .bss | 未初始化的静态变量 | `static int w;` |
| **R** | GLOBAL | OBJECT | .rodata | 只读全局数据 | `const char* msg = "hello";` |
| **r** | LOCAL | OBJECT | .rodata | 只读静态数据 | `static const int N = 100;` |
| **U** | GLOBAL | NOTYPE | UND | 未定义，需要从外部导入 | `printf`（来自 libc） |
| **W** | WEAK | FUNC | .text | 弱函数，可被同名 Global 覆盖 | `__cxa_finalize` |
| **w** | WEAK | FUNC | .text | 弱函数，Local 版（少见） | |
| **V** | WEAK | OBJECT | .data | 弱对象（变量），可被覆盖 | 头文件中的 `inline` 变量 |
| **v** | WEAK | OBJECT | .data | 弱对象，Local 版（少见） | |
| **C** | GLOBAL | COMMON | COMMON | 公共块，链接前未决定位置的全局变量 | C 中的 `int x;`（无 extern 无初始化） |
| **A** | GLOBAL | NOTYPE | ABS | 绝对地址符号，不被重定位 | 链接器脚本中的符号 |
| **N** | — | — | — | 调试符号 | 被 `-g` 编译产生的 debug info |
| **i** | LOCAL | NOTYPE | — | 间接引用（COFF/PE，ELF 少见） | |
| **?** | — | — | — | 未知类型 | 文件格式损坏 |


### 4.3 三个最容易混淆的对比

#### T vs t

```c
// a.c
void global_func() { }       // → T (Global Func)
static void local_func() { } // → t (Local Func)
```

`static` 直接改变了符号的绑定属性。链接时 `local_func` 对其他 .o 不可见。

#### D vs B vs C

```c
int initialized = 42;   // → D (Data, .data section, 文件中有初始值)
int uninitialized;      // → B (BSS, .bss section, 文件不占空间, 启动清零)

// 注意：这个区别在最终可执行文件里可能不容易看出来
// 因为在 .o 文件中是 C (COMMON)，链接后才变成 B

// 在 .o 阶段：
int tentative;          // → C (COMMON, 未指定是否 extern, 未初始化)
                        //    意义：多个 .o 可以各自声明 int tentative;
                        //    链接时合并为同一个，取最大的那个
```

**为什么 C 存在？** 这是 C 语言的"试探性定义"（tentative definition）——C 允许你在不同 .c 文件中多次声明同一个全局变量而不报错。编译器不知道最终用的是谁，所以标记为 COMMON，让链接器来做最终决议。C++ 没有这个问题（ODR 规则），所以 C++ 编译器通常直接标记为 B。

#### B vs b

```c
int global_zero;            // → B (所有文件都能访问)
static int static_zero;     // → b (只有当前文件能访问)
```

B 和 b 的内容完全一样（零值/未初始化），唯一区别是可见性。所以 BSS 段内是按"全局 vs 静态"分区的。

---

## 五、符号可见性：st_other 字段

在 STB_LOCAL / GLOBAL / WEAK 之外，还有一层**可见性**控制（st_other 的低 2 位）：

| 值 | 宏 | 含义 |
|:---:|------|------|
| 0 | `STV_DEFAULT` | 默认，正常导出 |
| 1 | `STV_INTERNAL` | 内部，即使符号是 GLOBAL，链接器也不允许外部引用 |
| 2 | `STV_HIDDEN` | 隐藏，动态链接器不会导出给其他 .so |
| 3 | `STV_PROTECTED` | 保护，可被外部引用但不可被覆盖（类似 final） |

**这个和控制符号可见性的编译器属性对应：**

```c
__attribute__((visibility("default")))   void fn_a();
__attribute__((visibility("hidden")))    void fn_b();  // 只在当前 .so 内可见
__attribute__((visibility("internal")))  void fn_c();
__attribute__((visibility("protected"))) void fn_d();
```

编译时用 `-fvisibility=hidden` 可以把默认可见性设为 hidden，然后手动标记需要导出的符号。这是减少 .so 的 ABI 暴露面的标准做法。

**用 readelf 可以直接看到：**

```
Symbol table '.dynsym' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     1: 0000000000001129    23 FUNC    GLOBAL DEFAULT  12 export_fn
     2: 0000000000001140    30 FUNC    GLOBAL HIDDEN   12 internal_fn
     3: 000000000000115e    15 FUNC    GLOBAL PROTECTED 12 protected_fn
```

---

## 六、深入本质：符号表设计中反映的计算机科学问题

### 6.1 为什么要有两种符号表（.symtab vs .dynsym）？

这是从"编译时"到"运行时"的信息折叠过程：

```
.symtab (完整版)
├── 所有函数名，包括 static 的
├── 所有变量名，包括局部的
├── 调试用的源文件名（STT_FILE）
├── Section 符号（STT_SECTION）
└── 链接器的内部工作符号

    ↓ strip / 编译优化去掉 .symtab

.dynsym (精简版)
├── 只有动态链接需要的符号
├── 导出的 API（别的 .so 会调用的）
├── 需要导入的符号（从别的 .so 来的）
├── 不包含 local 符号
├── 不包含 .symtab 才有的调试/内部信息
└── 比 .symtab 小很多倍
```

**本质：** `.symtab` = 编译器给链接器的全部工作材料；`.dynsym` = 动态链接器运行时需要的最小信息集。从 `.symtab` 到 `.dynsym` 是一种信息压缩——保留运行时必要的，去掉编译时中间状态的。

这就是为什么 `strip` 删的是 `.symtab` 而不是 `.dynsym`。动态链接器必须能工作，但调试器可以不需要。

### 6.2 为什么弱符号（WEAK）存在？

弱符号解决了一个很实际的问题：**库需要提供默认实现，但允许用户覆盖。**

```c
// libc 中（弱定义）
__attribute__((weak))
void *malloc(size_t size) {
    return __libc_malloc(size);  // 默认实现
}

// 用户代码（强定义，覆盖弱定义）
void *malloc(size_t size) {
    return my_custom_allocator_malloc(size);
}
```

链接规则：
1. 多个强定义同名符号 → **链接错误**（multiple definition）
2. 一个强定义 + 多个弱定义 → **选强定义**
3. 全是弱定义 → **选任意一个**（通常是先遇到的）

这就是 `__cxa_finalize`、`pthread_create` 等函数被标记为 W 的原因——允许用户或框架提供替代实现。

**WEAK 的底层本质：它是链接器中符号决议优先级系统的一部分，让你可以在不修改库源码的情况下"偷换"库的实现——C 语言没有 override 关键字，弱符号填补了这个空白。**

### 6.3 COMMON 的历史债务

COMMON (nm 中的 C) 是 C 语言的一个历史遗留。FORTRAN 最早引入了 COMMON 块的概念——多个编译单元共享一个未初始化的数据块。C 继承了它：

```c
// file1.c
int global_x;       // tentative definition → COMMON

// file2.c
int global_x;       // 又是 tentative definition → COMMON
// 链接时合并为同一个 global_x

// file3.c
int global_x = 42;  // 这是正式定义 → 变成 D
// 链接器：有正式定义就用正式的，COMMON 的全合并过去
```

**本质：** COMMON 是编译单元边界的一种妥协——编译器无法独立判断未初始化全局变量是否在其他 .o 中被"正式定义"，所以交给链接器做最终裁决。C++ 因为 ODR（单一定义规则）直接禁止了这种行为，这就是为什么 C++ 代码里 `int x;` 在 .o 中通常是 B 而不是 C。

### 6.4 符号解析的本质：名字怎么变成地址

当你在代码中写 `printf("hello")` 时：

```
阶段 1：编译（生成 .o）
  call printf@PLT   ← 生成一个未解析的调用点 + 重定位条目
  printf → 在 .o 的 .symtab 中标记为 UND（U）

阶段 2：静态链接（生成可执行文件）
  链接器搜索 libc.so 的 .dynsym，找到 printf 的定义
  → 填充 GOT/PLT 表项
  → 但此时还不知道 printf 的最终地址（因为 libc.so 被加载到哪不确定）

阶段 3：动态链接（运行时）
  ld.so 加载 libc.so 到内存
  → 找到 libc.so 中 printf 的实际地址
  → 填充 GOT 表项 = 该地址
  → 首次调用时 PLT stubs 跳转到 GOT 指向的真实地址

阶段 4（如果用了延迟绑定/lazy binding）：
  PLT 第一次被调用时触发动态链接器解析
  → 之后 PLT 直接跳转到已解析的地址
  → 不开 lazy binding (LD_BIND_NOW=1) 则在启动时全部解析
```

**从 UND 到地址的整个过程，就是符号表的存在意义。**

---

## 七、实战技巧

### 7.1 快速排查未定义符号

```bash
# 看一个 .so 缺什么依赖
readelf --dyn-syms libfoo.so | grep UND

# 或者更直接
nm -u libfoo.so

# 看看系统能不能找到所有依赖（模拟动态链接）
ldd -r libfoo.so     # -r 会显示 unresolved symbols
```

### 7.2 检查符号导出是否正确

```bash
# 看 .so 对外暴露了什么（不应该暴露的内部函数是不是被标记了？）
nm -gD libfoo.so | grep ' T '

# 验证用了 -fvisibility=hidden 编译
# 正确的：只有明确标记的函数才是 T，内部函数都应该是 t
readelf --dyn-syms libfoo.so | grep FUNC
```

### 7.3 查 C++ 符号名混乱（mangling）

```bash
# C++ 编译后符号名变成了 _Z3fooi，用 -C 解混淆
nm -C libfoo.a | grep foo
# _Z3fooi  →  foo(int)

# 找不到符号时——检查是不是 C 和 C++ 混编问题
# C 函数在 C++ 中调用需要 extern "C" {}，否则符号名对不上
```

### 7.4 剥离调试符号

```bash
# 分成两个文件
objcopy --only-keep-debug a.out a.out.debug
objcopy --strip-debug a.out                # 去掉调试信息
objcopy --add-gnu-debuglink=a.out.debug a.out  # 建立关联

# 完全剥离（连 .symtab 都不要）
strip a.out
# 这时 nm a.out 只能看到 .dynsym 的内容
```

---

## 八、总结：一条统一的认知线

从 ELF 文件的角度，符号表是**连接两个世界的翻译器**：

```
          源代码世界                       二进制世界
    ┌──────────────────┐         ┌──────────────────────┐
    │ "调用 printf"     │ ──────→ │ call 0x...  (PLT)    │
    │ "变量 x 在 .data"  │ ──────→ │ mov rax, [rip+0x...] │
    │ "这个函数我写的"   │ ──────→ │ T (Global Func)      │
    │ "这个函数别人的"   │ ──────→ │ U (Undefined)        │
    └──────────────────┘         └──────────────────────┘
              │                            │
              └──────── 符号表 ─────────────┘
                   nm, readelf -s
```

而 `nm` 和 `readelf` 的每个输出行，本质上是在回答四个问题：

1. **这个符号在哪？** → st_value（地址）
2. **它对谁可见？** → BIND：LOCAL / GLOBAL / WEAK（大小写）
3. **它是什么类型？** → TYPE：FUNC / OBJECT / NOTYPE（字母本身）
4. **它属于哪个 Section？** → st_shndx（字母和 Section 的对应：T→.text, D→.data, B→.bss）

掌握了这四层信息，任何符号表输出你都能一眼看穿。

## 相关概念

[[proc-kallsyms与BTF-vmlinux]] · [[pkg-config]] · [[BTF (BPF Type Format) 详解]]

## 参考

- 对话讨论 (2026-07-21)
- ELF 规范：System V ABI, Chapter 4 (Object Files)
