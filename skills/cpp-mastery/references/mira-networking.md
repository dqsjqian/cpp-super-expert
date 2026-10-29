# Mira —— C++23 协程网络库

> 事实快照：2026-10-29，Mira 1.2.1（github.com/dqsjqian/Mira，MIT）。回答版本、依赖、平台验证范围前，有联网工具就先查 Releases / CHANGELOG / README；README 中「主线，尚未单独发布」的能力要与已发布版本区分说明。

## 一、定位

**传输为基，协议其上**：一套完成式（completion-based）I/O 接口统一 kqueue（macOS/iOS）、epoll（Linux/Android）、IOCP（Windows），协议只依赖 `AsyncStream` 流契约，不直接认识套接字。覆盖 TCP / UDP / TLS / WebSocket / HTTP/1.1 / HTTP/2 / QUIC / HTTP/3 / SOCKS5 / DNS·DoH / MQTT。

| 设计抉择 | 含义 |
|---|---|
| 网络库，不是兼容层 | 不以兼容既有 HTTP 库或宿主框架为目标 |
| 等待完成，而非等待就绪 | `co_await read_some(buffer)` 直接返回读取结果，就绪/完成差异留在后端 |
| 组合，而非绑定 | HTTP 不依赖 OpenSSL，TLS 不硬编码 HTTP；运行时可组合 HTTP → TLS → TCP |
| 错误与边界说清楚 | `Mira::Result<T>` = `std::expected<T, std::error_code>`；取消、截止时间、资源上限是一等设计 |

Aria 3.x 的 HTTP 适配器（REST + SSE）、AriaAgent 的流式 LLM 客户端、AriaRead 的 Web 服务都建在 Mira 上。

## 二、模块与 CMake 组件

| 模块 | 职责 | 额外依赖 |
|---|---|---|
| `Mira::core` | `Task`、`TaskScope`、`Result`、`AsyncStream`、执行器、`EventLoop`、`LoopGroup`、`BoundedExecutor`、`ResourceBudget` | 无 |
| `Mira::transport` | TCP/UDP、本地流（POSIX Unix-domain）、有界系统解析器、交错候选 `tcp::dial`、`tcp::serve` 受管服务生命周期 | 无 |
| `Mira::http` | HTTP/1 解析/序列化、单连接服务、流式请求/响应、`Expect: 100-continue` 双工 | 无（只认流契约） |
| `Mira::tls` | 同 loop 全双工 TLS 流、证书与主机名验证、mTLS、多 ALPN、SNI 多证书 | OpenSSL 3+（`MIRA_ENABLE_TLS=ON`） |
| `Mira::client` / `client_tls` | HTTP/1 / HTTPS 每 origin 有界连接池、SOCKS5 拨号组合 | `client_tls` 需 TLS |
| `Mira::http2` | nghttp2 会话、多流、Extended CONNECT | nghttp2（`MIRA_ENABLE_HTTP2=ON`） |
| `Mira::quic` / `http3` | QUIC v1、迁移、会话恢复、0-RTT、QPACK、Retry | ngtcp2 + nghttp3 + OpenSSL 3.5+（`MIRA_ENABLE_HTTP3=ON`） |
| `Mira::ws` / `crypto` | WebSocket/WSS、子协议、RFC 7692 压缩 | OpenSSL Crypto + zlib（`MIRA_ENABLE_WEBSOCKET=ON`） |
| `Mira::socks` / `dns` / `mqtt` | SOCKS5（RFC 1928/1929）、DNS 报文与 DoH（RFC 8484）、MQTT 3.1.1/5.0 | 无（TLS 由调用方组合） |

基础 core / transport / HTTP/1 构建零第三方依赖；可选协议依赖版本与 SHA256 固定在 `tools/protocol-dependencies.json`，默认不自动联网下载，由 ForgeDeps + ForgeRecipes 构建。

## 三、核心概念与代码

**TCP echo —— 整个库的世界观**
```cpp
Mira::Task<Mira::Result<void>>
echo_tcp(Mira::transport::tcp::Socket& socket) {
    std::array<std::byte, 4096> buffer{};
    for (;;) {
        auto read = co_await socket.read_some(buffer);
        if (!read) {
            if (read.error() == Mira::Errc::eof) co_return Mira::Result<void>{};
            co_return Mira::fail(read.error());
        }
        auto written = co_await Mira::write_all(
            socket, std::span<const std::byte>{buffer}.first(*read));
        if (!written) co_return Mira::fail(written.error());
    }
}
```

