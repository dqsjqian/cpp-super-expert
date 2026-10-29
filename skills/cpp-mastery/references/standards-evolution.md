# C++ 标准演进全景（C++98/03 → 11 → 14 → 17 → 20 → 23 → 26）

> 事实基线（2026-09）：C++23 全面落地；C++26 已于 2026-03 定稿（伦敦会议），GCC 16.1 支持其大部分特性，Clang/MSVC 部分支持。

## 一图流：每代一句话

| 标准 | 发布 | 一句话定位 |
|---|---|---|
| C++98/03 | 1998/2003 | 面向对象 + STL + 异常 + 模板元编程雏形 |
| C++11 | 2011 | 「现代 C++」起点：移动语义、lambda、auto、智能指针、并发内存模型 |
| C++14 | 2014 | C++11 的补全与打磨（泛型 lambda、返回推导、变量模板） |
| C++17 | 2017 | 工程实用主义：结构化绑定、optional/variant/string_view、折叠表达式、并行算法 |
| C++20 | 2020 | 「四大件」：concepts、ranges、coroutines、modules，外加三路比较与 calendar |
| C++23 | 2024 | 标准库大补给：expected、mdspan、print、deducing this、stacktrace |
| C++26 | 2026 | 编译期革命：反射、contracts、std::execution、SIMD/线性代数、erroneous behavior |

## C++98/03 —— 地基

- 类与 RAII、异常安全基础（basic/strong/nothrow guarantee）
- STL 六大件：容器、迭代器、算法、函数对象、适配器、分配器
- 模板：特化、偏特化、SFINAE、模板元编程（TMP）雏形
- C++03 修 bug：`std::vector` 存量不再失效引用的价值确立（技术上仍实现定义，实践上被依赖）

## C++11 —— 现代 C++ 起点

**语言**：auto 类型推导 / 移动语义与右值引用（`&&`、`std::move`、`std::forward`、完美转发）/ lambda 表达式 / `nullptr` / 范围 for / 委托构造与继承构造 / `override`/`final` / `constexpr` 函数 / `enum class` / 可变参数模板（`...` 模板参数包）/ `static_assert` / 别名模板（`using`）/ 统一初始化 `{}` 与初始化列表 / `default`/`delete` 函数 / `noexcept` / 尾置返回类型 / 字面量改进（`0b`、原生字符串 `R"()"`、数字分隔符 `'`）

**库**：`unique_ptr`/`shared_ptr`/`weak_ptr` / `std::thread`、`mutex`、`condition_variable`、`atomic` 与内存模型 / `std::future`/`promise`/`packaged_task`/`async` / `unordered_map` 等哈希容器 / `array`/`forward_list` / `function`/`bind` / `tuple` / `chrono` / 正则 `regex` / 随机数库 / `begin`/`end` 自由函数

## C++14 —— 打磨

- 泛型 lambda（参数 `auto`）/ lambda 初始化捕获 `[x = std::move(v)]`
- 函数返回类型推导（`auto f()`）/ `decltype(auto)`
- 变量模板 / `constexpr` 放宽（循环、分支可写）
- `std::make_unique` / `shared_timed_mutex`（读写锁）/ `integer_sequence` / 二进制字面量（其实是 14 的语言特性）/ `exchange`/`quoted`
- 弃用 `std::random_shuffle`

## C++17 —— 工程实用主义

**语言**：结构化绑定 `auto [a, b] = f();` / `if constexpr`（编译期分支，SFINAE 大杀器）/ 折叠表达式 / `if`/`switch` 带初始化器 / 内联变量 / `constexpr` lambda / CTAD 类模板实参推导（`std::vector v{1,2,3};`）/ 保证的复制消除（RVO 不再是可选优化）/ `[[nodiscard]]`/`[[maybe_unused]]`/`[[fallthrough]]` / 十六进制浮点字面量

**库**：`optional` / `variant` + `visit` / `string_view` / `any` / `filesystem` / 并行算法（`execution::par`）/ `apply`/`invoke` / `shared_mutex` / `scoped_lock`（多锁死锁免疫）/ `charconv`（零分配高性能转换）/ 数学特殊函数 / `type_traits` 扩充（`is_invocable` 等）

## C++20 —— 四大件时代

**语言**：**concepts**（`requires` 子句、约束、重载按约束排序）/ **coroutines**（`co_await`/`co_return`/`co_yield`，语言机制，库设施在后续标准）/ **modules**（`export module`）/ **ranges**（`std::ranges::` 算法、view 管道 `|`）/ 三路比较 `<=>` 与 `==` 重写规则 / `constinit`/`consteval` / lambda 模板参数化 / 指定初始化器 `.field = v` / `[[likely]]`/`[[unlikely]]` / 协程相关 `[[no_unique_address]]` / `char8_t`

