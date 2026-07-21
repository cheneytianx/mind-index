---
tags: [linux, kernel, syscall, filesystem, concurrency]
created: 2026-07-21
source: conversation
---

# open 与 openat：dirfd 的底层原理

> `*at` 系统调用的本质，是把路径解析的起点从进程级共享状态（`cwd`）转变为可显式传递的稳定句柄（dirfd）。这是从"隐式上下文驱动"到"显式参数驱动"的范式转换。

## 一、open vs openat：路径解析从哪开始

```c
int open(const char *pathname, int flags, mode_t mode);
int openat(int dirfd, const char *pathname, int flags, mode_t mode);
```

| 维度 | `open` | `openat` |
|------|--------|----------|
| 路径解析起点 | `current->fs->pwd`（cwd） | `dirfd` 指向的目录 inode |
| 相对路径含义 | 相对于 cwd | 相对于 dirfd |
| 绝对路径 | 从 `/` 开始 | 从 `/` 开始 |
| 并发安全性 | ❌ cwd 是进程全局的，多线程互相踩 | ✅ dirfd 是稳定的 inode 快照，不受 chdir 影响 |

`AT_FDCWD`（= -100）使 `openat` 退化为 `open`——从 cwd 解析。这个设计让库函数内部可以统一用 `*at`，对外暴露 `AT_FDCWD` 作为默认。

## 二、chdir 为什么影响整个进程——从 task_struct 拆解

### 2.1 Linux 线程的本质

Linux 没有独立的"线程"概念。线程就是一个和父线程共享了某些资源的 `task_struct`：

```c
// 创建线程（pthread_create）：
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD, ...)
//     ↑          ↑           ↑              ↑
//   共享地址空间  共享fs    共享文件表    共享信号处理
```

**`CLONE_FS` 是根源——它让所有线程共享同一个 `fs_struct`。**

### 2.2 fs_struct：文件系统上下文的容器

```c
// include/linux/fs_struct.h
struct fs_struct {
    int users;           // 引用计数
    spinlock_t lock;
    int umask;
    struct path root;    // chroot 设置的根目录
    struct path pwd;     // ★ 当前工作目录——chdir 改的就是这个
} __randomize_layout;
```

**`pwd`（present working directory）是 `fs_struct` 的一个字段。所有共享这个 `fs_struct` 的线程，读写的都是同一块内存。**

```
主线程 task_struct           子线程 task_struct
┌──────────────────┐      ┌──────────────────┐
│ pid=1000          │      │ pid=1001          │
│ tgid=1000         │      │ tgid=1000         │  ← 同一个 tgid
│ fs ───────────────┼──┐   │ fs ───────────────┼──┐
└──────────────────┘  │   └──────────────────┘  │
                      │                         │
                      ▼                         ▼
              ┌───────────────────────────────────┐
              │         同一个 fs_struct            │
              │  pwd = /home/user                  │
              │  root = /                          │
              │  umask = 022                       │
              │  users = 2   ← 引用计数             │
              └───────────────────────────────────┘
```

### 2.3 chdir 内核路径

```c
// fs/open.c (简化)
SYSCALL_DEFINE1(chdir, const char __user *, pathname)
{
    struct path path;
    error = user_path_at(AT_FDCWD, pathname,
                         LOOKUP_FOLLOW | LOOKUP_DIRECTORY, &path);
    // ↑ 解析路径名拿到的目标目录

    struct fs_struct *fs = current->fs;  // ★ current=当前线程的 task_struct

    spin_lock(&fs->lock);
    struct path old_pwd = fs->pwd;
    fs->pwd = path;       // ★ 改的是共享的 fs_struct！
    spin_unlock(&fs->lock);

    path_put(&old_pwd);
    return 0;
}
```

**关键：** 虽然 `current` 是"当前线程自己的 task_struct"，但 `current->fs` 指向的 `fs_struct` 是所有线程共享的同一个。改的是共享对象。

