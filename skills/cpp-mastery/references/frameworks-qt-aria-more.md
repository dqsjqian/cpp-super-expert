# 框架速查（Qt 6 / Aria 系列 / Mira / SQLiteCpp / Boost·Asio / gRPC / Abseil / Eigen / fmt·spdlog）

## Qt 6 —— 跨平台 GUI/应用框架之王

**版本线（2026-10）**：最新 **Qt 6.12 LTS**（2026-09-30 发布，5 年维护；Qt Canvas Painter 转正、QML 热重载内置进引擎、HarmonyOS 成为 LTS 平台）；上一代 LTS **Qt 6.8**（含 Qt Graphs/HttpServer/GRPC 转正）；6.11 为非 LTS。注意：**Qt Charts 与 Qt Data Visualization 已废弃**，新代码用 **Qt Graphs**。

**核心机制**：
- **元对象系统**：Q_OBJECT 宏 + moc 生成元信息；信号槽是类型安全的观察者（跨线程自动队列化）；`connect` 新式指针语法（`connect(btn, &QPushButton::clicked, this, &Foo::onOk)`）+ lambda 重载；第五种重载（Qt 6.7+）允许直接 `connect(sender, &S::sig, receiverContext, lambda)`。
- **事件循环与线程亲和**：QObject 属于创建它的线程；跨线程信号自动 `QueuedConnection`（参数类型需注册 `qRegisterMetaType`）；**禁止跨线程直接操作 widget**——用信号槽或 `QMetaObject::invokeMethod` 编组。
- **模型/视图**：QAbstractItemModel 族（QListView/QTableView/QTreeView + QML ListView）；数据变更发 `dataChanged`/`beginInsertRows` 配对；大数据用 QAbstractListModel 只实现必需虚函数。
- **QML 与 C++ 交互**：`Q_PROPERTY`（READ/WRITE/NOTIFY 三件套，NOTIFY 是 QML 绑定的命）、`qmlRegisterType`/`QML_ELEMENT`、context property 与 singleton；C++ 侧暴露模型用 QAbstractListModel。
- **内存**：QObject 父子树所有权（父删子亡）；`deleteLater()` 而非直接 delete（事件循环安全）；`QPointer`（弱引用 widget）。
- **常用件**：QString（隐式共享，跨平台文本首选）、QThread（默认 QThread 对象本身不在新线程——run() 才在，惯用法 worker moveToThread）、QNetworkAccessManager、QSqlDatabase、QCborValue/QtSerialization。

**构建**：CMake `find_package(Qt6 COMPONENTS Widgets Quick REQUIRED)` + `qt_standard_project_setup()` + `qt_add_executable`；QML 模块 `qt_add_qml_module`。

## Aria 系列 / Mira / SQLiteCpp —— 详见专门文档

- **Aria**（github.com/dqsjqian/Aria，MIT，当前 3.2.6）：C++23 响应式 MVVM 框架，ViewModel 纯 C++、View 用各平台原生控件，`BindingEngine` + `IViewAdapter` 连接 Qt6 / AppKit / UIKit / JNI / HTTP 五个适配器。**3.0 起最低 C++23**，3.2.2 起工具链下限 GCC 16 / Clang 23 / Xcode 27 / VS2026；接入用 `find_package(aria 3.0 CONFIG REQUIRED)`。旧资料里的「1.2.0 / C++20 / `third_party/aria`」均已过时。完整 API、迁移、推荐决策表、AriaTools / AriaAgent / AriaRead / AriaTutorial 用法见 `aria-ecosystem.md`。
- **Mira**（当前 1.2.1）：C++23 协程网络库，kqueue / epoll / IOCP 完成式 I/O，TCP/UDP/TLS/WebSocket/HTTP1/H2/QUIC/H3/SOCKS5/DoH/MQTT，见 `mira-networking.md`。
- **SQLiteCpp**（dqsjqian 版，当前 1.2.1）：C++23 纯头文件 SQLite 封装，流式绑定 + 语句缓存，内置 SQLite 3.54.1，见 `sqlitecpp.md`（注意与同名 SRombauts/SQLiteCpp 区分）。

