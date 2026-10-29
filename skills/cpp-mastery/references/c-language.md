# C 语言全景（C89/C90 → C23 → C2y 草案）

> 事实快照：2026-10。C23 正式标准为 ISO/IEC 9899:2024（`__STDC_VERSION__ == 202311L`）；下一版非正式名 C2y，按 WG14 计划约 2029 年发布（亦被称为 C29），最新工作草案为 N3886（2026-05）。草案内容在定稿前都可能变化，回答时标注「草案」。

## 一、标准演进时间线

| 标准 | `__STDC_VERSION__` | 定位 | 关键内容 |
|---|---|---|---|
| C89 / C90 | 未定义（C90 无此宏） | 第一版 ANSI/ISO C | 函数原型、`const`/`volatile`、`void*`、标准库雏形 |
| C95（AMD1） | `199409L` | 修正案 | 宽字符 `<wchar.h>`/`<wctype.h>`、二字符组 |
| C99 | `199901L` | 现代 C 起点 | `//` 注释、`inline`、`restrict`、`_Bool`/`<stdbool.h>`、`long long`、`<stdint.h>`/`<inttypes.h>`、变长数组 VLA、复合字面量、指定初始化器 `.x = 1`、柔性数组成员、`for` 内声明、`__func__`、`<complex.h>`、`snprintf` |
| C11 | `201112L` | 并发与类型安全 | `_Generic`、`_Static_assert`、`_Alignas`/`_Alignof`、`_Noreturn`、`_Atomic`/`<stdatomic.h>`、`_Thread_local`、`<threads.h>`、匿名结构/联合、`quick_exit`、Annex K（`*_s` 边界检查函数，可选）；VLA 降为可选 |
| C17 / C18 | `201710L` | 缺陷修复 | 无新特性，澄清大量 DR；保守项目的「稳定基线」 |
| C23 | `202311L` | 大幅现代化 | 见下节 |
| C2y（草案） | 草案阶段 `202xxxL` | 进行中 | 见第三节 |

## 二、C23 必知清单

**关键字与类型**
- `bool` / `true` / `false` / `nullptr` / `static_assert` / `alignas` / `alignof` / `thread_local` 成为真关键字（`<stdbool.h>` 等变为空壳）；`typedef int bool;` 直接编译失败。
- `nullptr` 与 `nullptr_t`（`<stddef.h>`）；`typeof` / `typeof_unqual`；对象的 `constexpr`（仅对象，没有 constexpr 函数）；`auto` 推导对象类型。
- `_BitInt(N)` 精确位宽整数；二进制字面量 `0b1010`；数字分隔符 `1'000'000`；`enum E : unsigned char { ... }` 固定底层类型。
- 空初始化 `= {}`（含 VLA）；有符号整数**强制二进制补码**表示（溢出仍是 UB）。

**属性**：`[[nodiscard]]`（可带原因）、`[[maybe_unused]]`、`[[deprecated]]`、`[[fallthrough]]`、`[[noreturn]]`、`[[unsequenced]]`、`[[reproducible]]`。

**语法清理**
- `int f();` 在 C23 中等价于 `int f(void);`；K&R 风格函数定义被移除。依赖旧语义的代码在 GCC 15 默认模式下会报不兼容指针类型。
- 函数定义可省略不用的参数名；标签可放在声明前和复合语句末尾。

**预处理**：`#embed`（嵌入二进制资源）、`#elifdef` / `#elifndef`、`#warning`、`__has_include`、`__has_c_attribute`、`__VA_OPT__`。

**标准库**：`<stdbit.h>`（位计数、前导/尾随零、单比特判断，以及 `__STDC_ENDIAN_NATIVE__` 等字节序宏）、`<stdckdint.h>`（`ckd_add` / `ckd_sub` / `ckd_mul` 检查溢出的算术）、`memset_explicit`（不被优化掉的清零）、`memccpy`、`strdup` / `strndup`、`timegm`；`realloc(p, 0)` 变为 UB，别再依赖它释放内存。

## 三、C2y（草案）已进工作草案的特性

