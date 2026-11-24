---
tags: [cpp, concurrency]
created: 2026-06-06
source: conversation
---

# C++ 并发控制机制

> 第一部分：锁家族、原子变量与"锁 vs 原子"的选型。
> 原子变量的底层原理（硬件乱序、CAS、内存序、无锁栈）见 [[原子变量深度剖析]]。

## 一、为什么需要并发控制？

多个线程同时访问**共享数据**时，如果其中至少有一个在**写**，就会产生**[[数据竞争]]（data race）**。在 C++ 标准里，数据竞争是**[[未定义行为]]（UB）**——程序可能崩溃、可能得到错乱的值，甚至看起来"偶尔正常"。

```cpp
int counter = 0;
// 线程A和线程B同时执行
counter++;  // 危险！这不是原子操作
```

`counter++` 实际是三步：**读 → 改 → 写**。两个线程交错执行就会丢失更新。

解决思路有两类：
1. **[[互斥量|锁]]（mutex）**——让临界区"互斥"，同一时刻只有一个线程进入。
2. **[[原子操作|原子变量]]（atomic）**——让单个操作"不可分割"，无需加锁。

---

## 二、锁机制（Mutex Family）

### 2.1 基础：`std::mutex`

最基本的互斥量，提供 `lock()` / `unlock()`：

```cpp
#include <mutex>
std::mutex mtx;
int counter = 0;

void add() {
    mtx.lock();
    counter++;        // 临界区，受保护
    mtx.unlock();
}
```

**问题**：手动 `lock/unlock` 极易出错——如果临界区内抛异常或提前 `return`，`unlock()` 不会被调用，导致**[[死锁]]**。

### 2.2 RAII 封装：`lock_guard`（C++11）

利用"构造时加锁、析构时解锁"（[[RAII]]），自动管理：

```cpp
void add() {
    std::lock_guard<std::mutex> lock(mtx);  // 构造即加锁
    counter++;
}   // 离开作用域自动 unlock（即使抛异常也安全）
```

特点：**最轻量**，但功能单一——不能手动解锁，不能延迟加锁。

### 2.3 灵活的 `unique_lock`（C++11）

`lock_guard` 的"加强版"，支持更多控制：

```cpp
std::unique_lock<std::mutex> lock(mtx);          // 立即加锁
std::unique_lock<std::mutex> lock(mtx, std::defer_lock);  // 先不加锁
lock.lock();      // 手动加锁
lock.unlock();    // 手动解锁
// 还可以转移所有权（可移动）
```

它比 `lock_guard` 略重（内部有状态标志），但**条件变量必须配合它使用**（见后文）。

> **选择原则**：默认用 `lock_guard`，需要手动控制/条件变量/可移动时才用 `unique_lock`。

### 2.4 各种 mutex 类型

| 类型 | 版本 | 用途 |
|------|------|------|
| `std::mutex` | C++11 | 基础互斥量 |
| `std::recursive_mutex` | C++11 | 同一线程可重复加锁（避免递归调用自死锁） |
| `std::timed_mutex` | C++11 | 支持 `try_lock_for` / `try_lock_until` 超时 |
| `std::recursive_timed_mutex` | C++11 | 递归 + 超时 |
| `std::shared_mutex` | **C++17** | 读写锁（多读单写） |
| `std::shared_timed_mutex` | C++14 | 读写锁 + 超时 |

### 2.5 重点：读写锁 `std::shared_mutex`（C++17）

很多场景是**读多写少**（如配置、缓存）。普通 mutex 让读操作也互斥，浪费性能。读写锁允许：
- **多个读者同时持有共享锁**
- **写者独占**

```cpp
#include <shared_mutex>
std::shared_mutex rw_mtx;
int data = 0;

int read() {
    std::shared_lock lock(rw_mtx);   // 共享锁（读），可多个并发
    return data;
}

void write(int v) {
    std::unique_lock lock(rw_mtx);   // 独占锁（写）
    data = v;
}
```

- 读用 `std::shared_lock`，写用 `std::unique_lock`。

> 详见内核对比：[[spin-lock-vs-mutex]]

### 2.6 一次性初始化：`std::call_once`（C++11）

保证某段代码在多线程下**只执行一次**（典型单例初始化）：

```cpp
std::once_flag flag;
void init() {
    std::call_once(flag, [](){ /* 只执行一次 */ });
}
```

### 2.7 死锁与多锁：`std::lock` / `std::scoped_lock`

**死锁经典场景**：线程A锁住mtx1等mtx2，线程B锁住mtx2等mtx1。

C++11 提供 `std::lock` 一次性按无死锁算法锁多个：

```cpp
std::lock(mtx1, mtx2);  // 要求两个都锁上，内部避免死锁
std::lock_guard<std::mutex> g1(mtx1, std::adopt_lock);
std::lock_guard<std::mutex> g2(mtx2, std::adopt_lock);
```

