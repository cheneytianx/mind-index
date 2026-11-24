---
tags: [linux, build-tools, c, concept]
created: 2026-07-21
source: conversation
---

# pkg-config：编译时的库元数据查询工具

> 一句话：pkg-config 自动查询库的编译和链接参数，把你从手动记住 `-I` / `-l` 的苦役中解放出来。

## 它解决什么问题

在 C 项目中使用第三方库，需要告诉编译器两件事：

```bash
# 手动方式——每个库路径不同、不同发行版安装位置不同
gcc main.c -o app \
  -I/usr/include/glib-2.0 \
  -I/usr/lib/x86_64-linux-gnu/glib-2.0/include \
  -lglib-2.0
```

pkg-config 把它自动化：

```bash
gcc main.c -o app $(pkg-config --cflags --libs glib-2.0)
# 自动展开成所有需要的 -I 和 -l
```

## 工作原理

### 核心：.pc 文件

每个库安装时附带一个 `.pc` 文件（纯文本，键值对格式），放在约定目录：

```
/usr/lib/pkgconfig/          ← 系统库（apt/yum 安装的 -dev 包）
/usr/share/pkgconfig/        ← 架构无关的库
/usr/local/lib/pkgconfig/    ← 用户自己编译安装的
```

以 `openssl.pc` 为例：

```ini
prefix=/usr
exec_prefix=${prefix}
libdir=${exec_prefix}/lib/x86_64-linux-gnu
includedir=${prefix}/include

Name: OpenSSL
Description: Secure Sockets Layer and cryptography libraries
Version: 3.0.2
Requires.private: libcrypto
Libs: -L${libdir} -lssl
Libs.private: -ldl -lpthread
Cflags: -I${includedir}
```

| 字段 | 含义 |
|------|------|
| `Name` / `Version` | 库名和版本，支持 `pkg-config --exists` 存在性检测 |
| `Requires` | 使用本库时**必须**公开的依赖（会传递给调用者） |
| `Requires.private` | 仅静态链接时需要的私有依赖 |
| `Libs` | 链接选项：`-L<路径> -l<库名>` |
| `Libs.private` | 仅静态链接追加的额外链接选项 |
| `Cflags` | 编译选项：`-I<头文件路径>` |

### 查询流程

```
pkg-config --cflags --libs openssl
         │
         ▼
  1. 遍历 $PKG_CONFIG_PATH 和默认路径
         │
         ▼
  2. 找到 openssl.pc，解析为键值对
         │
         ▼
  3. 展开变量引用：${prefix} → /usr
         │
         ▼
  4. 检查 Requires：递归展开所有依赖库
         │
         ▼
  5. 合并输出 -I 和 -l，去重，按依赖顺序排列
```

### 常用命令

```bash
pkg-config --cflags openssl        # 只看编译选项
pkg-config --libs openssl          # 只看链接选项
pkg-config --modversion openssl    # 库版本
pkg-config --exists openssl        # 检测库是否存在（返回码判断）
pkg-config --static --libs openssl # 静态链接（含 Libs.private）
pkg-config --list-all              # 列出系统中所有已知库
pkg-config --variable=prefix openssl  # 查 .pc 中任意变量
```

## 核心价值：依赖的递归解析

这是它相比手动 `-l` 最大的优势。

假设使用 `gtk+-3.0`，其 `.pc` 声明：

```
Requires: gdk-3.0 atk cairo cairo-gobject gdk-pixbuf-2.0 gio-2.0
```

每个又各自有依赖。**整棵依赖树可能几十个库。** 你只需：

```bash
pkg-config --cflags --libs gtk+-3.0
```

pkg-config 自动递归展开，帮你处理所有 `-I`、`-l`，去重，按正确顺序排列（被依赖的放后面）。

## 与构建系统的集成

Linux 上几乎所有构建系统都原生支持或提供封装：

```makefile
# 原生 Makefile
CFLAGS += $(shell pkg-config --cflags glib-2.0)
LDLIBS += $(shell pkg-config --libs glib-2.0)
```

```cmake
# CMake
find_package(PkgConfig REQUIRED)
pkg_check_modules(GLIB REQUIRED glib-2.0)
target_include_directories(app PRIVATE ${GLIB_INCLUDE_DIRS})
target_link_libraries(app PRIVATE ${GLIB_LIBRARIES})
```

```meson
# Meson
glib_dep = dependency('glib-2.0')
executable('app', 'main.c', dependencies: glib_dep)
```

Autotools 提供 `PKG_CHECK_MODULES` 宏，是自动工具链的标准做法。

## 交叉编译

交叉编译时不能用宿主机的 `.pc`（指向宿主 `/usr/lib`），需用目标 sysroot 的：

```bash
PKG_CONFIG_PATH=/path/to/sysroot/usr/lib/pkgconfig \
PKG_CONFIG_SYSROOT_DIR=/path/to/sysroot \
pkg-config --cflags --libs zlib
```

- `PKG_CONFIG_PATH`：指向 sysroot 中的 .pc
- `PKG_CONFIG_SYSROOT_DIR`：自动把 .pc 里的 `/usr/lib` 前缀替换为 sysroot 路径

这是嵌入式 Linux 构建系统（Buildroot、Yocto）深度依赖的能力——工具链包装器自动设置这些变量。

## 与 CMake Config 的竞争关系

CMake 查找库的优先级：

```
find_package(XXX CONFIG)  → 找 XXXConfig.cmake（CMake 原生）
  找不到 ↓
find_package(XXX MODULE)  → 用 FindXXX.cmake 规则
  找不到 ↓
pkg_check_modules(XXX ...) → 回退到 pkg-config
```

趋势是 CMake 的 Config 文件越来越主流，但大量 C 库（非 C++）和 Linux 系统库仍以 `.pc` 为准。短期内不会消失。

## 总结

pkg-config 做的事极简：**把每个库的编译链接参数存成纯文本，提供统一的查询接口。** 设计哲学是经典 Unix 风格——做一件事做好，纯文本格式，可脚本化。

你不一定直接调用它，但你的构建系统几乎必然在底层调用了它。

## 相关概念

[[CMake]] · [[交叉编译]]

## 参考

- 对话讨论 (2026-07-21)