以 WG14 最新工作草案为准，以下为已广泛报道并进入草案的条目：
- `_Countof` 运算符与 `<stdcountof.h>` 中的 `countof` 宏（数组元素个数，替代 `sizeof a / sizeof a[0]`）。
- `if` 声明：`if (int err = op(); err != 0) { ... }`。
- `switch` 的 case 范围：`case 1 ... 3:`（省略号两侧空格必需）。
- 具名循环：给循环/switch 加标签，`break label;` / `continue label;` 跳出多层。
- `0o` / `0O` 八进制前缀与 `\o{}`、`\x{}`、`\u{}` 定界转义；旧式隐式八进制字面量走向过时。
- `_Generic` 接受类型操作数；`__COUNTER__` 标准化；`strnlen` / `wcsnlen` 进入标准库；复数支持 `++`/`--`。
- `defer`：先以技术规范（TS 25755）推进，GCC/Clang 有实验实现；是否已并入正式草案以最新 N 号文档为准，生产代码暂用 `goto cleanup` 或编译器 `__attribute__((cleanup))`。

GCC 15 起用 `-std=c2y` / `-std=gnu2y` 可试用其中多项（if 声明、case 范围、具名循环、`_Generic` 类型操作数等）；Clang 19 起陆续跟进。

## 四、编译器与构建

| 项 | 要点 |
|---|---|
| GCC | **GCC 15 起 C 默认 `-std=gnu23`**（此前 gnu17）；`#embed`、`[[unsequenced]]` 等在 GCC 15 落地。老代码升级编译器后报 `bool` typedef、`f()` 原型冲突，多半是这个原因 |
| Clang | 支持 `-std=c23` / `c2y`；默认标准随版本变化，跨编译器项目一律显式指定 |
| MSVC | `/std:c11`、`/std:c17`（VS 2019 16.8 起）、`/std:clatest`；**不支持 VLA**；C11 `<threads.h>` 在 VS 2022 17.8 起提供；`_Atomic` 支持较晚且曾为实验性 |
| Apple 平台 | libc 未提供 `<threads.h>`，可移植线程代码用 pthreads 或自行封装 |
| CMake | `set(CMAKE_C_STANDARD 23)` + `set(CMAKE_C_STANDARD_REQUIRED ON)` + `set(CMAKE_C_EXTENSIONS OFF)`（`17`/`23` 取值自 CMake 3.21 起）；混合工程 `project(x LANGUAGES C CXX)` |

推荐警告集：`-Wall -Wextra -Wpedantic -Wshadow -Wconversion -Wstrict-prototypes -Wmissing-prototypes -Werror=implicit-function-declaration`；静态分析：GCC `-fanalyzer`、clang-tidy、cppcheck；规范：MISRA C:2023、SEI CERT C。

## 五、惯用法

**不透明指针（C 的封装与 ABI 稳定）**
```c
/* widget.h */
typedef struct widget widget;                 /* 不完整类型，布局对外不可见 */
widget *widget_create(int size);
int     widget_resize(widget *w, int size);   /* 返回 0 成功，负数为错误码 */
void    widget_destroy(widget *w);            /* 接受 NULL，与 free 语义一致 */
```

**单出口清理（C 没有析构函数）**
```c
int load(const char *path) {
    int rc = -1;
    FILE *f = NULL;
    char *buf = NULL;
    if (!(f = fopen(path, "rb"))) goto out;
    if (!(buf = malloc(4096)))    goto out;
    /* ... */
    rc = 0;
out:
    free(buf);
    if (f) fclose(f);
    return rc;
}
```
GCC/Clang 扩展 `__attribute__((cleanup(fn)))` 可实现作用域自动清理（systemd、GLib 的 `g_autoptr` 即基于它），非标准，MSVC 不支持。

**错误处理**：返回值错误码 + 输出参数；`errno` 只在函数文档声明会设置时读取，且调用前清零；溢出检查用 C23 `ckd_*`，旧标准用 GCC/Clang `__builtin_*_overflow`。

**泛型**：`_Generic` 做类型分派（`<tgmath.h>` 的原理）；宏 + `typeof`（C23）写类型安全的 `MAX`、容器宏；复杂泛型容器考虑「侵入式链表 + `container_of`」。

## 六、高频 UB 与坑

