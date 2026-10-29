# SQLiteCpp —— C++23 SQLite 封装（dqsjqian/SQLiteCpp）

> 事实快照：2026-10-29，SQLiteCpp 1.2.1（github.com/dqsjqian/SQLiteCpp，MIT），内置 SQLite 3.54.1。版本与性能数字以仓库 README / 标签为准。

## 零、先确认是哪个「SQLiteCpp」

两个同名库 API 完全不同，回答前先看用户代码或依赖来源：

| 特征 | dqsjqian/SQLiteCpp（本文主角） | SRombauts/SQLiteCpp（SQLiteC++） |
|---|---|---|
| 命名空间与入口 | `sqlite::database`，`#include <SQLiteCpp.h>` | `SQLite::Database` / `SQLite::Statement`，`#include <SQLiteCpp/SQLiteCpp.h>` |
| 风格 | 流式：`db << sql << arg >> out` | 面向对象：`query.bind(1, v)`、`while (query.executeStep())`、`getColumn(i)` |
| 标准 | C++23（低于直接 `#error`） | 4.x 主线 C++17；3.x 维护线 C++11 |
| 形态 | 纯头文件，内置 SQLite amalgamation | 编译型静态库，常配系统 libsqlite3 或自带源码 |
| 渊源 | 延续 sqlite_modern_cpp 的流式写法；Aria 系列的 AriaRead 已从 sqlite_modern_cpp 切到它（ForgeRecipes 中依赖名仍沿用 `sqlite_modern_cpp`） | 独立发展的老牌封装 |

用户只说「SQLiteCpp」又无上下文时，先问一句或按其代码中的类名判断；SRombauts 版的问题按其官方 README 与 Doxygen 回答。

## 一、定位与要求

C++23 SQLite 封装：流式参数绑定、强类型结果、结构化行映射、预编译语句缓存；纯头文件、自闭环、只认一个 SQLite 版本。
- 头文件要求 C++23 与 SQLite ≥ 3.38；仓库 `vendor/sqlite` 内置官方 3.54.1 amalgamation（已对照 sqlite.org 校验 SHA3-256），克隆即可构建，无需包管理器或联网。
- CI：GCC 16、Clang 23（libstdc++/libc++）、AppleClang、MSVC（VS 2026）、clang-cl 23、MinGW；Linux/Windows 含 ARM64 原生运行；iOS、Android（NDK r30）仅构建。每次推送跑 ASan+UBSan、TSan、clang-tidy、覆盖率、60 秒 libFuzzer、逐头文件单独编译。

## 二、API 总览

| 领域 | 入口 |
|---|---|
| 连接 | `database`、`sqlite_config`（`flags`、`zVfs`、`encoding`、`statement_cache_size`=32、`busy_timeout_ms`）、`config_flag`、`busy_timeout`、`configure`、`interrupt`、`progress_handler`、`load_extension` |
| 语句 | `operator<<`、`operator>>`、`execute`、`execute_many`、`try_execute`、`row.as<Ts...>()`、`column_name` / `column_declared_type` / `column_table_name` |
| 事务 | `transaction`（`deferred` / `immediate` / `exclusive`）、`savepoint` |
| 结果 | `select<Record>(...)`、`as<Ts...>()`、`pragma`、`explain` |
| 数据 | `blob`（增量 I/O）、`no_copy`（`SQLITE_STATIC` 零拷贝绑定）、`serialize` / `deserialize`、`backup` |
| 函数 | `define`（标量 / 聚合）、`define_window` |
| 文件与诊断 | `read_header`、`has_sqlite_header`、`cache_used`、`schema_used` |
| 钩子 | `on_update`、`on_commit`、`on_rollback` |
| 字面量 | `"SELECT 1"_sql` |
| 加密 | `sqlcipher.h` 中的 `sqlcipher_database`（`set_key` / `rekey`），需改链 SQLCipher 构建 |

## 三、代码速查

```cpp
#include <SQLiteCpp.h>

sqlite::database db("app.db");                 // 或 ":memory:"
db << "CREATE TABLE IF NOT EXISTS item (id INTEGER PRIMARY KEY, value TEXT)";
db << "INSERT INTO item (value) VALUES (?)" << "hello";

std::string v;
db << "SELECT value FROM item WHERE id = ?" << 1 >> v;   // 单行：0 行抛 no_rows，多行抛 more_rows
```

**批量写入 + RAII 事务**
```cpp
const std::vector<std::tuple<int, std::string>> rows{{1, "one"}, {2, "two"}};
sqlite::transaction tx(db, sqlite::transaction_behavior::immediate);
db.execute_many("INSERT INTO item VALUES (?, ?)", rows);   // 编译一次，执行 N 次
tx.commit();                                               // 不 commit，离开作用域自动回滚
```

**结构化读取**
```cpp
struct item_record { int id = 0; std::string value; };
const auto items = db.select<item_record>(
    "SELECT id, value FROM item ORDER BY id", &item_record::id, &item_record::value);

for (auto&& row : db << "SELECT id, value FROM item") {
    auto [id, text] = row.as<int, std::string>();
}
db << "SELECT id, value FROM item" >> [](int id, const std::string& text) { /* 逐行回调 */ };
```

**复用语句**
```cpp
auto insert = db << "INSERT INTO item VALUES (?, ?)";
for (const auto& [id, text] : rows) {
    insert << id << text;
    insert.execute();          // 执行后自动重置，下一轮从 ?1 开始；本轮没绑定的参数一律为 NULL
}
```

