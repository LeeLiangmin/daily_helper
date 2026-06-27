# ABI（Application Binary Interface）深度技术调研报告

## TL;DR
- **ABI 是编译后二进制代码层面的"契约"**：它规定函数调用约定、数据布局、寄存器使用、栈帧、名称修饰和对象模型等，决定两个独立编译的二进制模块能否正确互操作；而 API 是源码层面的契约。C ABI 因其简单、无名称修饰、与系统调用绑定，成为所有语言互操作（FFI）的事实标准"通用语"。
- **Rust 和 C++ 都缺乏稳定的跨编译器/跨版本原生 ABI**：Rust 故意不稳定化 `extern "Rust"`（保留布局优化自由度），通过 `extern "C"` + `#[repr(C)]` 与外界互操作；C++ 因 vtable/RTTI/异常/名称修饰的复杂性，GCC/Clang（Itanium ABI）与 MSVC ABI 互不兼容，且历史上多次破坏（GCC 3→4 改 soname、GCC 5 的 libstdc++ COW string ABI 分裂）。
- **工程上靠 C ABI + 符号版本化 + ABI 检测工具管理兼容性**：Swift 自 5.0 起在 Apple 平台稳定 ABI；Go 用双 ABI（ABI0 栈式 / ABIInternal 寄存器式，1.17 引入）；Python 用 Limited API/Stable ABI（abi3）。版本化符号（`.symver`/glibc）、libabigail、abi-compliance-checker 是保证动态库兼容的核心手段。

## Key Findings

1. **ABI ≠ API**：API 是源代码层面的接口（函数签名、类型名），ABI 是机器码层面的接口（符号名、内存布局、调用约定）。API 兼容只需重新编译，ABI 兼容则要求无需重新编译即可链接/加载。
2. **C ABI 是事实标准**，因为操作系统系统调用接口、动态链接器、几乎所有语言运行时都以它为基准；它没有名称修饰、调用约定可预测、类型布局明确。
3. **System V AMD64 ABI**（Linux/macOS/BSD）与 **Windows x64 ABI** 是两套不同的 x86-64 C 调用约定，寄存器分配、影子空间（shadow space）、红区（red zone）等都不同。
4. **C++ ABI 极其复杂**，Itanium C++ ABI（GCC/Clang）与 MSVC ABI 在名称修饰、vtable 布局、异常处理上完全不兼容，且 C++ 标准委员会至今拒绝破坏 ABI。
5. **Rust ABI 不稳定是有意设计**，但 v0 符号修饰方案已稳定化为 Rust 自有方案；生态用 `stabby`、`abi_stable` 提供可选的稳定 ABI。
6. **工程实践**依赖符号版本化、SONAME 管理、libabigail/abidiff 检测。

## Details

### 一、什么是 ABI

#### 1.1 定义与概念
ABI（Application Binary Interface，应用程序二进制接口）是软件在"进程内机器码访问"层面暴露的接口。它是两个独立编译的二进制程序模块之间的契约，规定了数据如何在内存中组织、函数如何被调用、系统如何管理资源。ABI 处于相对低的抽象层级，其兼容性取决于目标硬件和构建工具链。

一个完整的操作系统级 ABI 还包括目标文件、程序库的二进制格式（如 ELF）、可执行文件格式、动态链接语义等。例如 System V Release 4 ABI、Intel Binary Compatibility Standard (iBCS) 都是这类。嵌入式系统则有 EABI（Embedded ABI）。

#### 1.2 ABI 与 API 的区别
- **API**（Application Programming Interface）是源代码层面、相对高层、硬件无关、人类可读的接口契约。它在编译之前定义接口，包括函数签名、导出类型名。对于 C 库，这一信息封装在头文件中。
- **ABI** 描述两个独立编译产物之间的边界，包括编译器修饰后的导出符号名、导出类型的内存布局。

核心区别：API 是源码级契约，ABI 是编译级契约。如果 API 改变，需要重新编译；如果 ABI 改变，重新编译不够，所有依赖方都必须重建。一个经典类比：在 Java 中，库的 API 是其源码，ABI 是应用所依赖的 class 文件。

#### 1.3 ABI 包含哪些内容
- **函数调用约定（calling convention）**：参数如何传递（寄存器还是栈、顺序如何）、返回值放在哪里、哪些寄存器由被调用者保存（callee-saved/非易失寄存器）、栈的准备和清理由谁负责。
- **数据类型布局**：标量类型的大小、对齐、字节序，结构体/联合体的成员排布、填充（padding）、位域规则。
- **寄存器使用**：哪些寄存器传参、哪些是易失/非易失。
- **栈帧结构**：栈如何增长、对齐要求、返回地址位置。
- **名称修饰（Name Mangling）**：源码符号名如何映射为链接器使用的符号名。
- **对象内存模型**：C++ 中的虚函数表（vtable）布局、RTTI、异常处理表等。