## Boost 与 Asio —— 准标准库与网络事实标准

- **Boost.Asio**：异步 IO 框架（proactor 模型）；`io_context` 事件循环 + `async_read/async_write` + 协程（`co_await` 直接可用）；独立版 asio（standalone，非 boost 命名空间）更轻。定时器 `steady_timer`、信号 `signal_set`。
- Boost 大件：`boost::beast`（HTTP/WebSocket，基于 Asio）、`boost::url`、`boost::json`、`boost::container`（flat_map 等先于标准）、`boost::mp11`（元编程）、`boost::describe`（轻量反射，C++26 反射前的过渡）、`boost::test`/`boost::ut` 类。
- 经验：Boost 是「标准的试验田」——unified call syntax 没成，但 optional/variant/filesystem/format 都从 Boost 毕业了；用之前查一下是否标准库已有。

## gRPC —— RPC 事实标准

- protobuf IDL + HTTP/2 多路复用；四种模式：unary / server-streaming / client-streaming / bidi。
- C++ 侧：`grpc::CompletionQueue`（异步 API，复杂）或**同步 API + 线程池**起步；`grpc::ServerBuilder` + 生成的 Stub；deadline/取消 `ClientContext`。
- 坑：默认 keepalive 不开、大消息默认 4MB 上限、channel 复用（别每请求建 channel）。

## Abseil / Folly —— 工业级基础库

- **Abseil**（Google）：`absl::StatusOr`（expected 前身）、`absl::string_view`/`span`（标准前身）、`absl::flat_hash_map`（开放寻址 + SIMD 分组探测的 Swiss Table，多数负载下快于 `std::unordered_map`，以自身基准为准）、`absl::Time`、`StrCat`/`StrFormat`、`absl::FunctionRef`（非拥有 callable 视图）。与标准同源设计，升级平滑。
- **Folly**（Meta）：`folly::Future`（支持 `thenValue` 等 continuation，`std::future` 没有 then）、`fbvector`/`fbstring`、`folly::coro`（协程库，Task/Generator/延迟执行）、`small_vector`（栈上小向量）。重，适合服务端大仓。

## Eigen / fmt / spdlog —— 三件套常客

- **Eigen**：头文件线代库，表达式模板零开销；`MatrixXd` 动态、`Matrix<double,3,3>` 固定更优；**对齐坑**：固定尺寸向量成员需 `EIGEN_MAKE_ALIGNED_OPERATOR_NEW`（3.4 起多数场景自动）；C++26 `std::linalg` 是标准化方向。
- **fmt**：`std::format` 的前世，编译时间友好；`{fmt}` 库在标准 print（23）之前的跨版本选择。
- **spdlog**：日志库标配，fmt 后端、异步 sink、轮转；`SPDLOG_ACTIVE_LEVEL` 控编译期级别。

## 框架选型速答

| 需求 | 默认答案 |
|---|---|
| 桌面 GUI（C++ 主导） | Qt 6 Widgets |
| 跨端 UI 一份代码 | Qt Quick（QML） |
| C++ 核心多端复用 + 原生 UI / 响应式 MVVM（C++23） | **Aria** |
| C++23 协程原生网络层（含 H2/H3/QUIC/MQTT） | **Mira** |
| 需兼容旧标准的高性能网络服务 | Asio（协程）+ protobuf/gRPC |
| C++23 嵌入式数据库访问 | **SQLiteCpp**（dqsjqian 版）；C++23 之前的工程用 SRombauts/SQLiteCpp（3.x 为 C++11、4.x 为 C++17）或 sqlite_orm（C++14+，AGPL 或付费 MIT 双许可，商用先看许可） |
| 基础数据结构与工具 | Abseil；服务端大仓 Folly |
| 科学计算 | Eigen（未来 `std::linalg`） |
