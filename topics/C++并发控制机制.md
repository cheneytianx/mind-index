
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

# 原子变量深度剖析
## 第一层：硬件为什么会"乱"？

要理解原子变量，先要理解它在对抗什么。现代 CPU 有三个"捣乱者"：

### 1.1 缓存（Cache）—— 可见性问题的根源

每个 CPU 核心都有自己的私有缓存（L1/L2）：

```
核心0 [L1] [L2] ┐
                 ├── L3共享 ── 主内存
核心1 [L1] [L2] ┘
```

核心0 写了一个变量，可能只写进了自己的 L1 缓存，核心1 读的还是主内存或自己缓存里的旧值。这就是**可见性（visibility）问题**。

> 注：现代 x86/ARM 都有**[[缓存一致性协议|缓存一致性协议（MESI）]]**保证缓存最终一致，但"最终"和"立即"之间的时间窗口，加上下面的乱序，才是真正的麻烦。

### 1.2 Store Buffer（写缓冲）—— 乱序的真凶之一

CPU 写内存很慢。为了不阻塞，写操作先进入 **Store Buffer**，CPU 继续执行后面的指令：

```
CPU执行 store x=1  → 进入Store Buffer（还没到缓存）
CPU执行 load y     → 直接从缓存读
```

结果：在别的核心看来，`load y` 仿佛发生在 `store x` 之前——**写-读被重排了**。这正是著名的 **[[Store Buffer|StoreLoad 重排]]**，也是 x86 唯一允许的重排。

### 1.3 编译器重排 + CPU 乱序执行

```cpp
a = 1;   // 编译器/CPU 可能把这两行
b = 2;   // 调换顺序执行（只要单线程看不出区别）
```

单线程下"as-if"规则保证结果不变，但**多线程下这种重排会暴露**。

**结论**：原子变量要解决两件事——**原子性**（不可分割）和**有序性/可见性**（控制重排和传播）。

---

## 第二层：原子性是怎么实现的？

### 2.1 普通操作为什么不原子

```cpp
counter++;
```

编译成汇编大致是：

```asm
mov eax, [counter]   ; 读
inc eax              ; 改
mov [counter], eax   ; 写
```

两个核心同时执行，可能都读到 5，各自加成 6，写回 6——丢了一次更新。

### 2.2 硬件原子指令

CPU 提供专门的原子指令。x86 上是 `lock` 前缀：

```asm
lock xadd [counter], eax   ; 原子地 fetch_add
lock cmpxchg [ptr], ebx    ; 原子地 CAS
```

`lock` 前缀会锁住缓存行（现代 CPU 用缓存锁定，而非锁总线），保证这条指令期间别的核心动不了这块内存。

> 内核侧对比：[[spin-lock-vs-mutex]] 中也有 spinlock 的硬件基础讨论。

### 2.3 `is_lock_free()` 与锁兜底

```cpp
std::atomic<int> a;        // 通常 lock-free（硬件直接支持）
std::atomic<BigStruct> b;  // 太大，硬件没有对应指令
```

当类型太大（如 100 字节的结构体），硬件没有对应原子指令时，标准库会**偷偷用一个锁**来模拟原子性。这时 `b.is_lock_free()` 返回 `false`。所以"atomic"不等于"一定无锁"。

```cpp
std::atomic<int> a;
std::cout << a.is_lock_free();  // 几乎一定是 true
```

---

## 第三层：[[Compare-and-Swap|CAS]] —— 无锁世界的基石

`compare_exchange` 是几乎所有[[无锁编程|无锁算法]]的核心，必须吃透。

### 3.1 语义

```cpp
bool compare_exchange_strong(T& expected, T desired);
```

逻辑（**原子地**完成）：

```
if (当前值 == expected) {
    当前值 = desired;
    return true;
} else {
    expected = 当前值;   // 注意！失败时会把当前值写回 expected
    return false;
}
```

失败时把真实值塞回 `expected`，是为了让你**下一轮重试**用最新值。

### 3.2 典型用法：CAS 循环

实现一个原子的"乘以2"（标准库没有 `fetch_mul`）：

```cpp
std::atomic<int> value{1};

void multiply_by_2() {
    int old = value.load();
    while (!value.compare_exchange_weak(old, old * 2)) {
        // 失败说明被别人改了，old 已被更新为最新值，自动重试
    }
}
```

这个"读 → 算 → CAS → 失败重试"的循环，是无锁编程的**标准范式**。

### 3.3 weak vs strong

| | `compare_exchange_weak` | `compare_exchange_strong` |
|--|------------------------|--------------------------|
| 虚假失败 | **可能**（即使值相等也可能返回false） | 不会 |
| 性能 | 在某些平台（ARM）更快 | 可能内部加循环 |
| 用法 | **循环中用 weak** | 不循环时用 strong |