如 x86 calling conventions 所述，调用约定、类型表示、名称修饰共同构成 ABI；不同编译器实现上有微妙差异，所以跨编译器链接往往困难。

#### 1.4 ABI 稳定性、兼容性与破坏
- **ABI 稳定性（ABI stability）**：保证一个版本编译的二进制能与另一个版本编译的二进制在运行时正确交互。
- **ABI 兼容性（ABI compatibility）**：编译产物在不同版本/组件间无需重新编译即可协同工作。向后兼容意味着新版库的 ABI 变化不影响链接旧版的应用。
- **ABI 破坏（ABI breakage）**：ABI 改变导致运行时失败——典型表现为段错误（segmentation fault）、数据损坏、找不到符号的链接错误。库的 ABI 若意外改变，依赖旧 ABI 的应用可能无法正确运行。

### 二、为什么 C ABI 是事实上的标准

#### 2.1 历史背景与原理
"C 类型是通用语，因为 C ABI 是通用语。"任何语言要打开文件、创建进程、分配内存，最终都必须调用这些接口——调用经由语言的 FFI，穿越平台的 C ABI，到达内核。

每个想调用外部代码的语言都提供 FFI（Foreign Function Interface）：Python 有 ctypes 和 cffi，Rust 有 `extern "C"`，Java 有 JNI，Zig 有 `@cImport`。FFI 处理语言级关注点（声明签名、编组字符串、管理内存所有权），但当调用真正发生时，FFI 必须发出遵守平台 ABI 的机器码。关键点在于：每个平台上，**C 编译器的调用约定成为该平台的标准 ABI**，所有其他语言需要互操作时都以该 ABI 为目标。当两种语言需要互操作时，边界处的函数调用（FFI）传统上通过 C（用 libffi 这样的库）进行。

C 之所以胜任这一角色，原因在于：无名称修饰、可预测的调用约定、明确的类型布局、成熟的工具链支持。大量基础设施（Linux、Git、curl、nginx、Apache、PostgreSQL、SQLite、OpenSSL、zlib、ffmpeg）用 C 编写并将长期保持。

#### 2.2 System V AMD64 ABI（x86-64 Linux/macOS 标准）
这是 Solaris、Linux、FreeBSD、macOS 等类 Unix 系统遵循的事实标准。要点：
- **整数/指针参数**：前六个依次用 RDI、RSI、RDX、RCX、R8、R9（R10 在嵌套函数时用作静态链指针）。
- **浮点参数**：前八个用 XMM0–XMM7。
- **系统调用**：用 R10 代替 RCX 传第四个参数，系统调用号放 RAX。
- **返回值**：整数 ≤64 位放 RAX，≤128 位放 RAX:RDX；浮点放 XMM0/XMM1。
- **被调用者保存寄存器**：RBX、RBP、RSP、R12–R15；其余为调用者保存（易失）。
- **栈对齐**：进入函数入口时 (%rsp + 8) 必须是 16 的倍数（传 __m256 时为 32）。GCC 4.5 起强制 16 字节对齐（因 SSE 对齐加载指令）。
- **红区（red zone）**：叶函数可使用 %rsp 以下 128 字节而不被信号/中断处理程序破坏。
- **聚合体传递**：结构体按 eightbyte 分类为 INTEGER/SSE/MEMORY；≤两个 eightbyte 且对齐合适的拆分到寄存器，大于四机器字（>32 字节）或不可对齐的传内存（调用者分配内存，传指针作为隐藏首参数）。
- **可变参数函数**：AL/RAX 须含用于浮点参数的向量寄存器数（0–8）。

ELF（Executable and Linkable Format）是 System V ABI 的一部分，由可移植基础文档加上平台特定补充（processor supplement，如 i386、x86-64）组成。

#### 2.3 Windows x64 ABI
微软 x64 调用约定（Windows 和 pre-boot UEFI 长模式），随 Visual Studio 2005 引入：
- **前四个整数/指针参数**：RCX、RDX、R8、R9。
- **前四个浮点参数**：XMM0–XMM3。
- **影子空间（shadow space / home space）**：调用者必须在调用前在栈上为前四个寄存器参数预留 32 字节，即使被调用者参数少于四个也要预留。被调用者可用此空间溢出（spill）寄存器参数。第五个及以后的参数压在影子空间之上。
- **返回值**：整数放 RAX，浮点放 XMM0。大于 64 位的结构体由调用者分配空间、传指针作隐藏首参数，返回指针放 RAX。
- **易失寄存器**：RAX、RCX、RDX、R8–R11、XMM0–XMM5；非易失：RBX、RBP、RDI、RSI、RSP、R12–R15。
- 与 System V 相比无红区，影子空间是其独有特征。32 位的 `__fastcall`/`__stdcall` 等关键字在 x64 下被忽略。

