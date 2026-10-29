# 高频坑与最佳实践速查表

## 语言层坑（按翻车率排序）

### 1. 悬垂引用/指针
- `std::string_view`/`std::span` 不拥有数据：`std::string_view sv = std::string("tmp");` 立刻悬垂。
- 返回 `const T&` 的函数内部返回了局部/临时。
- lambda 按引用捕获 `[&]` 后异步执行/存出作用域 → 按值捕获或 `shared_ptr`。
- 协程按引用捕获参数后挂起 → 协程参数一律按值。

### 2. 迭代器失效
- `vector`：插入/删除可能使全部迭代器失效（扩容重分配）；erase 惯用法 `it = v.erase(it);`。
- `map/unordered_map`：删除仅使被删元素迭代器失效；插入使 unordered 迭代器可能失效（rehash 时迭代器全灭、引用/指针仍有效）。
- 遍历中修改容器的正确姿势：索引循环倒序、先收集后处理、或 `std::erase_if`（20）。

### 3. auto 与引用语义
- `auto x = range_ref;` 拷贝；`auto&` 才是引用；`for (auto item : big_vector)` 每个元素深拷贝（写 `const auto&`）。
- `decltype(auto)` 保持引用性（转发场景）。
- 代理类型陷阱：`vector<bool>` 的 `auto` 拿到的是代理对象不是 bool。

### 4. 未初始化与默认值
- 类成员不初始化 = UB（C++26 前）；类内成员初始化器 `int x = 0;` 一律写上。
- `T x;` 与 `T x{};` 天壤之别；`new T` vs `new T()` 同理。
- `std::vector<T> v(n)`：n 个值初始化元素（对内置类型是 0）。

### 5. 整数与转换
- 有符号溢出 UB；无符号回绕（`v.size() - 1` 在空 vector 时爆成 SIZE_MAX）。
- `size_t` vs `int` 比较；`static_cast` 显式写，`-Wconversion` 开着。
- `char` 符号性是实现定义（ARM 上默认 unsigned，老代码移植高频坑）。

### 6. 对象生命周期
- 静态初始化顺序灾难：跨编译单元的全局对象互依 → Meyers singleton（函数内 static）或 `constinit`。
- 析构中回调外部已析构对象：订阅/观察者模式必须 weak 化退订。
- `std::function` 存 self-referential lambda：用 `shared_ptr<lambda>` + `weak_ptr` 打破。

### 7. ODR 与链接
- 头文件里定义非 inline 函数/变量 → 多重定义；头文件定义必须 `inline`（或 C++17 内联变量）。
- 不同编译单元同名类布局不同（条件编译/不同 include 顺序）→ UB，TSan 之外的静默灾难。
- 模板定义放 .cpp 而没显式实例化 → 链接错误。

### 8. 异常安全
- 析构函数绝不抛（`noexcept` 天然）；移动构造标 `noexcept` 否则 vector 扩容退化拷贝。
- `std::vector<std::unique_ptr<T>>` 的删除顺序是逆序——依赖析构顺序要小心。

## 编译器/平台差异坑

| 坑 | 说明 |
|---|---|
| MSVC 两阶段查找 | `/permissive-` 开了之后与 GCC 行为对齐，但历史代码可能炸 |
| `__PRETTY_FUNCTION__` 各家不同 | 调试宏要按编译器分支 |
| `wchar_t` 大小 | Linux 4 字节 / Windows 2 字节——跨平台文本用 UTF-8 + `char8_t`/`std::u8string` 慎重 |
| 名字修饰 | 跨编译器（GCC↔MSVC）不能直接链 C++ 符号 → C ABI 边界（`extern "C"`） |
| `std::filesystem::path` 分隔符 | `/` 通吃，构造时注意 `u8` 前缀与本地编码 |
| ARM 上 `char` 默认 unsigned | 与 x86 不同，`char c = -1; if (c < 0)` 分支行为翻转 |

## 面试高频深挖点（教学场景）

- 虚函数表与多继承布局、虚继承内存代价
- `unique_ptr` 大小（裸指针 + 无状态 deleter = 零开销；函数指针 deleter = +1 指针）
- move 后对象状态（有效但未指定）；为什么 `std::move` 本身不移动
- RVO/NRVO 与 C++17 保证的复制消除区别
- `memory_order_seq_cst` vs `acquire/release` 的重排语义
- `std::function` 的 SBO（小缓冲优化）实现思路
- `shared_ptr` 控制块布局（强/弱计数 + deleter + allocator）
- 为什么 `std::list::size()` 曾是 O(n)（C++11 起强制 O(1)）

## 代码评审清单（Best Practice 精选）

**正确性**
- [ ] 每个资源有明确所有者（RAII）；裸 new/delete 存在即扣分
- [ ] 类成员全部有初始化器
- [ ] 循环中容器修改检查过迭代器失效
- [ ] 跨线程数据访问有同步原语覆盖或证明无竞争
- [ ] `const` 正确性（成员函数/参数/局部）

**性能**
- [ ] 热路径无意外拷贝（range-for 用 `const auto&`，sink 用 move）
- [ ] `reserve`/预分配在已知规模处
- [ ] `noexcept` 移动构造
- [ ] 热路径无 `std::function`/`std::regex`/`shared_ptr` 拷贝

**可维护性**
- [ ] 头文件自包含（include what you use）
- [ ] 无魔数；单位进类型（`std::chrono` 而非裸 int 毫秒）
- [ ] 公共 API 标注标准版本需求；弃用路径 `[[deprecated("...")]]`
- [ ] clang-format/clang-tidy 过基线

## 学习路径建议（用户问「怎么系统学 C++」时）

1. **基础**：《C++ Primer》（5th，覆盖到 11）或 learnpp.com 打底 → 立刻上手写
2. **现代化**：《Effective Modern C++》（Meyers，11/14 惯用法）
3. **深水区**：《C++ Concurrency in Action》（Williams，2nd 含 C++17/20）、《C++ Templates》（Vandevoorde）、《Professional C++》（5th 含 C++20）
4. **跟进**：cppreference.com 当日常手册、C++ Weekly（Jason Turner）、ISO C++ 委员会邮件列表/trip reports、constexpr 未慰藉时看 Godbolt（compiler explorer）实测
5. 节奏：C++11/17 先吃透（生产主力），C++20 按需（concepts/ranges 先行），C++26 关注但按编译器支持节奏采用