**HTTP 处理函数对流类型泛型，换 TLS 不改业务代码**
```cpp
template<Mira::AsyncStream Stream>
Mira::Task<Mira::Result<void>> hello(const Mira::http::Request&,
                                     Mira::http::ResponseWriter<Stream>& writer,
                                     std::span<const std::byte>) {
    Mira::http::Response response;
    response.status = 200;
    response.headers.append("Content-Type", "text/plain; charset=utf-8");
    std::string_view body = "hello, Mira\n";
    co_return co_await writer.send(response,
        {reinterpret_cast<const std::byte*>(body.data()), body.size()});
}
// co_await Mira::http::serve_connection(socket, &hello<tcp::Socket>);
// co_await Mira::http::serve_connection(tls_stream, &hello<tls::Stream<tcp::Socket>>);
```

**TaskScope 结构化并发**
```cpp
Mira::Task<int> count_after_delay(Mira::EventLoop& loop) {
    int count = 0;
    Mira::TaskScope scope;
    scope.spawn(delayed_increment(loop, scope.get_stop_token(), count));
    co_await scope.join();     // 等所有子任务结束，count 不会悬空
    co_return count;
}
```
- `join()` 只能调用一次，调用即关闭接纳；首个子任务异常触发 `request_stop()`，收齐后重抛。
- **用过但未 join 的 scope 析构会 `std::terminate()`**（fail-fast，不做隐式清理）。
- await 空 `Task` 抛 `std::logic_error`。

**单操作取消与截止时间（`OperationOptions` 贯穿 EventLoop → TCP → TLS → HTTP）**
```cpp
co_return co_await loop.read(handle, into,
    {.stop = std::move(stop), .deadline = Mira::EventLoop::Clock::now() + 5s});
```
提交时已 stop → `Errc::cancelled`；已过期 → `Errc::timed_out`；两者同时命中取 `cancelled`；同批既有真实完成又有取消，真实完成优先。deadline 是**绝对时间点**，不是时长。`EventLoop::stop()` 只请求 `run()` 返回，逐操作取消是 `OperationOptions` 的职责。

**TLS**
```cpp
auto server_ctx = Mira::tls::Context::server({
    .cert_file = "server.pem", .key_file = "server-key.pem",
    .client_ca_file = "ca.pem",   // 非空即强制 mTLS
    .min_version = "1.2",
});
auto client_ctx = Mira::tls::Context::client({.ca_file = "ca.pem"});
// tls::Stream<T>::create(loop, transport, ctx, "host") → co_await stream.handshake() → 读写
```
没有关闭验证的开关；TLS 1.0/1.1 永远拒绝；**ALPN 协商后必须自己检查 `negotiated_protocol()` 再选 H1/H2**，库不自动切换。

**HTTP 服务端两个时长**：`ServerOptions::idle_timeout`（请求之间空等）与 `request_timeout`（首字节到响应写完），默认各 60 秒，设 0 关闭；每个 keep-alive 请求重新计算预算。空等超时视为正常结束（返回成功），请求中超时返回 `Errc::timed_out` 并关连接，不发 408。

## 四、构建与接入

工具链：CMake 3.21+、C++23；CI 基线 GCC 16+ / Clang 23+（Linux）/ Xcode 27 AppleClang / MSVC v145（VS 2026）；Android 需 **NDK r30+**（NDK 27/28 的 libc++ 关闭了 `std::stop_token`）。GCC 13 协程优化器有已知 ICE，clang-18 及更早因 `__cpp_concepts` 宏过旧导致 libstdc++ 隐藏 `std::expected`。

```bash
cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug && cmake --build build/debug -j
ctest --test-dir build/debug --output-on-failure
cmake -S . -B build/tls -DMIRA_ENABLE_TLS=ON            # TLS
python3 scripts/build.py --build-dir build/protocols --tls --http2 --http3   # H2/H3（需 OpenSSL 3.5+）
```

```cmake
# 源码方式（已校验并解压的发布包）
set(MIRA_BUILD_TESTS OFF)
set(MIRA_BUILD_EXAMPLES OFF)
add_subdirectory(vendor/Mira)
target_link_libraries(my_app PRIVATE Mira::transport Mira::http)   # TLS 再加 Mira::tls

# 安装方式
find_package(Mira REQUIRED COMPONENTS core transport http tls)
```
源码兼容策略不承诺稳定二进制 ABI：升级 Mira 后消费方要重新编译。生产 OpenSSL 应使用含安全修复的版本（截至 2026-09-30：3.5 LTS 为 3.5.9、3.6 分支为 3.6.5）；1.2.1 已支持 OpenSSL 4 证书 API，同时保留 OpenSSL 3.5 的 QUIC 支持。