#### 2.4 其他平台 C ABI 规范
- **ARM AAPCS64**（Procedure Call Standard for AArch64）：31 个 64 位通用寄存器（X0–X30），X0–X7 传整数参数和返回值，V0–V7（SIMD）传浮点；X0 放返回值。
- **Itanium C++ ABI**：虽以 Itanium 命名，实为 GCC/Clang 在多数平台采用的 C++ ABI，定义在"基础 C ABI"之上分层（有项目正把 C++ ABI 用可移植 C 概念重述）。
- **RISC-V Calling Convention**、**PowerPC System V ABI** 等各架构都有自己的处理器补充。

#### 2.5 标准库、系统调用与 C ABI 的关系
操作系统的系统调用接口本身就是用 C ABI 表达的（System V ABI 定义了系统调用约定）。C 标准库（libc/glibc）作为系统调用的封装，其导出符号遵循 C ABI，因而成为所有语言进入内核的统一门户。

### 三、Rust ABI 问题

#### 3.1 现状：`extern "Rust"` 不稳定、无规范
在 Rust 中，除非通过 `#[repr(_)]` 显式选择已知表示、或用 `extern "_"` 显式选择调用约定，编译器对类型布局和调用约定享有完全自由：这个过程明确是不稳定的，取决于编译器版本、优化级别等。具体后果是：通过不同编译器调用构建的软件单元（如动态库和依赖它的可执行文件）可能对 ABI 决策产生分歧，而链接器无从知晓。例如可执行文件认为 `Vec<Potato>` 最左 8 字节是堆分配指针，而库认为是长度。

#### 3.2 为什么 Rust 故意不稳定化自己的 ABI
保持 ABI 不稳定让编译器拥有布局优化自由（如字段重排、枚举的 niche 优化/空指针优化等），这些优化能减小内存占用、提升性能。稳定 ABI 会牺牲这些优化空间。Rust 选择"稳定但 unsafe 的 C ABI（`extern "C"`）"作为互操作手段，而把原生 ABI 留作实现细节。

#### 3.3 通过 `extern "C"` 与 C 互操作
导出可被 C 调用的函数用 `extern "C"` 加 `#[no_mangle]`（保持符号名不被修饰）。例如 librsvg：
```rust
#[no_mangle]
pub unsafe extern "C" fn rsvg_handle_new_from_file(
    filename: *const libc::c_char,
    error: *mut *mut glib_sys::GError,
) -> *const RsvgHandle { /* ... */ }
```
`extern "system"` 会选择目标平台合适的 ABI（如 win32 x86 上是 stdcall，x86_64 Windows 上是 C）。

非空指针优化：引用（`&T`、`&mut T`）、`Box<T>`、函数指针在 Rust 中定义为永不为 null，因此 `Option<extern "C" fn(c_int) -> c_int>` 正是表示可空函数指针的正确方式（对应 C 的 `int (*)(int)`），`None` 对应 null，无需额外判别位。

#### 3.4 布局控制
- **`#[repr(C)]`**：保证结构体/枚举布局与平台 C 表示兼容。
- **`#[repr(transparent)]`**：单字段结构体与其唯一字段有相同 ABI 表示（用于 newtype 包装跨 FFI）。
- **`#[repr(packed)]`** / `#[repr(C, packed)]`：无填充排布成员。

注意陷阱：用空枚举作 FFI 类型是极糟的做法（编译器依赖空枚举不可居留 uninhabited，处理 `&Empty` 是巨大隐患可触发 UB）。历史上还存在 `repr(C)` 枚举在 System V ABI 上按值传递/返回时的 bug（rust-lang/rust issue #68190），导致段错误或断言失败。

#### 3.5 Name Mangling：v0 修饰方案
Rust 的传统（legacy）修饰基于 Itanium C++ ABI 风格（符号以 `_ZN` 开头，用文件名哈希消歧，自 Rust 1.9 起使用），有信息丢失、依赖编译器内部等缺陷。RFC 2603 提出的 **v0 方案**：
- 符号以 `_R` 开头。
- 为二进制符号表中一切可出现的东西提供无歧义编码。
- 以可逆方式编码泛型参数信息（可解码出具体的泛型实例）。
- 不编码返回类型（Rust 无重载）。
- 符号限定为 A–Z、a–z、0–9 和 `_`，Unicode 名用改进的 punycode。
- 用基于字节地址的 backref（`B` + base-62 数）做压缩。

