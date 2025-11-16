# ReaderWriterQueue 项目学习笔记

## 项目概述
- **项目名称**：ReaderWriterQueue
- **作者**：Cameron Desrochers
- **许可证**：简化 BSD
- **功能**：C++11+ 无锁单生产者单消费者（SPSC）队列库，提供高效的并发数据结构。
- **主要特点**：
  - 无锁设计，O(1) 入队/出队。
  - 支持动态大小（ReaderWriterQueue）和固定大小（BlockingReaderWriterCircularBuffer）。
  - 避免伪共享，使用缓存行对齐。
  - 兼容现代编译器（GCC 4.7+、MSVC 2010+）。

## 主要文件
- **`readerwriterqueue.h`**：核心无锁队列实现，使用“队列的队列”设计，动态分配 Block。
- **`readerwritercircularbuffer.h`**：固定大小阻塞环形缓冲区，使用信号量实现阻塞。
- **`atomicops.h`**：提供原子操作、内存屏障、轻量级信号量（LightweightSemaphore）。
- **其他**：CMakeLists.txt、tests、benchmarks 用于构建、测试和性能评估。

## 核心概念
### 无锁队列设计
- **SPSC 架构**：仅支持一个生产者和一个消费者，避免竞争。
- **队列的队列**（ReaderWriterQueue）：
  - 由多个 Block（循环缓冲区）链接成链表。
  - 生产者拥有 `tailBlock`，消费者拥有 `frontBlock`。
  - 动态扩展：Block 满时分配新 Block。
- **环形缓冲区**：
  - 使用索引 `front` 和 `tail`，通过 `sizeMask`（size - 1）实现环绕：`index & sizeMask`。
  - 浪费一个槽位区分空/满状态。
- **避免伪共享**：变量对齐到缓存行，使用填充字节。
  - **伪共享定义**：多个线程访问同一缓存行（通常 64 字节）上的不同变量，导致缓存失效，降低性能。即使变量逻辑独立，也会触发缓存一致性协议开销。
  - **对齐实现**：使用宏 `MOODYCAMEL_MAYBE_ALIGN_TO_CACHELINE` 对齐类/结构体到缓存行边界；添加 `cachelineFiller` 填充字节，确保高竞争变量（如 `front`、`tail`、`nextSlot`、`nextItem`）位于不同缓存行。
  - **效果**：减少缓存失效，提高多核性能。未对齐时，伪共享可导致 2-10 倍性能损失；对齐后，吞吐量显著提升，内存使用略增但收益大于成本。

### 阻塞机制
- **信号量**：使用 `LightweightSemaphore`，基于原子操作和平台特定信号量（Windows Semaphore、POSIX sem_t 等）。
- **阻塞操作**：
  - `slots_`：跟踪空槽位，入队时 `wait()` 阻塞，出队时 `signal()` 唤醒。
  - `items`：跟踪元素，出队时 `wait()` 阻塞，入队时 `signal()` 唤醒。
- **超时支持**：`timed_wait(usecs)` 允许带超时的阻塞。

### 原子操作和内存屏障
- **`weak_atomic<T>`**：轻量原子类型，无内存顺序保证，仅用于硬件支持的原子类型。
- **内存屏障**：`fence(memory_order)` 确保操作顺序，避免数据竞争。
  - `memory_order_acquire`：读取屏障。
  - `memory_order_release`：写入屏障。
- **ThreadSanitizer 支持**：宏 `AE_NO_TSAN` 禁用 TSan 检查。

## 关键代码解释
### 模板和宏
- **类模板**：`template<typename T, size_t MAX_BLOCK_SIZE = 512>`，支持任意类型和块大小。
- **宏**：
  - `MOODYCAMEL_MAYBE_ALIGN_TO_CACHELINE`：对齐到缓存行。
  - `AE_NO_TSAN`：TSan 注解。
  - `explicit`：防止隐式转换。

### Block 结构体
- **成员**：`front`、`tail`（原子）、`data`（存储区）、`sizeMask`。
- **环绕**：索引更新使用 `& sizeMask`。
- **填充**：`cachelineFiller` 避免伪共享。

### 方法
- **`try_enqueue`/`try_dequeue`**：非阻塞，立即返回成功/失败。
- **`enqueue`/`dequeue`**：可能阻塞（ReaderWriterQueue 无阻塞，BlockingReaderWriterCircularBuffer 有）。
- **`peek`/`pop`**：查看/移除而不返回元素。
- **构造函数**：初始化 Block 链表或固定缓冲区。

## 使用示例
```cpp
#include "readerwriterqueue.h"

// 动态队列
moodycamel::ReaderWriterQueue<int> q(100);
q.enqueue(42);
int value;
if (q.try_dequeue(value)) { /* 使用 value */ }

// 阻塞队列
moodycamel::BlockingReaderWriterCircularBuffer<int> bq(100);
bq.wait_enqueue(42);
bq.wait_dequeue(value);