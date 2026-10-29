# 现代 C++ 核心机制速查（所有权 / 移动 / 模板与 concepts / 协程 / ranges / 错误处理）

## 所有权与 RAII

- **RAII 是 C++ 的第一原则**：资源生命周期 = 对象生命周期。构造获取、析构释放、异常安全免费获得。
- 所有权三问：谁拥有？独占还是共享？何时释放？——决定了用 `unique_ptr` / `shared_ptr` / 值语义 / 引用（非拥有）。
- **默认顺序**：值 > `unique_ptr` > `shared_ptr`。`shared_ptr` 是为「最后一个用完才释放」的共享所有权设计的，不是「想要 GC 的替代品」——引用计数有原子开销，且循环引用要 `weak_ptr` 打破。
- 参数传递速查（T 可拷贝、移动廉价的小类型）：
  - 只读小对象：`const T&` 或直接 `T`
  - 只读大对象：`const T&`
  - 要存进去（sink）：按值 `T` + 内部 `std::move`，或 `T&&` 重载（前者更简洁）
  - 要修改但不存：`T&`
  - 字符串字面量兼容：`std::string_view` 只读 / `std::string` sink

## 移动语义

- `std::move` 不移动任何东西——只是到右值的转换；真正的移动发生在移动构造/赋值里。`std::move` 后的对象处于「有效但未指定」状态，只能赋值、析构或重置。
- `std::forward<T>` 完美转发：模板 `T&&`（转发引用）时保持实参左右性。
- **noexcept 关键**：移动构造不标 `noexcept`，`std::vector` 扩容会退化为拷贝（`std::move_if_noexcept`）。所有自定义移动构造默认应 `noexcept`。
- 五法则/零法则：定义了析构/拷贝/移动任一，就要通盘考虑五个；能靠成员自管理（智能指针、容器）就一个都别写（零法则）。
- RVO：C++17 起按值返回纯右值**保证**消除拷贝；返回 `T{...}` 直接写，不要返回 `std::move(local)`（反而禁用 RVO）。

## 智能指针要诀

```cpp
auto p = std::make_unique<T>(args...);        // 首选
auto s = std::make_shared<T>(args...);        // 控制块与对象一次分配
std::weak_ptr<T> w = s;                       // 观察 shared，不延寿；用前 lock()
```
- `unique_ptr` 零开销抽象（等同裸指针），可自定义 deleter（`unique_ptr<FILE, decltype(&fclose)>`）。
- `shared_ptr` 控制块：强/弱两个计数，原子操作；`enable_shared_from_this` 用于对象内部取自身 shared（异步回调自救）。
- 别用 `shared_ptr` 存进容器再频繁拷贝（原子计数开销）；也别循环引用（经典：节点互相持 shared → 用 weak 打破）。

## 模板与 concepts

- SFINAE（C++98 时代）→ `if constexpr`（C++17，编译期分支，大幅杀 SFINAE）→ **concepts**（C++20，约束一等公民）：
```cpp
template <std::ranges::input_range R>
    requires std::convertible_to<std::ranges::range_reference_t<R>, std::string>
void print_all(R&& r);
// 或简写：void print_all(std::ranges::input_range auto&& r);
```
- concepts 参与重载决议（更约束的胜出）、报错信息可读（不再 200 行模板呕吐物）。
- 标准库 concepts 大族：`same_as`/`convertible_to`、`integral`/`floating_point`、`equality_comparable`/`totally_ordered`、`invocable`/`predicate`、`input_range`/`forward_range`/`sized_range`、`movable`/`copyable`/`default_initializable`。
- 模板健康度：少写递归 TMP，能用 `if constexpr`/concepts/`constexpr` 函数就别用类特化；变长模板包用折叠表达式消费。

## 协程（C++20 机制层）

- 关键字：`co_await` / `co_yield` / `co_return`。函数体含任一即协程，返回类型须满足协程协议。
- 机制五件套：`promise_type`（协程帧内的定制点）、awaiter（`await_ready`/`await_suspend`/`await_resume`）、协程帧（堆分配，可 HALO 优化消除）、`coroutine_handle`（控制权柄）、对称转移。
- **C++20 只给了机制没给库**：生产使用要么用库（cppcoro、libunifex、Boost.Asio 协程），要么等 `std::execution`（C++26）落地，要么自己写 awaiter（小心生命周期）。
- 生命周期红线：协程按引用捕获的局部变量在挂起后可能悬垂——参数按值传、成员用 `shared_ptr`/`unique_ptr` 管住再挂起。
- `std::generator`（C++23）已是标准生成器；lazy task 型库自己挑。

## ranges（C++20）

```cpp
using namespace std::views;
auto sq = iota(1) | transform([](int i){ return i*i; })
                  | filter([](int v){ return v % 2 == 0; })
                  | take(5);
for (int v : sq) std::print("{} ", v);   // 4 16 36 64 100
```
- view 是惰性、零拷贝的管道；`std::ranges::` 算法直接吃容器（`ranges::sort(v)`），带投影（`ranges::sort(people, {}, &Person::age)`）。
- 常用 view：`filter`/`transform`/`take`/`drop`/`take_while`/`drop_while`/`reverse`/`iota`/`keys`/`values`/`zip`（23）/`chunk`/`slide`（23）/`concat`（26）。
- 坑：`filter` 后的 view 不满足 `random_access_range`；`views::transform` 返回引用要小心悬垂；管道对右值容器要 `owning_view` 语义（C++20D 修正）。

## 错误处理策略选型

| 机制 | 适用 | 注意 |
|---|---|---|
| 异常 | 构造失败、深层错误穿透、错误路径不平凡 | 开销在抛出路径；`-fno-exceptions` 项目禁用；析构/移动不抛 |
| 错误码 / bool | 边界 API、C ABI、热路径 | 侵入签名、易被忽略 |
| `std::optional<T>`（17） | 「可能没有值」且无错误信息 | 只表达缺失，不表达原因 |
| **`std::expected<T, E>`（23）** | 可预期的失败、monadic 链 | `and_then`/`transform`/`or_else` 组合；E 建议用轻量错误类型 |
| `std::variant` + visit（17） | 多态结果无堆分配 | |
| contracts（26） | 前置/后置条件断言 | 定稿未普及，观察编译器支持 |

经验法则：可预期失败（文件不存在、解析失败）→ expected/optional；不可预期编程错误（不变量破坏）→ assert/异常；跨模块/进程边界 → 错误码或序列化的错误对象。