```
线程 A 调 chdir("/tmp")：
  current->fs->pwd = /tmp   ← 修改共享对象

线程 B 随后调 open("data.txt")：
  路径解析从 current->fs->pwd = /tmp 开始  ← 读到了被改过的值
```

## 三、openat 为什么免疫——不经过 fs->pwd

```c
SYSCALL_DEFINE4(openat, int, dfd, const char __user *, filename,
                int, flags, umode_t, mode)
{
    // dfd → current->files[dfd] → struct file * → f_path.dentry → inode
    // 路径解析从 dirfd 指向的 inode 开始，全程不碰 current->fs->pwd
}
```

```
openat(dirfd, "data.txt") 的路径解析：
  1. dirfd → fdget(dfd) → struct file * → inode（稳定，不变）
  2. 从 inode 开始解析 "data.txt"
  3. 全程不碰 current->fs->pwd
```

### 为什么 dirfd 比 cwd 安全

```
fs_struct->pwd    被 chdir 隐式修改，任何线程都能改，下次路径解析当场受影响
                  修改方不一定是"你"

files_struct->fd[] 只被显式操作（open/close/dup）修改
                   dirfd 是 open(dir) 时拿到的快照——inode 不变
                   只要不 close，永远指向同一个目录
```

**dirfd 是 immutable reference to a stable object。`fs_struct->pwd` 是 mutable global state。**

## 四、为什么 CLONE_FS 这样设计

POSIX 线程标准要求同一进程内的线程共享工作目录。Linux 通过 `CLONE_FS` 实现这个语义。

```c
// fork——不共享
clone(SIGCHLD)
// → 子进程复制一份独立的 fs_struct（pwd 独立）

// pthread_create——共享
clone(CLONE_FS | CLONE_VM | ...)
// → 所有线程共享同一个 fs_struct
// → chdir 影响全部线程，这是 POSIX 要求的
```

## 五、完整共享结构图

```
Process (tgid=1000)
═══════════════════════════════════════════════════════════

  线程 A (pid=1000)               线程 B (pid=1001)
  ┌─────────────────────┐       ┌─────────────────────┐
  │ task_struct         │       │ task_struct         │
  │  pid = 1000         │       │  pid = 1001         │
  │  tgid = 1000        │       │  tgid = 1000        │
  │                     │       │                     │
  │  fs ────────────────┼───┐   │  fs ────────────────┼───┐
  │  files ─────────────┼───┤   │  files ─────────────┼───┤
  │  mm ────────────────┼───┤   │  mm ────────────────┼───┤
  └─────────────────────┘   │   └─────────────────────┘   │
                            │                             │
          ┌─────────────────┘         ┌───────────────────┘
          ▼                           ▼
  ┌──────────────────────┐  ┌──────────────────────────────┐
  │     fs_struct         │  │       files_struct           │
  │  pwd = /home/user     │  │  fd[0] → stdin               │
  │  root = /             │  │  fd[1] → stdout              │
  │  umask = 022          │  │  fd[2] → stderr              │
  └──────────────────────┘  │  fd[3] → dirfd (inode X)     │
     ▲ 共享，chdir 改这里    │  fd[4] → socket               │
                            └──────────────────────────────┘
                               ▲ 共享，close/dup2 互相可见

  chdir("/tmp") → fs_struct->pwd = /tmp → 全线程路径解析当场变了
  但 openat(fd[3], "data.txt") 不受影响：
    fd[3] → struct file → inode X（/home/user 打开时的快照）
    路径解析从 inode X 开始，无视 pwd
```

## 六、*at 系统调用族的实际场景

### 场景 1：TOCTOU 防御

```c
// ❌ check 和 use 之间可能被替换
if (access("/etc/myapp/config", R_OK) == 0) {
    int fd = open("/etc/myapp/config", O_RDONLY);  // 可能不是同一个文件
}

// ✅ 用 dirfd 锁定目录
int dirfd = open("/etc/myapp", O_RDONLY | O_DIRECTORY);
int fd = openat(dirfd, "config", O_RDONLY | O_NOFOLLOW);
// dirfd 拿到后 /etc/myapp 的 inode 锁定，O_NOFOLLOW 阻止符号链接攻击
```