**可空列**：`std::optional<T>` 读写 NULL（如 `>> [&](long long id, std::optional<int> val) { ... }`）。

**无异常路径**
```cpp
if (auto r = db.try_execute("UPDATE item SET value = ? WHERE id = ?", std::string("x"), 1); !r) {
    log_error(r.code, r.error);        // execution_result：changes / code / error，noexcept
}
```
注意：`try_execute` 的参数受 `sqlite::bindable` 约束，**字符串字面量（数组类型）和 `const char*` 不满足**，要传 `std::string` 或 `std::string_view`；流式 `db << sql << "x"` 不受此限制。

**SQL 函数**
```cpp
db.define("my_concat", [](std::string a, std::string b) { return a + b; });   // 标量
db.define("my_count", [](int& acc, int) { ++acc; }, [](int& acc) { return acc; });  // 聚合：step + final
```

**钩子**
```cpp
db.on_update([](int action, std::string_view database, std::string_view table, sqlite3_int64 rowid) {
    // string_view 指向 SQLite 内部内存，只在回调期间有效，需要保留就拷贝
});
db.on_update(nullptr);   // 移除
```
钩子抛出的异常不会穿过 SQLite 的 C 栈帧：进度回调抛异常中断当前语句，commit 钩子抛异常把提交转为回滚。

**错误**：异常基类 `sqlite::sqlite_exception`（携带结果码、扩展码、SQL 文本、`sqlite3_error_offset` 字节偏移），按 SQLite 错误码派生具体类型，另有 `more_rows`、`no_rows`、`more_statements`（一条预编译语句只能含一条 SQL）、`invalid_utf16`、`unknown_binding`。

## 四、多线程规则

- SQLite 串行化模式（默认，内置 amalgamation 即此模式）下，一个 `sqlite::database` 可被多线程共享；语句缓存使用连接自身的互斥锁，并发 `db << ...` 安全（已在 TSan 下验证）。
- 单个语句对象（`db << sql` 的返回值）同一时刻只属于一个线程。
- 钩子和进度回调要在连接共享之前安装。
- 以 `OpenFlags::NOMUTEX` 打开的连接无互斥锁，只能单线程使用（缓存也因此零加锁开销）。

## 五、性能结论（1.2.x，Apple Silicon，:memory:，10 万次操作）

| 路径 | 相对裸 C API（同样复用预编译语句） |
|---|---|
| 逐行 `db << sql << ...`（默认缓存） | 写入 +16% |
| 持有语句逐行重新绑定 | +4% |
| `execute_many` | 持平 |
| 关闭缓存（`statement_cache_size = 0`） | +242%（反例，不是建议） |
| 单行查询 `>>` | +22%（其中约 10 个百分点是「确认只有一行」的额外 step） |
| 全表扫描（逐行回调 / `row.as` / `select<Record>`） | +1% ~ +2% |

结论：热路径用 `execute_many` 或持有语句；省内存把缓存设 1~4，别设 0；WAL 磁盘库比例基本不变。

## 六、CMake 接入

```sh
cmake -S . -B build -DBUILD_TESTING=ON
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
cmake --install build --prefix /path/to/sdk
```
```cmake
project(app LANGUAGES C CXX)                     # sqlite3.c 需要 C 编译器
find_package(SQLiteCpp CONFIG REQUIRED)
find_package(Threads REQUIRED)
add_library(sqlite3_core STATIC ${SQLiteCpp_SQLITE_DIR}/sqlite3.c)   # 安装包附带同一份 amalgamation
target_include_directories(sqlite3_core PUBLIC ${SQLiteCpp_SQLITE_DIR})
target_link_libraries(sqlite3_core PUBLIC Threads::Threads ${CMAKE_DL_LIBS})
target_link_libraries(app PRIVATE SQLiteCpp::SQLiteCpp sqlite3_core)
```
选项：`BUILD_TESTING`(ON)、`SQLITECPP_WARNINGS_AS_ERRORS`(OFF)、`SQLITECPP_COVERAGE`(OFF)、`SQLITECPP_BUILD_FUZZERS`(OFF)。链接自带 SQLite 时版本必须 ≥ 3.38，否则编译期报错；公开头文件包含 `<sqlite3.h>` 的库（如 AriaRead 的 `database.h`）要把锁定的 SQLite 设为 PUBLIC 依赖，避免消费方看到平台 SDK 里的旧 `sqlite3.h`。

## 七、SQLite 通用最佳实践（与封装无关，答题常用）

- 打开即设：`PRAGMA journal_mode=WAL;`（读写并发）、`PRAGMA synchronous=NORMAL;`（WAL 下常用折中）、`busy_timeout`、`configure(config_flag::foreign_keys, true)`（外键默认关闭）。
- 批量写入必须包事务，否则每条语句一次 fsync，慢几十到上百倍。
- 一律参数绑定，禁止拼接 SQL（注入与语句缓存双输）。
- 多线程高并发写：SQLite 同时只有一个写者；写多读少场景考虑单写线程 + 队列，读用独立连接。
- 大 BLOB 用 `blob` 增量读写，避免一次性搬进内存；`serialize` / `backup` 做快照与热备。
- 排查慢查询：`explain` 看查询计划，确认命中索引。