**v0 解码示例**（据 rustc book "v0 Symbol Format"）：crate `mycrate` 中的函数 `example` 修饰为 `_RNvCs15kBYyAo9fc_7mycrate7example`——`_R` 是 Rust 符号前缀，`N` 是嵌套路径标签，`v` 是值（函数）命名空间，`Cs15kBYyAo9fc_` 是 crate-root（`C` + 消歧符），`7mycrate` 与 `7example` 是长度前缀标识符，解码为 `mycrate::example`。带泛型的 `example::<i32, 1>` 修饰为 `_RINvCsgStHSCytQ6I_7mycrate7examplelKj1_EB2_`——`I` 是泛型参数标签，`l` 编码类型参数 `i32`，`K` 标记 const 参数、`j` 是 `usize` 类型、`1_` 是值 1，`E` 终止泛型参数，`B2_` 是指向 crate-root 的 backref。

v0 不试图兼容 C++ 修饰方案，目前也不作为 Rust 稳定 ABI 的规范（rustc book 原文："The v0 format is not presented as a stable ABI for Rust."）。可用 `-C symbol-mangling-version=v0` 启用。2025 年 11 月 20 日 Rust 博客宣布，自 nightly-2025-11-21 起 rustc 默认在 nightly 上使用 v0 方案。

#### 3.6 稳定 ABI 生态方案
- **stabby**（ZettaScaleLabs）：通过 `#[stabby::stabby]` 为类型的子集"钉住"ABI，同时保留 rustc 的部分布局优化（如枚举 niche 优化）。标注后结构体变为 `#[repr(C)]` 并实现 `IStable` trait（用关联类型表示布局含 niche）。本质上通过所选 stabby-abi crate 版本提供"版本化 ABI"。
- **abi_stable**（rodrimati1992）：最老的动态链接方案之一，加载多个 Rust 动态库时做运行时类型检查。核心是 `StableAbi` trait（用 derive 宏断言 FFI 安全并在加载时获取布局以校验）。提供 std 类型的 FFI 安全等价物（`RString`、`RVec`、`RBox`、`RArc`、`ROption` 等），用 `sabi_trait` 宏生成 FFI 安全的 trait 对象（`DynTrait`），用 prefix types 支持可扩展模块/vtable 而不破坏 ABI。每个 0.y.0 和 x.0.0 版本定义自己的不兼容 ABI；加载时校验类型布局，允许 semver 兼容的变更。

#### 3.7 社区讨论与未来展望
近期有 `interoperable_abi` 提案（`#[repr(interop)]`），基于已稳定的 C ABI 扩展 Rust 友好特性：良定义的胖指针（slice、str、trait 对象）、优化枚举空间、支持零大小字段等。也有 RFC 3435（按函数选择 ABI）的讨论。但有观点（如 Manishearth、blaz.is）认为真正需要的是"类型信息"而非稳定 ABI——像 Diplomat、CXX、uniffi 这类工具仍需生成包装层。Rust ABI 项目设想分阶段推进（`extern dyn crate` 动态加载、稳定 TypeId、在二进制中存储类型布局等）。

### 四、C++ ABI 问题

#### 4.1 复杂性来源
C++ ABI 远比 C 复杂，因为它要编码：
- **名称修饰**：命名空间、函数重载、模板都需编码进扁平的链接器符号表。
- **虚函数表（vtable）**：用于虚函数派发、访问虚基类子对象、RTTI。每个有虚函数或虚基类的类有一组 vtable，含 offset-to-top、typeinfo 指针、函数指针序列。
- **RTTI（运行时类型信息）**：typeinfo 结构。
- **异常处理**：栈展开表、异常类型匹配。
- **this 指针调整、thunk、构造 vtable、VTT**（虚继承场景）。

#### 4.2 Itanium C++ ABI（GCC/Clang）
GCC 和 Clang 在类 Unix 环境收敛到 Itanium ABI（尽管名为 Itanium，覆盖 x86、x86_64 等所有不自定义 ABI 的平台）。这保证了 Clang 产物可被 GCC 使用，反之亦然。它在"基础 C ABI"之上分层定义对象布局、对齐、vtable 布局、名称修饰。LLVM 系列编译器（包括 D、Rust 早期）在非 Windows 平台多遵循它。