> 为什么 weak 会"虚假失败"？因为 ARM 等用 **LL/SC（Load-Linked/Store-Conditional）** 实现，中断或缓存行被触碰就会导致 SC 失败。在循环里这无所谓（反正会重试），还省去了内部额外循环的开销。

### 3.4 [[ABA问题]]（无锁编程的经典陷阱）

CAS 只比较"值是否相等"，但值可能经历了 **A → B → A** 的变化：

```
线程1读到指针 A，准备 CAS
线程2：A出栈→B出栈→A又入栈（内存地址复用）
线程1：CAS 发现还是 A，成功！但其实结构已经变了 → 出错
```

解决方案：
- **带版本号的指针**（tagged pointer）：每次修改递增计数器，比较 `{指针, 版本号}`。
- **[[Hazard Pointer]] / [[RCU]]** 等[[内存回收]]技术。

这是写无锁栈/队列时绕不开的坑。

---

## 第四层：内存序 —— happens-before 的本质

现在进入最核心、也最难的部分。把前面三层串起来。

### 4.1 核心概念：[[happens-before]]

如果操作 A *happens-before* 操作 B，那么 A 的所有效果对 B **可见**。[[内存序]]的本质，就是**建立跨线程的 happens-before 关系**。

### 4.2 relaxed —— 只有原子性，没有顺序

```cpp
std::atomic<int> x{0}, y{0};

// 线程1                    // 线程2
x.store(1, relaxed);        r1 = y.load(relaxed);
y.store(1, relaxed);        r2 = x.load(relaxed);
// 可能出现 r1==1 但 r2==0！
```

`relaxed` 只保证**单个变量自身的修改顺序**（modification order）一致，**不建立任何跨变量、跨线程的顺序**。

**适用场景**：纯计数器（如统计请求数），不依赖其他数据：

```cpp
std::atomic<long> hits{0};
hits.fetch_add(1, std::memory_order_relaxed);  // 只要总数对就行
```

### 4.3 release-acquire —— 配对建立"同步"

这是**最重要、最实用**的内存序模型。

```cpp
std::atomic<bool> ready{false};
int data = 0;   // 普通变量

// 生产者
data = 42;                          // ① 普通写
ready.store(true, release);         // ② release 写

// 消费者
while (!ready.load(acquire)) {}     // ③ acquire 读
std::cout << data;                  // ④ 一定看到 42
```

**规则**：
- **release** 像一道"栅栏"：它**之前**的所有读写（含①），都不能重排到它**之后**。
- **acquire** 像另一道"栅栏"：它**之后**的所有读写（含④），都不能重排到它**之前**。
- 当 acquire 读到了 release 写的那个值时，**配对成功**，建立 happens-before：① → ④ 可见。

形象记忆：
```
release：把栅栏前的货物全部"发布"出去  ┐
                                        ├─ 配对后形成单向同步
acquire：把栅栏后的操作全部"接收"进来  ┘
```

注意是**单向**的：release 之后的写不保证、acquire 之前的读不保证。

### 4.4 acq_rel —— 用于读改写

像 `fetch_add`、`exchange`、`compare_exchange` 这种**既读又写**的操作，用 `acq_rel`：读的部分是 acquire，写的部分是 release。

```cpp
// 自旋锁的解锁/加锁
flag.exchange(true, std::memory_order_acq_rel);
```

### 4.5 seq_cst —— 全局唯一顺序

最强保证：所有 `seq_cst` 操作存在一个**全局总顺序**，所有线程看到的顺序一致。

回到 4.2 那个例子，如果全用 `seq_cst`，就**不可能**出现 `r1==1 && r2==0`。

代价：x86 上 `seq_cst` 的 store 需要插入 `mfence` 或用 `xchg`，比 release 贵；ARM 上代价更明显。

> **默认即 seq_cst**：`x.store(1)` 等价于 `x.store(1, memory_order_seq_cst)`。

### 4.6 六种内存序总览（按强度排序）

```
relaxed  <  consume  <  acquire/release  <  acq_rel  <  seq_cst
 最弱最快                                              最强最慢最安全
```

| 内存序 | 用在哪种操作 | 保证 |
|--------|-------------|------|
| relaxed | load/store/RMW | 仅原子性 + 单变量修改顺序 |
| acquire | load | 后续操作不上移；配对 release 建立同步 |
| release | store | 之前操作不下移；配对 acquire 建立同步 |
| acq_rel | RMW | 同时 acquire+release |
| seq_cst | 全部 | 全局总顺序（默认） |
| consume | load | acquire 的弱化（依赖链），实践中几乎被废弃 |

