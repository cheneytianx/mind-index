---
tags: [lock, mutex, concurrent, kernel, cpp]
created: 2026-06-23
source: conversation
---

# 可重入锁（Reentrant Lock）

> 同一线程持有锁的情况下再次获取同一把锁，不会死锁，而是给持有计数 +1。

可重入锁的核心矛盾：它把"不知道自己在做什么"从运行时错误变成运行时允许——这是福是祸，取决于你对调用图的掌控力。

---

## 一、它解决什么问题？

### 显式递归

```cpp
void foo() {
    std::lock_guard lock(mtx);
    bar();  // bar 里也要加同一把锁
}
void bar() {
    std::lock_guard lock(mtx);  // 普通 mutex → [[死锁]]
}
```

### 隐式虚函数回调（更隐蔽、更危险）

```cpp
class Base {
    std::mutex mtx;
public:
    void process() {
        std::lock_guard lock(mtx);  // ① 加锁
        onProcess();                 // ② 调用虚函数
    }
    virtual void onProcess() {}
};

class Derived : public Base {
    void onProcess() override {
        process();  // 再次进入 Base::process → lock(mtx) → [[死锁]]
    }
};
```

这个模式在 GUI 框架、事件系统、[[观察者模式]] 中极其常见——**你不知道回调里会做什么**。

---

## 二、底层实现

普通 mutex 是二分状态 `locked/unlocked`，可重入锁升级为三元组：

```
{ owner_thread_id, count, base_lock }
```

```c
void recursive_mutex_lock(mutex *m) {
    if (m->owner == current_thread) {
        m->count++;    // 同一线程持有，直接计数 +1
        return;
    }
    // 否则走标准 [[Futex]] 竞争
    while (atomic_cas(&m->base_lock, 0, 1) != 0) {
        futex_wait(&m->base_lock, 1);
    }
    m->owner = current_thread;
    m->count = 1;
}

void recursive_mutex_unlock(mutex *m) {
    assert(m->owner == current_thread);  // 只能持有者解锁
    m->count--;
    if (m->count == 0) {
        m->owner = 0;
        atomic_store(&m->base_lock, 0);
        futex_wake(&m->base_lock, 1);
    }
}
```

### 与普通 Mutex 对比

| | 普通 Mutex | 可重入锁 |
|---|---|---|
| 状态 | locked / unlocked | `{谁持有, 持有了几次}` |
| 同一线程再 lock | [[死锁]] | count++ |
| 存储开销 | 4 字节 | ~12 字节（owner + count + futex） |
| 快速路径 | 单次 CAS | 线程 ID 比较 + CAS |
| unlock 语义 | 任何一次 unlock 就释放 | **必须 count 降为 0 才真正释放** |

---

## 三、C++ 中的 `std::recursive_mutex`

详见 [[C++并发控制机制]] 第 2.4 节。

```cpp
#include <mutex>
std::recursive_mutex rmtx;

void recursive_func(int n) {
    std::lock_guard<std::recursive_mutex> lock(rmtx);
    if (n > 0) recursive_func(n - 1);  // 同一线程重复加锁，不会死锁
    // 每层递归 count+1，退出时 count-1
}
```

**C++ 标准要求：**
- `lock()` 可被同一线程多次调用
- **必须**调用等次数的 `unlock()` 才能真正释放锁
- 持有权不可在线程间转移
- 最大递归深度未指定（通常很大）

`std::recursive_timed_mutex`（C++11）支持 `try_lock_for` / `try_lock_until`。

> ⚠️ C++17 的 `std::scoped_lock` + `recursive_mutex` 仍要求递归计数匹配，不能减少 unlock 次数。

---

## 四、Linux 内核的态度：原则性不提供可重入锁

内核的 `spin_lock()` 和 `mutex_lock()` 都是**不可重入**的——持有后再次 lock 直接死锁。

**Linus 的观点（来自 LKML 多次讨论）：**

> "可重入锁是设计缺陷的遮羞布。如果你的代码需要可重入锁，说明你的锁范围划分有根本性问题——你加锁时不知道自己在做什么，不知道自己在保护什么数据。"

**内核社区的理由：**

1. **锁应该保护数据，而非保护代码路径。** 拿了一把锁就应该明确知道它在保护哪些数据结构。
2. **可重入锁让"意外持有锁"静默通过**，隐藏真正的问题。
3. **内核调用图极其复杂 + [[中断上下文]]**，可重入锁会让"锁在中断里被重复持有"变成静默行为而非即时的 bug 暴露。
4. **性能**——每次 lock/unlock 都要检查 `owner == current_thread`，热路径不可接受。

### 内核中"类可重入"的例外

`down_read()` 嵌套调用（[[读写信号量]] 的读锁）允许多个读者共存，但这不是经典"同一线程互斥重入"的语义——它只是不需要互斥。

---

## 五、工程判断：该不该用？

| 立场 | 理由 |
|---|---|
| **反对派**（内核社区、底层系统开发者） | 可重入锁是坏设计的标志，掩盖了锁范围问题，应重构代码 |
| **务实派**（应用开发者、GUI 框架作者） | 回调/事件系统天然存在"不知道会不会回来"的路径，重构成本太高 |
| **折衷派** | 可以用，但必须在文档中明确声明，且持续审视是否有更好的解耦方式 |

### 更好的替代方案（不用可重入锁时）

- **拆分锁粒度**：不保护整个调用树，只保护具体数据
- **免锁重入设计**：外层加锁后传 `already_locked` 标志给内层（Qt 常用）
- **消息队列化**：把回调推迟到锁释放后执行

---

## 六、一句话总结

**可重入锁的本质**：内核场景下是"设计缺陷的遮羞布"，用户态框架场景下是"务实工程的选择"——判断标准是调用图的**可预测性**：确定性越高越不该用，不可预测的回调路径越多越值得用。

---

## 相关概念

[[spin-lock-vs-mutex|Spin Lock]] · [[spin-lock-vs-mutex|Mutex]] · [[Futex]] · [[死锁]] · [[RAII]] · [[中断上下文]] · [[读写信号量]] · [[观察者模式]] · [[C++并发控制机制]]