**名称修饰示例**（Itanium ABI / Wikipedia "Name mangling"）：所有修饰符号以 `_Z` 开头（下划线接大写字母在 C 中是保留标识符，避免与用户符号冲突）。自由函数 `int foo(int)` 修饰为 `_Z3fooi`——`3` 是标识符 `foo` 的长度前缀，`i` 是参数 `int` 的类型码，返回类型对非模板函数不编码。嵌套名 `wikipedia::structure::Article::format()` 修饰为 `_ZN9wikipedia8structure7Article6formatEv`——`N` 开始嵌套名、其后是一串 `<长度,标识符>` 对、`E` 终止、`v` 表示无参数（void）。常见类型码：`v`=void、`i`=int、`c`=char、`d`=double；前缀 `P`=指针、`R`=引用、`K`=const。

vtable 细节示例（`_ZTV4Base` 即 Base 的 vtable 修饰名）：偏移 0 是 offset-to-top，偏移 8 是 typeinfo 指针，之后是按声明顺序的虚函数指针；析构函数有两个条目（D1 完整对象析构、D0 删除析构）。虚继承引入构造 vtable（如 `_ZTC7Derived0_4Left`）和 VTT（Virtual Table Table）。

#### 4.3 MSVC C++ ABI
微软为 Windows 定义了自己的 C++ ABI，与 Itanium 完全不同：名称修饰、vtable 布局、异常处理都不一样。Windows 上的 MSVC 不遵循 Itanium ABI；但 LLVM 系编译器针对 Windows 目标时会遵循 Windows ABI。

**MSVC 修饰示例**：装饰名以 `?` 开头。`int foo(int)` 修饰为 `?foo@@YAHH@Z`——`?` 引入装饰名，`foo` 是函数名，`@@` 终止作用域限定（空作用域即全局），`Y` 表示普通非成员函数，`A` 是 `__cdecl` 调用约定码，第一个 `H` 是返回类型 `int`（MSVC 与 Itanium 不同，**编码**返回类型），第二个 `H` 是参数 `int`，`@Z` 终止签名（无参函数则以 `XZ` 结尾，`X`=void）。对比：`void h(int)` → `?h@@YAXH@Z`、`void h(void)` → `?h@@YAXXZ`。须注意微软无官方完整修饰算法文档（仅记录"编码了什么"和 `undname` 工具），上述属逆向工程得来的约定。MSVC 修饰还会给 C 函数加修饰：`__cdecl` 加前导 `_`，`__stdcall` 加前导 `_` 和尾随 `@` 加参数字节数。

#### 4.4 GCC vs Clang 的 ABI 兼容性
在 Unix 平台二者都遵循 Itanium ABI，因此 GCC 编译的对象与 Clang 编译的对象通常可互链接。但仍有边角不兼容（如 libstdc++ 新 ABI 中某些要求 clang++ 当时未完全实现）。

#### 4.5 历史破坏案例
- **GCC 3.x → 4.0**：GCC 3.4 时 libstdc++ soname 升到 libstdc++.so.6（`.so` 号最后一次递增）。自 GCC 3.4 / `-fabi-version=2` 起 C++ ABI 趋于稳定，3.4.x 编译的库可被 4.y.z 链接（前提是 ABI 影响标志一致）。事后 GCC 团队认为"改 soname 是个广受诟病的错误"，因为改 soname 仍不足以处理符号 ABI 变化。
- **GCC 5.1 的 libstdc++ 双 ABI（dual ABI）**：C++11 标准禁止 COW（写时复制）字符串、要求 list 记录大小，于是 `std::string` 和 `std::list` 换了新实现。为向后兼容，新实现放在内联命名空间 `std::__cxx11` 中（如 `std::__cxx11::list<int>`），由 `_GLIBCXX_USE_CXX11_ABI` 宏（1 或 0）控制。若链接器报关于 `std::__cxx11` 命名空间或 `[abi:cxx11]` 标签的未定义引用错误，通常意味着混链了不同 `_GLIBCXX_USE_CXX11_ABI` 值编译的对象文件。`std::ios_base::failure` 异常类型也因基类从 `std::exception` 改为 `std::system_error` 而布局改变。GCC 引入 `abi_tag` 属性来在符号名中编码 ABI 变化、避免同名符号冲突。

#### 4.6 为什么跨编译器 ABI 至今是问题
C++ 标准从不强制名称修饰方案，且 ABI 正式不在标准范围内。如 WG21 论文 P2028R0《What is ABI, and What Should WG21 Do About It?》原文所述："Formally, ABI is outside of the scope of the standard. WG21 does not control the mangling of symbols, the calling convention, unspecified layout of types, or any of the other things that factor into ABI."标准化修饰本身不足以保证互操作，反而可能制造"可安全互操作"的错觉——因为异常处理、vtable 布局、结构体/栈帧填充等其他 ABI 方面同样导致不兼容。实际上，让不兼容的编译器使用不同修饰是一种安全特性：能在链接期而非运行期暴露不兼容，避免隐晦 bug。

