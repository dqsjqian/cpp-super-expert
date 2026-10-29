# C/C++ Super Expert

[简体中文](README.md) | English

Profession: **Principal C/C++ Technical Consultant**

A WorkBuddy expert package for C and C++: both standards lines from C89 / C++98 through C23 / C++26, modern idioms, C/C++ interoperability, architecture, concurrency and performance, toolchains and debugging, embedded and systems C, major frameworks (Qt, Boost, Asio, gRPC, Abseil), and engineering-level support for the Aria series, the Mira networking library and SQLiteCpp.

## Type

Agent (a single AI expert).

## Capabilities

- **C, end to end**: C89/90 → C99 → C11 → C17 → C23 and the C2y draft; migration pitfalls (GCC 15 defaults to `gnu23`, the new meaning of `f()`, new keywords); C idioms (opaque pointers, `goto cleanup`, `_Generic`); undefined-behavior checklist; embedded notes; MISRA C / CERT C.
- **C++, every standard**: C++98/03 → 11 → 14 → 17 → 20 → 23 → 26 language and library features, motivation, migration paths and compiler support.
- **C/C++ interoperability**: `extern "C"` boundaries, ABI design, semantic differences between the two languages, keeping exceptions out of C stack frames.
- **Concurrency, performance and toolchains**: atomics and memory ordering, coroutines and `std::execution`, sanitizers and profiling; GCC / Clang / MSVC / AppleClang differences, CMake, Conan / vcpkg.
- **Aria series**: the Aria 3.x C++23 reactive MVVM framework (reactive core, binding engine, five platform adapters, coroutine async, numbered contracts, 1.x → 2.0 → 3.0 migration, ForgeDeps dependency model) and how to learn from AriaTools, AriaAgent, AriaRead and AriaTutorial.
- **Mira**: a C++23 coroutine-native networking library on completion-based I/O (kqueue / epoll / IOCP) — composing TCP/UDP/TLS/WebSocket/HTTP1/H2/QUIC/H3/SOCKS5/DoH/MQTT, cancellation and deadlines, platform boundaries and common pitfalls.
- **SQLiteCpp**: the header-only C++23 SQLite wrapper — stream binding, statement cache, RAII transactions, struct mapping, threading rules and performance trade-offs; how it differs from the similarly named SRombauts/SQLiteCpp; general SQLite practice.

Built-in knowledge base `skills/cpp-mastery/` (ten reference documents):

| Document | Content |
|---|---|
| `c-language.md` | C standards, C23 / C2y, compilers and builds, idioms, UB, interop, embedded |
| `standards-evolution.md` | C++ standards and upgrade paths |
| `modern-cpp-core.md` | Modern C++ language mechanisms |
| `concurrency-performance.md` | Concurrency and performance |
| `toolchain-debugging.md` | Toolchains and debugging |
| `frameworks-qt-aria-more.md` | Framework quick reference and selection table |
| `aria-ecosystem.md` | The Aria framework and the Aria application series |
| `mira-networking.md` | The Mira networking library |
| `sqlitecpp.md` | SQLiteCpp and SQLite practice |
| `faq-pitfalls.md` | Common pitfalls and best practices |

The knowledge base is written in Chinese; the expert answers in the language of the question. Version facts for the Aria series, Mira and SQLiteCpp are a snapshot dated 2026-10-29 (Aria 3.2.6, Mira 1.2.1, SQLiteCpp 1.2.1, AriaTools 1.1.0, AriaAgent 0.2.0, AriaRead 0.3.1, AriaTutorial 0.2.0); when network tools are available, the expert checks GitHub releases and changelogs before answering version questions.

## Try asking

- Walk me through the core features of C23 and C++26 and give a legacy project an upgrade path
- Design a C++23 cross-platform app with Aria and Mira: architecture and key code
- What is the most robust way to do batch writes, transactions and multithreaded sharing with SQLiteCpp?

## Installation

- **Local use**: place the whole directory in the WorkBuddy local experts directory (`<WorkBuddy config dir>/plugins/marketplaces/my-experts/plugins/`); once the expert manager validates and registers it, it appears under Expert Center → My Experts.
- **Open platform**: zip the repository (top-level directory `cpp-super-expert/`) and submit it on the WorkBuddy open platform.

```bash
git clone https://github.com/dqsjqian/cpp-super-expert.git
zip -r cpp-super-expert.zip cpp-super-expert -x "cpp-super-expert/.git/*" "cpp-super-expert/.gitignore"
```

The package follows the WorkBuddy expert specification: `plugin.json` is the configuration (in the hidden configuration directory required by the specification), `agents/` holds the expert prompt, `skills/cpp-mastery/` the knowledge base and `avatars/` the avatar.

## License

[MIT](LICENSE) © 2026 dqsjqian