**库**：`std::span` / `std::format` / `std::ranges`（views::filter/transform/take/drop/iota…）/ `jthread` 与 `stop_token`（协作取消）/ `std::atomic` 扩展（`wait/notify`、浮点、`shared_ptr` 原子化）/ `std::barrier`/`latch`/`counting_semaphore` / `std::source_location` / calendar 与时区（`chrono` 大扩）/ `starts_with`/`ends_with` / `std::bit_cast`/`midpoint`/`lerp` / `syncstream` / `std::assume_aligned`

**坑提示**：modules 与 CMake/各编译器集成至今仍在磨合；coroutines 只是机制，写生产协程库要慎重（见 modern-cpp-core.md）。

## C++23 —— 标准库大补给

**语言**：`if consteval` / deducing this（显式对象参数，`auto&& self`）/ `static operator()`/`operator[]` / lambda 属性 / `auto(x)` 显式_decay 拷贝 / 多维下标运算符（真 `m[i, j]`）/ `[[assume]]` / 标签后缀 `operator++(int)` 不再需要伪参数

**库**：**`std::expected<T, E>`**（错误处理新宠）/ `std::mdspan`（多维数组视图）/ `std::print`/`println`（格式化打印直出）/ `std::stacktrace` / `std::generator`（协程生成器）/ `std::flat_map`/`flat_set`（适配器）/ `views::zip`/`chunk`/`slide`/`join_with`/`repeat` / `std::move_only_function` / `std::string::contains` / `std::to_underlying` / `std::unreachable` / `std::byteswap` / `std::out_ptr`/`inout_ptr`

## C++26 —— 编译期革命（已定稿，编译器支持参差）

> GCC 16.1 覆盖大部分；Clang 17+/部分、MSVC 部分特性。生产环境按需启用，逐特性核对支持度。

**语言**：
- **编译期反射**：`^^` 运算符（非 TS 的 `reflexpr`）+ `std::meta` 命名空间；配合 annotations（注解元数据）
- **contracts**：`contract_assert`、`pre`/`post` 契约关键字、`<contracts>` 头
- pack indexing（`T...[0]`）/ **expansion statements**（`template for` 编译期循环遍历包/tuple）
- placeholder `_`（无名占位变量）/ `= delete("reason")` 带理由删除
- **erroneous behavior**：未初始化读从 UB 降级为「错误行为」（实现须诊断或定义行为）+ `[[indeterminate]]` 属性
- constexpr 大扩：结构化绑定、placement new、`void*` 转换、容器与适配器、虚继承、常量求值中抛异常
- `#embed`（二进制资源嵌入）+ `__has_embed` / unevaluated strings / 结构化绑定引入包、可作条件、可带属性
- 禁止返回临时引用 / 删除不完整类型指针 ill-formed / 弃用 `memory_order::consume` 与 `[[carries_dependency]]`
- 注：trivially_relocatable_if_eligible / replaceable_if_eligible **已从 C++26 移除**（实现 bug，推迟到后续标准）——别在资料里写错

**库**：
- **`std::execution`**（`<execution>`）：sender/receiver 结构化并发模型，`std::execution::task`（源自 Nvidia stdexec）
- 新头：`<hazard_pointer>`、`<rcu>`、`<simd>`（数据并行）、`<linalg>`（BLAS 线代接口）、`<hive>`（元素复用容器，源自 plf::colony）、`<inplace_vector>`（定容连续数组）、`<debugging>`、`<text_encoding>`
- `std::copyable_function` / `std::is_within_lifetime` / `std::submdspan` / 文件流 native handles
- `views::concat` / string+string_view 拼接 / `std::println` 空行 / `ranges::generate_random` / chrono 哈希 / complex 的 tuple 协议
- 饱和算术（`std::add_sat`/`div_sat`…）/ 2022 SI 前缀（quecto/ronto/ronna/quetta）/ bounds-hardened 标准库（越界检查增强）

## 升级路线建议（高频问题）

| 团队现状 | 建议 |
|---|---|
| C++11/14 | 直接跳 C++17：成本最低收益最大（结构化绑定、optional/variant/string_view、if constexpr 立竿见影） |
| C++17 | C++20 按特性渐进：先用 `std::format`/`span`/concepts/`jthread`，modules 与协程库等生态再熟一点 |
| C++20 | C++23 安心跳（expected、print、mdspan 均为纯增益）；C++26 特性逐个核对编译器支持后启用（反射/contracts 早期尝鲜可选） |
| 新项目 | 工具链允许就 C++20 起步（GCC 12+/Clang 15+/MSVC v143 均稳），关键依赖确认 C++23 支持再上调 |

编译器支持速查（2026-09）：C++20 —— GCC 10+/12 稳、Clang 14+、MSVC 19.29+/v143；C++23 —— GCC 13/14、Clang 17+、MSVC 19.34+ 渐全；C++26 —— GCC 15（部分）/16.1（大部分）、Clang 17~19（部分）、MSVC 部分（详见 cppreference 编译器支持表）。