自带成对示例（终端 A 起 server，终端 B 跑 client）：`mira_echo_server` / `mira_tcp_client`、`mira_udp_server` / `mira_udp_client`、`mira_hello_world_server` / `mira_http1_client`、`mira_h2_prior_knowledge_server` / `mira_h2_client`、`mira_h3_server` / `mira_h3_client`，以及 ws、sse、socks5、doh、mqtt 示例。

## 五、平台验证范围（如实表述）

| 平台 | 后端 | 证据 |
|---|---|---|
| macOS | kqueue | 基础/TLS/H2/H3 运行测试、ASan/UBSan、独立 HTTP/3 curl 与 Retry 互操作 |
| Linux | epoll | GCC/Clang 运行测试、GCC ASan/UBSan、独立 HTTP/3 互操作 |
| Windows | IOCP | MSVC/MinGW 原生基础/TLS/H2/H3 测试 |
| iOS / Android | kqueue / epoll | arm64 模拟器运行通过；**无真机运行证据** |

CI 另有协议模糊测试、MQTT/mosquitto 互操作、Autobahn WebSocket 全量（含压缩）测试。所有吞吐数字都是 loopback 证据，**不是跨库排名、不是公网性能**。

## 六、明确的边界（别替它吹）

- 不做 io_uring / splice / MSG_ZEROCOPY（按真实瓶颈再选）；gRPC / Redis / WebRTC 属于生态层，不在库内。
- Windows：Unix-domain socket 工厂明确返回 `not_supported`；UDP ancillary（接收元数据/逐包选项）明确不支持，不静默降级。
- HTTP/1 客户端池默认不自动重放业务请求；`tcp::connect_with_retry` 只重试建连。
- 内置 0-RTT 反重放存储 `MemoryReplayStore` 只在同进程有效，不跨进程/重启；QUIC Retry 不保证 token 一次性使用，listener 不是抗洪泛系统。
- 客户端不记忆服务端 SETTINGS、原生系统信任（Keychain/Windows 证书库）仍未实现，默认用 OpenSSL trust。

## 七、高频坑

| 现象 | 原因与对策 |
|---|---|
| 程序在作用域结束时 `terminate` | `TaskScope` 用过但没 `co_await scope.join()` |
| 取消一次读之后 TLS 连接彻底不可用 | 设计如此：TLS 请求取消或超时会令整个会话失效（不重放密文）；需要可取消的读写要接受重建连接 |
| IOCP 上取消读后数据「丢了」 | 被取消的读可能已丢弃内核搬运的字节，该连接不能复用 |
| H2 连上了却按 H1 解析 | 没检查 `negotiated_protocol()`；库不自动切协议 |
| 跨线程使用 socket 出错 | `LoopGroup` 每个 worker 有独立线程亲和的 loop，socket 必须在所属 worker 创建和使用，不迁移 |
| `try_post` 返回 `would_block` | 应用准入走有界投递，满额即背压；`EventLoop::post` 是可靠 continuation 通道，别拿来做业务排队 |
| Android 编译报 `stop_token` 不存在 | NDK 低于 r30 |
| 0-RTT 请求被执行两次 | 0-RTT 天生可重放：只对幂等方法启用 `EarlyDataPolicy::replay_safe`，服务端对带 `early_data` 标记的请求自行判断，不能执行两次就答 425 |

## 八、选型对照（中立）

| 需求 | 建议 |
|---|---|
| C++23 新项目，要协程原生、取消/截止时间/资源上限一等公民，且需要 H2/H3/QUIC、MQTT、DoH 等多协议 | Mira |
| 需要 C++11/14/17 兼容、成熟生态与海量资料 | Boost.Asio / standalone Asio（+ Boost.Beast 做 HTTP/WebSocket） |
| C 项目或需要跨语言绑定的事件循环 | libuv |
| 需要成熟的通用传输客户端（HTTP/FTP/SMTP 等多协议、C API） | libcurl |
| 头文件即用的小型 HTTP 服务/客户端，阻塞模型可接受 | cpp-httplib |

选型时说清 Mira 的门槛：C++23 + 新工具链（GCC 16 / Clang 23 / VS2026 / NDK r30），移动端只有模拟器证据。