C++ 标准委员会（WG21）在 2020 年布拉格会议上就是否破坏 ABI 进行了一系列投票，最终决定不破坏（据会议纪要 P2130R0："We decided not to promise ABI stability... We did not have a consensus for a big ABI break for C++23"；底层问题文档为 P1863R1《ABI — Now or Never》）。正如 cor3ntin《The Day The Standard Library Died》所言："the C++ committee took a series of polls on whether to break ABI, and decided not to. There was no applause."委员会在三者间无法取舍，正如 Bryce Adelstein Lelbach 的著名表述："Performance / ABI Stability / Ability to Change — You can pick two, choose wisely."许多影响深远的库改进（如更好的 hashing）因 ABI 顾虑被早早过滤掉。WG21 设有专门的 ABI Review Group（2026 年由 Daveed Vandevoorde 主持）。

#### 4.7 `extern "C"` 在 C++ 中的作用与限制
`extern "C"` 让 C++ 函数以 C 链接（不修饰名称、用 C 调用约定）导出，是 C++ 暴露稳定 ABI 的主要手段。但限制明显：无法跨边界传递模板、泛型、类的字段可见性；C++ 模板会被编译并静态链接进调用方代码（因为无法把 `Array<T>` 中 `T` 的大小信息跨 C ABI 传递）；数据所有权只能靠文档约定。

### 五、其他语言的 ABI 问题

#### 5.1 Swift ABI 稳定性（Swift 5.0 之后）
Swift 5.0 在 Apple 平台（macOS/iOS/watchOS/tvOS）实现了 ABI 稳定，提供应用的二进制兼容性保证：一个版本编译器构建的应用能与另一版本构建的库交互。结果是 Swift 运行时和标准库随 OS 发布，应用不再需把它们嵌入 bundle（据 Swift.org 官方博客《ABI Stability and More》，ABI 稳定后应用不再需将标准库及 "overlay" 库内嵌；社区量化为此前每个应用约增加 5 MB，如 "This added around 5 MB to the size of your app"）。

相关机制：
- **库演进（resilience/library evolution）**：可选特性，用 `@inlinable`（Swift 4.2/SE-0193）、`@frozen`（Swift 5.1，取代实验性 `@_fixed_layout`）等注解在性能与未来灵活性间权衡。`@frozen` 承诺结构体不增删重排存储属性、枚举不增删重排 case；结构体必须"生而 frozen"或永远 resilient，加/去 `@frozen` 本身是二进制不兼容变更。
- **模块稳定性（module stability，Swift 5.1）**：引入基于文本的 `.swiftinterface` 模块接口文件，使不同编译器版本构建的二进制框架可被使用。需开启 "Build Libraries for Distribution"。

非 Apple 平台（Linux/Windows）的 ABI 尚未稳定，Swift Core Team 称随这些平台成熟会评估稳定化。

#### 5.2 Go ABI（internal vs register-based）
Go 有两套 ABI：
- **ABI0**：基于 Plan 9 的栈式调用约定，所有参数和返回值经栈内存传递。这是稳定的、汇编代码使用的 ABI，也是跨语言边界的保证契约。
- **ABIInternal**：Go 1.17 引入的寄存器式调用约定，用于纯 Go 函数调用。据 Go 官方提案《Proposal: Register-based Go calling convention》（Austin Clements / David Chase, issue #40724）："Preliminary experiments indicate this will achieve at least a 5–10% throughput improvement across a range of applications."（约 5–10% 吞吐提升）；同一提案指出："accessing arguments in registers is still roughly 40% faster than accessing arguments on the stack."（寄存器访问比栈访问快约 40%）。这个 ABI 不稳定，随 Go 版本变化。

二者通过透明的 **ABI 包装器（ABI wrapper/bridge）** 互相调用。Go 刻意为 Go 设计自有 ABI 而非用平台 ABI，原因是 goroutine 的可伸缩性需求：运行时动态调整 goroutine 栈大小，这对 ABI 提出平台 ABI 不满足的要求；且 Go 的栈式约定无被调用者保存寄存器，简化了 GC 栈追踪、栈增长和 panic 展开。amd64 上 R14 作 goroutine 指针、X15 作固定零寄存器。