---

## 第五层：[[内存栅栏]] `std::atomic_thread_fence`

除了把内存序附加在原子操作上，还能**独立**插入栅栏：

```cpp
data = 42;
std::atomic_thread_fence(std::memory_order_release);  // 独立栅栏
ready.store(true, std::memory_order_relaxed);  // 即使是 relaxed store

// 另一端
while (!ready.load(std::memory_order_relaxed)) {}
std::atomic_thread_fence(std::memory_order_acquire);
assert(data == 42);  // 依然成立
```

栅栏与"附加在原子上的内存序"效果类似，但更灵活——可以把栅栏和原子操作分开，有时能减少栅栏数量、优化性能。一般情况优先用附加在原子操作上的内存序，更直观。

---

## 第六层：实战 —— 自己写一个无锁栈

把所有知识串起来，看 CAS + 内存序如何协作：

```cpp
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
    };
    std::atomic<Node*> head{nullptr};

public:
    void push(T value) {
        Node* node = new Node{value, nullptr};
        node->next = head.load(std::memory_order_relaxed);
        // CAS 循环：直到成功把 node 接到链表头
        while (!head.compare_exchange_weak(
                   node->next, node,
                   std::memory_order_release,   // 成功：release，发布 node
                   std::memory_order_relaxed))  // 失败：relaxed，重试
        {}  // 失败时 node->next 被自动更新为最新 head
    }

    bool pop(T& result) {
        Node* old_head = head.load(std::memory_order_acquire);
        while (old_head &&
               !head.compare_exchange_weak(
                   old_head, old_head->next,
                   std::memory_order_acquire,
                   std::memory_order_relaxed))
        {}
        if (!old_head) return false;
        result = old_head->data;
        // delete old_head;  // 危险！这里有 ABA + 内存回收问题
        return true;
    }
};
```

**这段代码体现的要点**：
1. `push` 用 CAS 循环把新节点接到头部。
2. 成功时用 `release`，对应 `pop` 的 `acquire`，保证 `node->data` 对取出者可见。
3. 失败用 `relaxed`，因为还要重试，无需同步。
4. **`pop` 里的 `delete` 被注释掉**——直接删会引发 **ABA** 和"别的线程正在读已删节点"的问题。工业级实现需要 **Hazard Pointer** 或 **RCU**。这说明：**无锁 ≠ 简单**，内存回收是最大难点。

---

## 第七层：常见误区与最佳实践

### 误区
1. **"用了 atomic 就线程安全了"**——单个操作原子，但**多个原子操作的组合不原子**：
   ```cpp
   if (a.load() == 0)   // 这之间可能被别的线程改
       a.store(1);      // 要用 CAS 才对
   ```
2. **`atomic<int> a; a = a + 1;`** 不是原子的！这是"load + add + store"三步。要用 `a.fetch_add(1)` 或 `a++`。
3. **滥用 relaxed**——为了"性能"乱用，导致极难复现的 bug。

### 最佳实践
1. **默认用 `seq_cst`**（即不写内存序参数），正确第一。
2. 性能瓶颈确认在原子操作上，再考虑 acquire/release。
3. **relaxed 只用于独立计数器**这类不依赖其他数据的场景。
4. 无锁数据结构优先用**成熟的库**（如 `folly`、`boost.lockfree`），别自己造。
5. 能用 mutex 解决且性能够用时，**别上无锁**——可维护性远比那点性能重要。

---

## 全局脉络总结

```
硬件层：缓存 + Store Buffer + 乱序执行  →  造成可见性和重排问题
   ↓
原子性：lock 前缀 / LL-SC 指令  →  保证操作不可分割
   ↓
CAS：compare_exchange  →  无锁算法的基石（注意 weak/strong、ABA）
   ↓
内存序：建立 happens-before  →  控制可见性与重排
   ├─ relaxed：仅原子性
   ├─ release/acquire：配对同步（最常用）
   ├─ acq_rel：读改写
   └─ seq_cst：全局顺序（默认、最安全）
   ↓
栅栏：atomic_thread_fence  →  独立控制
   ↓
应用：无锁数据结构（难点在内存回收）
```

**一句话精髓**：原子变量 = **原子性**（硬件指令保证）+ **有序性**（内存序建立 happens-before）。前者解决"不可分割"，后者解决"什么时候、对谁可见"。

---

想继续深入哪个方向？我可以展开：
- **happens-before 的完整形式化定义**（sequenced-before / synchronizes-with / inter-thread happens-before）
- **MESI 缓存一致性协议**的具体状态转换
- **Hazard Pointer / RCU** 如何解决无锁内存回收
- **x86 vs ARM** 内存模型的实际差异与对应汇编