| 坑 | 说明与对策 |
|---|---|
| 有符号溢出 | UB（C23 强制补码也不改变这一点）；编译器据此删除「溢出检查」。用 `ckd_add` 或先转无符号 |
| 严格别名 | 通过不兼容类型的指针访问同一对象是 UB；类型双关用 `memcpy` 或 union（C 允许，C++ 不允许）；`-fno-strict-aliasing` 只是止痛药 |
| 整型提升与隐式转换 | `uint8_t a = 200, b = 100; a + b` 的结果类型是 `int`；有符号与无符号比较陷阱（`-1 < 1u` 为假）；`-Wconversion -Wsign-compare` |
| 未序列化修改 | `i = i++ + 1;`、`f(i++, i++)` 是 UB |
| 越界与悬垂 | 越界读写、返回局部变量地址、`free` 后使用、双重 `free`；ASan 首选 |
| 未初始化读取 | 自动存储期变量未初始化即读取，结果不可依赖；`-Wuninitialized`、MSan |
| 移位 | 移位量 ≥ 位宽或为负是 UB；负数左移是 UB |
| VLA | 栈溢出无法检测、MSVC 不支持、C11 起可选；生产代码避免 |
| `volatile` | 只保证不优化掉访问，**不是同步原语**；线程间同步用 `_Atomic` / 互斥锁 |
| 位域 | 布局、符号性、字节序依实现而定；协议解析别用位域映射报文 |
| 格式化字符串 | `printf("%d", size_t_val)` 类型不符是 UB；用 `%zu`、`PRId64`；`-Wformat=2` |
| `strncpy` / `strcat` | `strncpy` 不保证 `\0` 结尾；优先 `snprintf` 或自写带长度的拷贝 |

## 七、C 与 C++ 互操作

```c
/* api.h —— C 与 C++ 共用头文件 */
#ifdef __cplusplus
extern "C" {
#endif
typedef struct db db;
int  db_open(const char *path, db **out);
void db_close(db *d);
#ifdef __cplusplus
}
#endif
```

| 差异 | C | C++ |
|---|---|---|
| `void*` 隐式转换 | 允许 `int *p = malloc(n);` | 必须显式转换（或别用 malloc） |
| 字符字面量类型 | `'a'` 是 `int` | `'a'` 是 `char` |
| 空参数列表 | C23 前 `f()` 是「参数未指定」 | 等价 `f(void)` |
| 指定初始化 | 任意顺序、可嵌套、可数组下标 | C++20 起支持，但必须按声明顺序、不支持数组下标 |
| 复合字面量 / VLA / `restrict` | 标准特性 | 非标准（编译器扩展，`__restrict`） |
| `const` 全局变量链接性 | 外部链接 | 默认内部链接 |
| union 类型双关 | 允许 | UB（用 `std::bit_cast`） |

跨边界规则：只传 C 兼容类型（标准布局 struct、不透明指针、整数、函数指针）；**异常绝不能穿过 C 栈帧**（C++ 回调给 C 库时在边界 `try/catch` 全捕获）；内存谁分配谁释放，提供配对的 `*_free`；C++ 侧用 `std::unique_ptr<T, decltype(&t_free)>` 托管 C 资源。

## 八、嵌入式与系统编程速查

- MMIO：`volatile uint32_t *const REG = (volatile uint32_t *)0x40021000u;`；读改写需关中断或用硬件位带/原子指令。
- 中断与主循环共享变量：`volatile sig_atomic_t` 或 `_Atomic`，复杂数据结构用临界区。
- 定宽类型用 `<stdint.h>`；字节序显式处理（`htonl` 族；C23 `<stdbit.h>` 只提供字节序判断宏，不提供字节交换函数），协议解析逐字节组装，不做结构体强转。
- 静态内存优先：禁用或封装 `malloc`，内存池 + 编译期容量检查（`static_assert`）。
- 合规：汽车/工控项目遵循 MISRA C:2023；安全编码参照 SEI CERT C。

## 九、学习与查证来源

- cppreference C 部分（en.cppreference.com/w/c）：语言与库语义、编译器支持表。
- WG14 文档站（open-std.org/jtc1/sc22/wg14）：标准草案与提案（N 号文档）。
- GCC / Clang 各版本 release notes 与「Porting to GCC N」：默认标准变化与迁移坑。
