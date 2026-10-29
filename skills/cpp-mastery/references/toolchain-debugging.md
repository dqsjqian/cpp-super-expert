# 工具链与调试速查（编译器 / CMake / 包管理 / Sanitizers / 排错）

## 编译器矩阵（2026-09）

| 编译器 | 当前主线 | C++20 | C++23 | C++26 | 备注 |
|---|---|---|---|---|---|
| GCC | 14 稳 / 15 / **16.1（大部分 26 特性）** | 10+ 稳 | 13+ | 15 部分 / 16 大部分 | libstdc++；`-std=c++2c`（15+） |
| Clang / LLVM | 18 / 19 / 20 | 14+ 稳 | 17+ 渐全 | 17~19 部分 | libc++/libstdc++ 双前端；反射在实验 fork（Dan Katz 版） |
| Apple Clang | Xcode 16/17 | ✅ | 大部分 | 部分 | 跟 LLVM 主线有延迟 |
| MSVC | v143（VS2022）/ v144（VS2026 方向） | ✅（/std:c++20） | 大部分（/std:c++latest） | 部分 | ABI 独树一帜；两字符集选项 `/utf-8` 记得开 |
| Intel / NVCC 等 | 跟随主链 | 部分有延迟 | | | GPU 侧依赖主编译器 |

速查：`-std=c++17/20/23/2b/2c`（GCC/Clang）；MSVC `/std:c++17/20/latest`。特性宏查支持：`__has_feature`/`__has_include`/FTM（feature-test macros，如 `__cpp_concepts`、`__cpp_lib_expected`）。

## CMake 实战要诀

```cmake
cmake_minimum_required(VERSION 3.28)          # 能上就上 3.28+：预设/导出全
project(demo LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)           # 不准编译器降级
set(CMAKE_CXX_EXTENSIONS OFF)                 # -std=c++20 而非 gnu++20
add_executable(myapp main.cpp)                # 目标必须先定义
target_compile_features(myapp PRIVATE cxx_std_20)
```
- **一切走 target**：`target_link_libraries`/`target_include_directories`/`target_compile_options`，禁用目录级 `include_directories`/`add_definitions`（全局污染）。
- `find_package(foo CONFIG REQUIRED)` + alias 目标；`FetchContent` 拉源码依赖；大项目 presets（`CMakePresets.json`）管配置矩阵。
- Debug/Release 分目录：`cmake -B build/flavors/release -DCMAKE_BUILD_TYPE=Release`（build/ 是容器别直接配）。
- 交叉编译：toolchain file（`CMAKE_TOOLCHAIN_FILE`）——iOS/Android NDK 各有官方链。
- MSVC 注意：`/utf-8`、`/permissive-`（标准一致模式）、`/Zc:preprocessor`（标准预处理器，正则/`__VA_OPT__` 需要）。

## 包管理

| 工具 | 模式 | 备注 |
|---|---|---|
| **Conan 2** | 去中心化仓 + 二进制缓存 | C++ 生态最成熟，CI 友好 |
| **vcpkg** | manifest（vcpkg.json）+ 注册表 | 微软生态，Windows 丝滑，CMake 工具链集成开箱 |
| CMake FetchContent | 源码拉取编译 | 零外部依赖，适合小库 |
| 系统 apt/brew/MSYS2 pacman | 发行版包 | 简单但版本老 |

选型：单一项目小依赖 FetchContent；多项目/大依赖 Conan 或 vcpkg 二选一定全司规范。

## Sanitizers 与调试

```bash
# GCC/Clang 一条龙（Debug 构建）
-fsanitize=address,undefined -fno-omit-frame-pointer -g   # ASan+UBSan
-fsanitize=thread                                          # TSan（与 ASan 互斥，分开跑）
```
| 工具 | 抓什么 | 开销 |
|---|---|---|
| ASan | 堆/栈越界、use-after-free、double-free、泄漏（LSan） | ~2x 慢 + 3x 内存 |
| UBSan | UB：空指针解引用、溢出、错位访问、非法转换 | ~1.2x |
| TSan | 数据竞争、死锁 | ~5-15x，多线程必跑 |
| MSan | 未初始化读 | 需全链插桩，最贵最狠 |
| Valgrind/memcheck | 老牌内存检查（不用重编） | ~10-50x 慢 |
| MSVC | `/fsanitize=address`、`/fsanitize=fuzzer` | VS2022 17.x 起可用 |

调试器：gdb/lldb（命令行两兄弟，`bt`/`frame`/`watch`/`thread apply all bt`）；core dump 分析（`ulimit -c`、`-c core` 符号分离 `objcopy --only-keep-debug`）；崩溃聚合符号化（`llvm-symbolizer`）。

静态分析：clang-tidy（规则集 `bugprone-*,performance-*,modernize-*,cppcoreguidelines-*`，**基线文件**管存量——增量严管、存量放行）、cppcheck、PVS-Studio（商业）。CI 里 `run-clang-tidy` + diff 过滤。

fuzzing：libFuzzer（`-fsanitize=fuzzer,address`）、AFL++；对解析器/协议层持续 fuzz。

## 排错剧本（高频场景）

| 症状 | 第一刀 |
|---|---|
| 段错误无栈 | ASan 重跑 / core dump + `bt` |
| 偶发崩溃/数据错乱 | TSan；检查回调跨线程、对象生命周期 |
| 符号未定义 (undefined reference) | ABI 不匹配（GCC↔Clang 的 libstdc++/libc++）、模板定义没进 TU（显式实例化或挪头文件）、链接顺序 |
| 重复定义 (multiple definition) | 头文件里非 inline 定义、ODR 违反（不同编译选项编同一头）、静态库符号冲突 |
| 静态初始化顺序灾难 | 函数内 static（Meyers singleton）、`constinit`（20） |
| 发布版崩调试版不崩 | 未初始化变量、UB 依赖、断言副作用——上 UBSan |
| 模板报错长篇 | 读**最里层用户代码**那一行；concepts 化重写根治 |
