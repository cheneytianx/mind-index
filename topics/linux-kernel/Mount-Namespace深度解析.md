---
tags: [linux, kernel, namespace, container, filesystem]
created: 2026-07-27
source: conversation
aliases: [Mount Namespace, mount namespace]
---

# Mount Namespace 深度解析

## 一、本质：Mount Namespace 是什么

**Mount Namespace 是内核为每个 namespace 维护的一份独立的 mount tree（挂载点拓扑），本质上是一个 `struct mnt_namespace` 指向的私有 `struct mount` 链表。**

### 内核数据结构

```c
struct mnt_namespace {
    atomic_t count;           // 引用计数
    struct mount *root;       // 根挂载点
    struct list_head list;    // 该 namespace 下所有 mount 的双向链表
    struct user_namespace *user_ns;
    u64 seq;
};
```

每个挂载点对应一个 `struct mount`，核心字段：

```c
struct mount {
    struct mount *mnt_parent;       // 父挂载点（namespace 内的拓扑）
    struct dentry *mnt_mountpoint;  // 挂载目标目录
    struct vfsmount mnt;            // 指向根 dentry
    struct mnt_namespace *mnt_ns;   // 所属 namespace
};
```

底层文件系统实例 `struct super_block` 则是**全局共享**的：

```c
struct super_block {
    struct file_system_type *s_type;
    struct super_operations *s_op;
    struct dentry *s_root;
};
```

### 关键架构

```
Mount Namespace（per-ns）          文件系统（全局共享）
┌──────────────────────┐          ┌──────────────────────┐
│  struct mount 链表    │          │  struct super_block   │
│  ├── / (mount #1)    │ ──────→  │  ├── inode 表         │
│  ├── /proc (mount #2)│          │  ├── dentry cache     │
│  ├── /sys (mount #3) │          │  └── 文件数据         │
│  └── /tmp (mount #4) │          └──────────────────────┘
└──────────────────────┘
```

---

## 二、创建新 Mount Namespace 后进程表现的变化

### 2.1 创建过程

```c
clone(CLONE_NEWNS)  // 或 unshare(CLONE_NEWNS)
    ↓
子进程获得一份父 mount tree 的副本
    ↓
副本中所有挂载点被标记为 MS_PRIVATE（防止挂载事件泄漏回父 ns）
    ↓
两个 mount tree 此后独立演化
```

### 2.2 前后差异

| 维度 | 共享 namespace | 新 Mount Namespace |
|------|---------------|---------------------|
| `mountinfo` 内容 | 与其他进程相同 | 初始相同，之后独立 |
| `mount` 操作的影响 | 全局可见 | 仅 namespace 内可见 |
| `umount` 操作的影响 | 影响全局 | 仅 namespace 内生效 |
| `rm` / `touch` / `write` | — | **不隔离**（见下文实验分析） |

### 2.3 Mount Propagation（挂载传播）

创建新 namespace 时，挂载点的传播类型自动转换：

| 传播类型 | 含义 |
|---------|------|
| `MS_SHARED` | 挂载事件双向传播 |
| `MS_PRIVATE` | 完全隔离，不传播（新 ns 默认） |
| `MS_SLAVE` | 单向接收来自 master 的事件 |
| `MS_UNBINDABLE` | 类似 private 但禁止 bind mount |

```
父 Namespace                   子 Namespace (新创建)
┌───────────┐                 ┌───────────┐
│ / (MS_SHARED)               │ / (MS_PRIVATE)  ← 自动转 private
│ ├── /proc                   │ ├── /proc
│ ├── /sys                    │ ├── /sys
│ └── /data                   │ └── /data
│                             │
│ mount 新设备 →               │ ← 看不到
│                             │   独立演化 →
└───────────┘                 └───────────┘
```

---

## 三、实验分析：为什么 rm 会在两个 ns 同步生效

### 3.1 实验现象

- 进程进入新 Mount Namespace（`/proc/$$/ns/mnt` 指向不同的 ns ID）
- 在新 ns 下 `rm /tmp/xxx`
- 旧 ns 下文件也消失了

### 3.2 原因：Mount Namespace 隔离的边界

**Mount Namespace 隔离的是"挂载关系"（mount table），不是"文件系统数据"。**

```
        父 NS                           子 NS
   mount tree                         mount tree（独立副本）
          │                                  │
          │ / 的 mount 实例                   │ / 的 mount 实例
          │   └── /tmp                        │   └── /tmp
          │         │                         │         │
          └─────────┼─────────────────────────┘
                    │
                    ▼
          同一个 ext4/xfs super_block
          同一个 inode 表
          同一个 directory entry
          
          rm /tmp/xxx → unlink(inode) → 两边同时看到数据变化
```

### 3.3 两类操作的对比

| 操作 | 作用层面 | namespace 间可见性 |
|------|---------|-------------------|
| `mount` | `struct mount` 链表（per-ns）| **隔离** |
| `umount` | `struct mount` 链表（per-ns）| **隔离** |
| `rm` | `super_block` → inode/dentry（全局共享）| **不隔离** |
| `touch` | `super_block` → inode/xattr（全局共享）| **不隔离** |
| `write` | `super_block` → 数据块（全局共享）| **不隔离** |

**一句话**：`rm` 操作穿过 mount 层直接到达文件系统层，而文件系统层是所有 namespace 共享的。Mount Namespace 只改"路径怎么走到文件系统"这个映射关系，不改文件系统里的数据。