#### 5.3 Python C 扩展 ABI（Stable ABI / Limited API）
- **Limited API**（Python 3.2 引入，PEP 384）：C API 的安全子集。扩展模块定义 `Py_LIMITED_API` 后，只暴露这个子集。
- **Stable ABI（abi3）**：Limited API 对应的前向兼容 ABI——只用 Limited API 的扩展编译一次即可在多个 Python 3.x 版本上加载，无需重新编译。
- 把 `Py_LIMITED_API` 定义为支持的最低 Python 版本对应的 `PY_VERSION_HEX`（如 0x030A0000 表示 3.10）。在某些平台上 Python 会查找带 `abi3` 标签的共享库文件名（如 `mymodule.abi3.so`）。Stable ABI 的所有函数都作为真正的函数（非宏）存在于 Python 共享库中，使非 C 预处理器语言也能使用。Windows 上应链接 `python3.dll` 而非版本特定的 `python39.dll`。
- Stable ABI 防止 ABI 问题（缺符号的链接错误、结构体布局或函数签名改变导致的数据损坏），但不保证完整行为兼容（如某函数在新版接受 NULL 而旧版会解引用崩溃）。
- 对加密库等而言，abi3 的核心价值是"O(1) 构建"——旧 wheel 在新 Python 上仍可用，无需为每个 Python 版本重新构建。Trail of Bits 指出 abi3 wheel 构建/标记两步都未被验证（setuptools 与 wheel 是两套代码库），易出现标记与实际不符。
- 未来：PEP 803（abi3t，针对自由线程构建的 Stable ABI，Python 3.15+）、PEP 809（按年份命名的 Stable ABI，如 abi2026）正在演进。

#### 5.4 通用模式
几乎所有语言都通过"降级到 C ABI"实现互操作：把丰富的语言类型在边界处映射为 C 兼容的布局，经由 libffi 之类的库或语言内建 FFI 完成调用。.NET 中 C 是默认的互操作目标；COM/IUnknown ABI 也是按 C 对齐设计的稳定面向对象 ABI。

### 六、实际工程影响

#### 6.1 动态库（.so/.dll）的 ABI 兼容性管理
关键是 SONAME 机制：在 ELF 系统上，文件名 `libstdc++.so.5.0.4` 对应 DT_SONAME `libstdc++.so.5`，相同 DT_SONAME 的二进制向前兼容。破坏 ABI 时应递增主版本号（soname）。但 GCC 经验表明，仅改 soname 不足以处理符号级 ABI 变化——若加载多个依赖不同库版本的共享对象，仍会发生同名符号冲突。

#### 6.2 版本化符号（versioned symbols）
glibc 自 2.1（约 1999）引入 ELF 符号版本化（扩展自 Sun 的方案），让同一函数的多个版本共存于单个库。例如新二进制绑定 `memcpy@@GLIBC_2.14`（`@@` 标记默认版本），旧二进制继续用 `memcpy@GLIBC_2.2.5`（`@` 标记旧版本），无需重新编译。这是老程序仍能在现代 Linux 上运行的原因。

实现机制：三个 ELF 节——`.gnu.version`（每个符号映射到版本索引）、`.gnu.version_d`（定义可用版本及依赖，即 Elfxx_Verdef）、`.gnu.version_r`（可执行文件中所需符号版本，即 Elfxx_Verneed）。用 `.symver` 汇编指令（`.symver real, name@version` 或 `name@@version`）和链接器版本脚本（version script，含 `local: *;` 隐藏内部符号）控制。例如：
```c
__asm__(".symver clock_gettime,clock_gettime@GLIBC_2.17");
```
可强制绑定到旧版符号。二进制中最高的 GLIBC 版本标签决定目标系统所需的最低 glibc 版本（可用 `objdump -T` / `readelf -V` 检查）。

#### 6.3 ABI 检测工具
- **libabigail**（Red Hat 维护，2015 年首发）：把库归约为称为 "corpus" 的 ABI 制品集（类型、函数、变量及元数据，从 ELF + DWARF 解析）。命令行工具：`abidw`（导出 XML）、`abidiff`（比较两个库/XML）、`abicompat`、`abipkgdiff`（比较 RPM 等二进制包）。返回状态码是位图，可判断 ABI 变化类型（如第 4 位/值 8 = 破坏向后兼容，第 3 位/值 4 = 潜在破坏需人工评审）。是目前唯一活跃维护的 ABI 合规检查工具，被 RHEL、Fedora、Android、glibc 采用。
- **abi-compliance-checker**（lvc）：基于 ABI Dumper。优点：分别分析向后二进制兼容和源码兼容、给 ABI 变化分级、解释影响、可视化报告、按根因分组、估算兼容率、支持 MSVC 和 XCode 二进制；缺点：用 C++ 实现可能更慢更耗内存、报告生成不可配置。
- **abi3audit**（Trail of Bits）/ **abi3info**：检查 Python abi3 wheel 的合规性。
- libstdc++ 自带 `make check-abi`（对照基线检查导出符号）。