### 场景 2：多线程服务器的标准范式

```c
void handle_request(int client_fd, const char *docroot) {
    int root_fd = open(docroot, O_RDONLY | O_DIRECTORY);
    // 所有文件操作相对于 root_fd，不需要 chdir，不影响其他线程
    int fd = openat(root_fd, requested_path, O_RDONLY);
}
```

### 场景 3：路径穿越防御

```c
// ❌ 危险——目录穿越
snprintf(fullpath, sizeof(fullpath), "/data/%s", user_input);
int fd = open(fullpath, O_RDONLY);  // user_input = "../../etc/shadow"

// ✅ 安全
int basedir = open("/data", O_RDONLY | O_DIRECTORY);
int fd = openat(basedir, user_input, O_RDONLY | O_NOFOLLOW);
// 内核保证路径解析不逃逸到 basedir 上级
```

### 场景 4：跨目录原子 rename

```c
int src_dir = open("/tmp/staging", O_RDONLY | O_DIRECTORY);
int dst_dir = open("/data/production", O_RDONLY | O_DIRECTORY);
renameat(src_dir, "a.txt", dst_dir, "b.txt");
// 源和目标各自相对 dirfd，renameat2 还可加 RENAME_EXCHANGE（原子交换）
```

## 七、完整 *at 族

| 传统版 | at 版 | Linux 引入 |
|--------|-------|-----------|
| `open` | `openat` | 2.6.16 |
| `stat` / `lstat` | `fstatat` | 2.6.16 |
| `chmod` | `fchmodat` | 2.6.16 |
| `chown` | `fchownat` | 2.6.16 |
| `unlink` | `unlinkat` | 2.6.16 |
| `rename` | `renameat` / `renameat2` | 2.6.16 / 3.15 |
| `mkdir` | `mkdirat` | 2.6.16 |
| `mknod` | `mknodat` | 2.6.16 |
| `link` | `linkat` | 2.6.16 |
| `symlink` | `symlinkat` | 2.6.16 |
| `readlink` | `readlinkat` | 2.6.16 |
| `utimes` | `utimensat` | 2.6.22 |
| `execve` | `execveat` | 3.19 |

全部共享同一个 `dirfd` 参数语义。

### unlinkat 的独特之处

`unlinkat` 比 `unlink` 多一个 `flags` 参数：

| flags | 含义 |
|-------|------|
| 0 | 行为同 `unlink`，删除文件 |
| `AT_REMOVEDIR` | 删除目录（行为同 `rmdir`） |

一个系统调用同时覆盖了 `unlink` 和 `rmdir`。

## 八、关键结论

**`chdir` 改的是 `fs_struct.pwd`——一个被进程中所有线程共享的字段。它改的不是 `task_struct` 的线程私有字段，而是通过 `task_struct->fs` 指针共享的 `fs_struct` 对象。一个线程调 chdir，全进程所有线程的相对路径解析都被"篡改"。**

**`openat(dirfd, ...)` 之所以免疫，是因为它不经过 `current->fs->pwd`——从 `dirfd` 对应的 inode 开始解析，而那个 inode 是 `open(dir)` 时拿到的稳定快照，不受任何后续 chdir 影响。**

**`fs_struct` 本质上是"隐式上下文"的容器——程序依赖它却不一定意识到它存在，更难意识到它是共享的。`*at` 系统调用的意义就是把隐式上下文变成显式参数。** 这和 Rust 把 `&mut` 作为显式参数传递而不是靠程序员记住"这个变量是否可变"是同一个范式转换。

## 相关概念

[[linux内核id详解]] · [[linux线程信号与进程级退出]] · [[线程单独被杀的内核机制]]

## 参考

- 对话讨论 (2026-07-21)