---

## 四、"视图"模型

### 4.1 类比：导航路线图

```
Mount Namespace = 导航路线图（决定"哪条路径通往哪个文件系统"）
底层文件系统    = 实际地貌（super_block + inode + 数据块）
```

- **`mount` / `umount`** → 在地图上改路标，"这条路不通了"
- **`rm` / `touch` / `write`** → 直接在地上挖坑，"不管地图怎么画，坑已经挖了"

### 4.2 同一路径指向不同数据的条件

```
       父 NS 视图                        子 NS 视图
   /tmp → ext4 某个区域              /tmp → tmpfs（新挂载的独立内存文件系统）
                │                              │
                ▼                              ▼
          同一个磁盘                      完全独立的 tmpfs
```

只有当新 ns 对 `/tmp` 重新 mount（比如 mount 一个 tmpfs），同一路径才指向不同的底层数据，此时 `rm` 才真正隔离。

### 4.3 结论

> **视图可以变，底层数据不一定独立——取决于你在这个视图上 mount 了什么。**
>
> **单靠 Mount Namespace 不能提供文件系统隔离。**

---

## 五、chroot：换个"根"

### 5.1 内核机制

每个进程的 `task_struct` 里有一个 `fs_struct`：

```c
struct fs_struct {
    struct path root;   // 进程眼中的 "/"
    struct path pwd;    // 当前工作目录
};
```

**chroot 就是修改 `root` 指针，指向另一个目录的 dentry。**

```
chroot 前                        chroot /newroot 后

进程 root → /                    进程 root → /newroot（宿主机视角）
              │                                  │
   宿主机真实的根                    /newroot 的子目录树变成新"世界"
```

路径解析的起始锚点变了——内核解析路径时从 `fs_struct.root` 开始，而不是全局根。

### 5.2 致命缺陷：可以逃逸

```c
// 经典 chroot 逃逸手法
mkdir("foo");
chroot("foo");
// root → 宿主机 /newroot/foo/
// cwd 还没变，仍然在某个旧路径

chdir("../../../../../");  // cwd 沿旧路径一直走到宿主机真实 /
chroot(".");               // root 设回宿主机 /
// 逃逸成功
```

**chroot 从一开始就不是安全边界**，它只是一个"建议你待在这个区"的导航限制，有 root 权限就可以绕过。

### 5.3 其他限制

- `/proc` 不隔离 → 能看到宿主机所有进程
- 网络栈共享
- 设备文件暴露 → `/dev/sda` 可访问宿主机磁盘

---

## 六、pivot_root：容器真正的根切换

`pivot_root` 是 Linux 2.6 引入的系统调用，专门为容器设计。

### 6.1 与 chroot 的对比

| | chroot | pivot_root |
|---|---|---|
| 改变方式 | 只改 `root` 指针（进程级） | 整个 mount tree 的根**物理替换**（namespace 级） |
| 旧根可达性 | 仍可通过 cwd/fd 访问 | 被移到子目录后 umount，物理不可达 |
| 逃逸难度 | 低（root 权限即可） | 高（旧根已卸载） |
| 作用范围 | `fs_struct`（单进程） | `mnt_namespace.root`（整个 namespace） |

### 6.2 操作逻辑

```
pivot_root(new_root, put_old);
// 1. new_root 变成当前 mount namespace 的根
// 2. 旧根（host /）整体移入 put_old 下
// 3. chdir 到新根，umount put_old
//    → 旧根连路径都消失了
```

---

## 七、完整拼图：Docker 如何实现文件系统隔离

### 7.1 三步组合

```
1. clone(CLONE_NEWNS)     ← 隔离挂载点拓扑（自己的 mount 表）
2. mount overlayfs        ← 写时复制（改动不进底层数据）
      lowerdir = 镜像层（只读）
      upperdir = 容器可写层
      merged   = 合并视图
3. pivot_root(merged_dir) ← 把 overlayfs 合并视图设为新 "/"
   旧根挪走并卸载          ← 让宿主机根文件系统物理不可达
```

### 7.2 每个组件负责什么

```
Mount Namespace → 我有自己的"导航路线图"（挂载表私有）
pivot_root      → 我换了一张"世界地图"（根目录被整体替换）
overlayfs       → 我在地图上涂画不影响别人的地图（写时复制隔离数据）
```

### 7.3 回到 rm 实验

有了这三层后：

```
容器内 rm /tmp/xxx
  ↓
作用于 overlayfs 的 merged view
  ↓
写入 upper layer → 标记 whiteout（"已删除"）
  ↓
lower layer（宿主机原始文件）→ 完好无损
```

| 视角 | /tmp/xxx 状态 |
|------|-------------|
| 容器内（overlay merged） | 不存在（被 whiteout 遮住） |
| 宿主机（ext4 原始路径） | 存在，完整 |

---

## 八、总结

> **Mount Namespace → 独立挂载表（视图，隔离 mount 操作，不隔离数据）**
> **chroot / pivot_root → 重新定义根（pivot_root 是真正的不可逆替换）**
> **overlayfs → 写时复制（隔离数据修改）**
>
> 三者组合，才是 Docker 文件系统隔离的完整方案。

### 相关笔记

- [[容器网络-bridge模式数据包路径]]：网络层面的隔离
- [[seccomp与Linux安全机制全景]]：系统调用级安全限制
- [[openat与dirfd-从fs-struct看chdir的进程级影响]]：fs_struct 与路径解析