#### 6.4 ABI 设计最佳实践
- 公共接口尽量用 C ABI（`extern "C"`）暴露，避免暴露 C++ 类布局、模板、内联函数。
- 用 "pimpl"（指针实现）惯用法隐藏实现，避免结构体布局成为 ABI 一部分。
- 用不透明指针（opaque pointer）和句柄。
- 仅在结构体尾部添加字段，预留保留字段；用版本字段或大小字段。
- 用符号版本化和版本脚本控制导出面、隐藏内部符号。
- 把 ABI 检查（libabigail/abidiff）纳入 CI/发布流程。
- 破坏 ABI 时递增 SONAME 主版本。
- Rust 场景：用 `#[repr(C)]`、`stabby` 或 `abi_stable`；Python 场景：用 Limited API 编译 abi3 wheel。

## Recommendations

1. **跨语言/跨编译器互操作一律以 C ABI 为边界**。设计共享库的公共接口时，用 `extern "C"`（C++/Rust）暴露纯 C 风格函数与 `#[repr(C)]`/POD 数据类型，绝不在 ABI 边界暴露 C++ 模板、STL 容器、Rust 原生类型或异常。基准：若你的头文件中出现 `std::string`、`std::vector`、模板或 `inline` 函数作为导出接口，就已经把自己绑定到特定编译器/标准库 ABI。
2. **C++ 库分发**：在 Linux 上明确并固定 `_GLIBCXX_USE_CXX11_ABI` 取值，并在文档中声明；同一发行链内统一用相同 GCC/libstdc++ 主版本构建所有 C++ 组件。若必须分发预编译 C++ 库，强烈建议改用 C 包装层（C ABI shim）。触发重建的阈值：libstdc++ soname 递增、或链接器报 `std::__cxx11` / `[abi:cxx11]` 未定义引用。
3. **Rust 动态链接**：默认静态链接（Rust 的常态）；只有在确需运行时插件/动态加载时才引入 `abi_stable` 或 `stabby`，并接受其"每个主版本不兼容、加载时校验布局"的代价。不要假设两个用不同 rustc 构建的 `cdylib` 能安全传递任何非 `#[repr(C)]` 类型。
4. **建立 ABI 回归门禁**：对任何承诺二进制兼容的共享库，在 CI 中加入 `abidiff`（libabigail）对照上一个发布版本；将返回码第 4 位（值 8，破坏向后兼容）设为构建失败，第 3 位（值 4，潜在破坏）设为人工评审。Python 扩展用 `abi3audit` 验证 wheel 标记与实际一致。
5. **用符号版本化延长库寿命**：对长期维护的 C 库采用 glibc 风格的版本脚本（`local: *;` 隐藏内部符号 + 显式导出面），需要改变函数语义时新增版本节点（`@@` 默认 + `@` 旧版）而非原地修改。
6. **语言选择层面**：若项目核心诉求是稳定的二进制框架分发，Swift（Apple 平台）和有 abi3 的 Python 扩展提供了开箱即用的稳定 ABI；C++ 和 Rust 则需自行通过 C ABI 边界管理。

## Caveats
- **平台 ABI 持续演进**：System V ABI 等是"可扩展"标准，处理器补充不断新增；本报告的寄存器/对齐细节针对当前主流 x86-64，具体项目须查阅对应处理器补充文档。
- **MSVC C++ ABI 无官方完整算法文档**：微软记录"编码了什么"和 `undname` 工具，但字节级修饰算法是逆向工程得来的，相关示例属约定俗成而非单一权威规范。
- **Rust v0 默认化是 nightly 阶段**：2025-11-21 起 v0 成为 nightly 默认，但截至本报告日期（2026 年 6 月）尚未见其成为 stable 默认的确认；且 v0 明确"不是 Rust 稳定 ABI 的规范"。
- **"ABI 稳定"不等于"行为兼容"**：Python Stable ABI 等只防止符号/布局层面的破坏，不防止语义变化（如参数取值范围改变导致的崩溃）。
- **C++ 标准委员会 ABI 决策有争议**：2020 年布拉格"不破坏 ABI"的决定在社区（如 cor3ntin）看来争议极大，未来是否、何时破坏仍无定论；相关投票"无共识"。
- 部分细节来自社区博客/二级来源（Medium、个人博客），已尽量交叉印证核心事实（寄存器分配、宏名、版本号）于一手规范（System V ABI PDF、Itanium ABI 站点、Rust/Swift/Go/Python 官方文档、GCC 文档、WG21 论文）。