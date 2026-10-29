# C/C++ Super Expert / C/C++ 超级专家

简体中文 | [English](README.en.md)

职业头衔：**首席 C/C++ 技术顾问**（Principal C/C++ Technical Consultant）

从 C89 / C++98 一路精通到 C23 / C++26 的资深技术顾问：两条标准线全景、现代惯用法、C/C++ 互操作、架构与并发性能、工具链排错、嵌入式与系统级 C，深度掌握 Qt、Boost、Asio、gRPC、Abseil 等主流框架，并对 Aria 系列、Mira 网络库、SQLiteCpp 提供工程级技术支持。

## 类型

Agent 型（单个 AI 专家）

## 功能

- **C 语言全线**：C89/90 → C99 → C11 → C17 → C23 → C2y 草案每代特性与迁移坑（GCC 15 默认 `gnu23`、`f()` 语义变化、新关键字冲突）；C 惯用法（不透明指针、`goto cleanup`、`_Generic`）、C 的 UB 速查、嵌入式要点、MISRA C / CERT C。
- **C++ 全标准**：C++98/03 → 11 → 14 → 17 → 20 → 23 → 26 每代语言与库特性、演进动机、迁移路径与编译器支持现状。
- **C/C++ 互操作**：`extern "C"` 边界、ABI 设计、两种语言的语义差异陷阱、异常不穿越 C 栈帧。
- **并发与性能、工具链**：原子与内存序、协程与 `std::execution`、Sanitizers 与 profiling；GCC / Clang / MSVC / AppleClang 差异、CMake、Conan / vcpkg。
- **Aria 系列**：Aria 3.x C++23 响应式 MVVM（响应式核心、绑定引擎、五平台适配器、协程异步、编号契约、1.x→2.0→3.0 迁移、ForgeDeps 依赖模型），以及 AriaTools、AriaAgent、AriaRead、AriaTutorial 的架构与参考用法。
- **Mira 网络库**：C++23 协程原生、kqueue/epoll/IOCP 完成式 I/O，TCP/UDP/TLS/WebSocket/HTTP1/H2/QUIC/H3/SOCKS5/DoH/MQTT 的组合、取消与截止时间、平台边界与高频坑。
- **SQLiteCpp**：C++23 纯头文件 SQLite 封装的流式绑定、语句缓存、RAII 事务、结构化映射、多线程规则与性能取舍；与同名 SRombauts/SQLiteCpp 的区分；SQLite 通用最佳实践。
- **框架与高频坑**：Qt 6（含 6.12 LTS 版本动态）、Boost·Asio、gRPC、Abseil / Folly、Eigen、fmt / spdlog；悬垂引用、迭代器失效、ODR、静态初始化顺序、代码评审清单。

内置知识库 `skills/cpp-mastery/`（十份参考文档）：

| 文档 | 内容 |
|---|---|
| `c-language.md` | C 标准演进、C23 / C2y、编译器与构建、惯用法、UB、互操作、嵌入式 |
| `standards-evolution.md` | C++ 各标准特性与升级路线 |
| `modern-cpp-core.md` | 现代 C++ 语言机制 |
| `concurrency-performance.md` | 并发与性能 |
| `toolchain-debugging.md` | 工具链与调试 |
| `frameworks-qt-aria-more.md` | 主流框架速查与选型总表 |
| `aria-ecosystem.md` | Aria 框架与 Aria 系列应用 |
| `mira-networking.md` | Mira 网络库 |
| `sqlitecpp.md` | SQLiteCpp 与 SQLite 实践 |
| `faq-pitfalls.md` | 高频坑与最佳实践 |

自研技术栈事实快照日期：2026-10-29（Aria 3.2.6、Mira 1.2.1、SQLiteCpp 1.2.1、AriaTools 1.1.0、AriaAgent 0.2.0、AriaRead 0.3.1、AriaTutorial 0.2.0）。专家在可联网时会先核对 GitHub Release / CHANGELOG 再回答版本类问题。

## 使用示例

- 梳理 C23 与 C++26 的核心新特性，并给老项目一份升级路线
- 用 Aria + Mira 设计一个 C++23 跨端应用，给出架构与关键代码
- SQLiteCpp 的批量写入、事务和多线程共享怎么做最稳？

## 版本记录

- **1.1.0**：花名改为「C/C++ 超级专家」，职业头衔改为「首席 C/C++ 技术顾问」；新增 C 语言全线支持；新增 Aria 系列、Mira 网络库、SQLiteCpp 三份专门知识库；修正旧 Aria 信息（1.2.0 / C++20 → 3.2.6 / C++23）与 Qt 版本线（6.12 LTS）。
- **1.0.1**：上一版（首席 C++ 架构顾问：C++98~C++26 全标准 + Qt / Aria / Boost 等框架）。

## 头像

头像位于 `avatars/expert.png`。如需替换为自定义头像，要求：
- 格式：PNG（推荐）或 JPG
- 尺寸：512×512 px
- 大小：单张不超过 500KB

## 安装

- **本地使用**：把整个目录放进 WorkBuddy 本地专家目录（`<WorkBuddy 配置目录>/plugins/marketplaces/my-experts/plugins/`），由专家管理流程完成校验与注册后，即可在【专家中心 → 我的专家】中使用。
- **开放平台**：把本仓库打成 zip（顶层目录为 `cpp-super-expert/`）后在 WorkBuddy 开放平台提交。

```bash
git clone https://github.com/dqsjqian/cpp-super-expert.git
zip -r cpp-super-expert.zip cpp-super-expert -x "cpp-super-expert/.git/*" "cpp-super-expert/.gitignore"
```

专家包格式遵循 WorkBuddy 专家开发规范：`plugin.json` 为配置文件（位于规范要求的隐藏配置目录），`agents/` 为专家提示词，`skills/cpp-mastery/` 为知识库，`avatars/` 为头像。

## 许可证

[MIT](LICENSE) © 2026 dqsjqian