C++17 的 `std::scoped_lock` 把这个模式简化为一行（**推荐**）：

```cpp
std::scoped_lock lock(mtx1, mtx2);  // 一次锁多个，自动避免死锁 + RAII
```

### 2.8 条件变量 `std::condition_variable`（C++11）

用于**线程间等待/通知**，比如生产者-消费者：

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> q;

// 消费者
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, []{ return !q.empty(); });  // 队列空就释放锁并睡眠
int item = q.front(); q.pop();

// 生产者
{ std::lock_guard<std::mutex> l(mtx); q.push(42); }
cv.notify_one();  // 唤醒一个等待者
```

**关键点**：
- `wait` 必须配 `unique_lock`（因为要在等待时解锁、唤醒后重新加锁）。
- 一定要用**带谓词的 wait**（lambda），防止**[[虚假唤醒]]（spurious wakeup）**。

---

## 三、原子变量（`std::atomic`，C++11）

### 3.1 基本概念

`std::atomic<T>` 让对 `T` 的操作**不可分割**，无需加锁（对于简单类型通常用 CPU 原子指令实现，**lock-free**）：

```cpp
#include <atomic>
std::atomic<int> counter{0};

void add() {
    counter++;            // 原子，安全！
    counter.fetch_add(1); // 等价写法
}
```

常用成员函数：
- `load()` / `store()`：原子读/写
- `fetch_add` / `fetch_sub` / `fetch_and` / `fetch_or`
- `exchange(v)`：原子地设新值、返回旧值
- `compare_exchange_weak/strong`：**CAS（比较并交换）**，无锁数据结构的核心

```cpp
// CAS 示例：只有当 expected==当前值 时才写入 desired
int expected = 0;
counter.compare_exchange_strong(expected, 10);
```

可用 `is_lock_free()` 检查是否真正无锁。

### 3.2 内存序（memory_order）—— 最难也最重要

原子操作不仅保证"不可分割"，还控制**编译器/CPU 对内存操作的重排序**。这是因为现代 CPU 和编译器为了性能会乱序执行。

C++ 提供六种内存序：

| 内存序 | 含义 |
|--------|------|
| `memory_order_relaxed` | 只保证原子性，**不保证顺序**。性能最高 |
| `memory_order_acquire` | 读操作：后续读写不能重排到它之前 |
| `memory_order_release` | 写操作：之前读写不能重排到它之后 |
| `memory_order_acq_rel` | 同时具备 acquire + release（用于读改写操作） |
| `memory_order_seq_cst` | **顺序一致性**，最强、默认值。所有线程看到一致的全局顺序 |
| `memory_order_consume` | acquire 的弱化版（实际很少用，多数实现退化为 acquire） |

**最经典模式：acquire-release 配对**（实现"发布"数据）：

```cpp
std::atomic<bool> ready{false};
int data = 0;

// 生产者线程
data = 42;                              // ①
ready.store(true, std::memory_order_release);  // ② 之前的写不会重排到此之后

// 消费者线程
while (!ready.load(std::memory_order_acquire)) {}  // ③ acquire
assert(data == 42);  // ④ 保证能看到 data=42（① happens-before ④）
```

**建议**：不确定时就用默认的 `seq_cst`（最安全），等真正需要极致性能且理解透彻后再用 relaxed/acquire/release。

### 3.3 `std::atomic_flag`

最底层、保证一定 lock-free 的原子布尔，常用于实现**[[spin-lock-vs-mutex|自旋锁]]**：

```cpp
std::atomic_flag lock = ATOMIC_FLAG_INIT;
void spin_lock() {
    while (lock.test_and_set(std::memory_order_acquire)) {}  // 自旋等待
}
void spin_unlock() {
    lock.clear(std::memory_order_release);
}
```

C++20 才给它加了 `test()` 等方法，C++11~17 它功能很有限。

---

## 四、锁 vs 原子：怎么选？

| 维度 | 锁（mutex） | 原子（atomic） |
|------|------------|---------------|
| 适用场景 | 保护**一段代码 / 复合操作** | 保护**单个变量的简单操作** |
| 性能 | 有系统调用开销，可能阻塞 | 通常无锁，更快 |
| 复杂度 | 简单直观 | 内存序难掌握，易出微妙 bug |
| 死锁风险 | 有 | 无 |

**经验法则**：
- 简单计数器、标志位 → `std::atomic`
- 保护多个变量、复杂逻辑、调用其他函数 → mutex
- 读多写少 → `std::shared_mutex`(C++17)
- 不确定 → 先用 mutex，正确比快重要

## 相关笔记

- [[原子变量深度剖析]] — 原子变量的硬件原理与内存序
- [[reentrant-lock]] — 内核态可重入锁，与用户态 mutex 对照
- [[spin-lock-vs-mutex]] — 自旋锁与互斥锁的取舍
