# 并发与性能速查

## 线程基础

- `std::thread`（11）裸线程：析构前必须 `join()`/`detach()`，否则 `std::terminate`。**首选 `std::jthread`（20）**：析构自动请求停止并 join，配套 `std::stop_token` 协作取消。
- 线程池：标准库没有，选 Asio 的 `thread_pool`、自研或知名库；别每任务一线程。
- 同步原语速查：`mutex`+`lock_guard`/`unique_lock` / `shared_mutex`（读写锁）/ `recursive_mutex`（设计坏味道信号）/ `condition_variable`（必须配谓词循环防虚假唤醒）/ `scoped_lock`（一次锁多个，死锁免疫）/ `latch`（一次性倒计时）/ `barrier`（循环同步）/ `counting_semaphore`（资源计数）。
- `std::async`（11）坑多（返回 future 析构阻塞、未指定懒/急执行），新代码别用；用线程池 + packaged_task 或协程。

## 原子与内存模型

```cpp
std::atomic<int> x{0};
x.store(1, std::memory_order_release);
bool ok = x.load(std::memory_order_acquire) == 1;   // acquire/release 配对：同步发布
```
- 六级内存序：`relaxed`（仅原子性）/ `consume`（**C++26 已弃用**，实践从不编译成真依赖序）/ `acquire`/`release`（获得-释放配对，日常 95% 场景）/ `acq_rel`（RMW）/ `seq_cst`（全序，默认，略贵）。
- 常用原子技巧：引用计数（`shared_ptr` 控制块内部就是它）、标志位发布、`compare_exchange` 自旋。
- `std::atomic<T>::wait/notify`（20）：比条件变量轻的阻塞等待。
- `std::atomic<std::shared_ptr<T>>`（20）：跨线程共享 shared_ptr 的正规姿势。
- **无锁 ≠ 更快**：单生产者-单消费者队列无锁赚，激烈竞争的 CAS 旋转变慢；先测再上。无锁结构要过 TSan + 内存序理论双重审查。
- C++26 新援：`<hazard_pointer>`（风险指针，安全内存回收）、`<rcu>`（读-复制-更新，读多写少）。

## 协程与结构化并发

- C++20 协程是机制层（见 modern-cpp-core.md）；调度层的三条路：
  1. **Boost.Asio 协程**（`co_await async_read(...)`，工业级 IO 协程事实标准）
  2. 自研/第三方 task 库（cppcoro、libunifex）
  3. **`std::execution`（C++26）**：sender/receiver 模型，`std::execution::task`，源自 Nvidia stdexec——未来统一形态，落地初期能用 Asio 就先用 Asio
- 取消传播：`stop_token`（thread 层）与协程取消（awaiter 检查）要打通，泄漏协程帧是生产高发 bug（用 RAII scope + cancel_and_join 模式收口）。

## 性能优化方法论

**先测量再优化**：perf（Linux）/ Instruments（macOS·iOS）/ VTune（Windows）+ 微基准（Google Benchmark / nanobench），没有 profile 数据的优化是玄学。

高频优化点（按投入产出排序）：
1. **算法与数据结构**：O(n²)→O(n log n) 赢过一切微优化
2. **内存布局**：缓存友好（SoA vs AoS、热点字段前置、`assume_aligned`）；`std::vector` 连续性是王道，`std::list` 几乎总是错（缓存不友好）
3. **减少分配**：reserve、对象池、`inplace_vector`（26）、`hive`（26）、arena allocator；SSO（string 小串优化）意识
4. **拷贝消除**：`std::move` 恰当处、string_view/span 传参、RVO（C++17 保证）
5. **分支与内联**：`[[likely]]`/`[[unlikely]]`（20）、`__builtin_expect`、PGO；虚调用热点用 CRTP/devirtualization
6. **SIMD**：手写 intrinsics 最后再上；C++26 `std::simd` 提供可移植数据并行类型
7. 编译选项：`-O2` 基线（`-O3` 测过再上）、LTO、`-march=native`（部署目标可控时）

## 性能反模式清单

- `std::shared_ptr` 按值传递打穿热路径（原子计数 + 可能的控制块缓存行乒乓）
- 逐元素 `push_back` 不 reserve
- 返回 `std::string_view` 指向函数内局部 string
- 锁粒度过大：整库一把大锁 → 读写锁/分片/无锁
- 热路径 `std::function`（类型擦除 + 可能堆分配）→ 模板参数/`move_only_function`（23）
- `std::regex` 热路径（慢出名）→ 手写解析器/RE2
- 虚函数单行调用在 SIMD 内层循环里

## UB 与安全加固

- 未初始化读：C++26 前是 UB，C++26 起为 erroneous behavior（实现须定义行为）；编译器 `-Wuninitialized`/`-Wsometimes-uninitialized` 常抓漏。
- 整数溢出：有符号溢出是 UB；用 `-ftrapv`、`__builtin_add_overflow` 或 C++26 饱和算术 `std::add_sat`。
- 越界：`std::span` + `at()`（带检查）调试期用，release 走断言前置；C++26 bounds-hardened 标准库。
- 悬垂：`string_view`/`span` 不拥有——**生命周期纪律**：视图不得活过被视物。
- 排错工具链见 toolchain-debugging.md（ASan/UBSan/TSan/Valgrind）。
