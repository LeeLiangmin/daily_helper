# 代码测试覆盖率赋能文档

GCC 与 Clang/LLVM 体系完整指南 | 版本 2.0 | 适用 GCC ≥ 9.x，Clang/LLVM ≥ 12.x

---

## 目录

1. [基本概念](#基本概念)
2. [GCC 覆盖率体系](#gcc-覆盖率体系)
3. [GCC 分支覆盖率专项](#gcc-分支覆盖率专项)
4. [Clang/LLVM 覆盖率体系](#clangllvm-覆盖率体系)
5. [两大体系对比](#两大体系对比)
6. [与 CMake 集成](#与-cmake-集成)
7. [与 Makefile 集成](#与-makefile-集成)
8. [CI/CD 集成实践](#cicd-集成实践)
9. [快速命令速查](#快速命令速查)

---

## 基本概念

**测试覆盖率（Code Coverage）** 衡量测试用例覆盖源代码的程度，帮助团队找出未被测试触达的代码路径，是提升代码质量的重要手段。

| 覆盖率类型 | 说明 | GCC | Clang |
|-----------|------|-----|-------|
| **行覆盖率（Line）** | 哪些源码行被执行过 | ✓ | ✓ |
| **函数覆盖率（Function）** | 哪些函数被调用过 | ✓ | ✓ |
| **分支覆盖率（Branch）** | if/switch 每条分支是否都被触发 | ✓ | ✓ |
| **区域覆盖率（Region）** | 比行更细的代码区块，能区分宏展开和模板实例 | — | ✓ |

---

## GCC 覆盖率体系

### 工具链数据流

```
源代码
  │
  ├─[编译: --coverage]──► .gcno  （结构注记，记录控制流图）
  │
  └─[运行]──────────────► .gcda  （执行计数，每次运行累积叠加）
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
                 gcov -b            lcov --capture
               .gcov 文本           coverage.info
                                        │
                                   genhtml
                                   HTML 报告
```

### 编译设置

`--coverage` 是核心标志，等价于 `-fprofile-arcs -ftest-coverage`，**编译和链接都必须加**。

```bash
# 单文件 C
gcc --coverage -O0 -g -o my_program my_program.c

# 多文件 C（分步编译）
gcc --coverage -O0 -g -c file1.c -o file1.o
gcc --coverage -O0 -g -c file2.c -o file2.o
gcc --coverage -O0 -g file1.o file2.o -o my_program   # 链接同样需要 --coverage

# C++ 项目
g++ --coverage -O0 -g -std=c++17 -o my_app main.cpp utils.cpp
```

> `-O0` 是准确覆盖率的前提：优化会折叠常量、内联函数，导致分支消失或行号错位。`-g` 保留调试信息，报告才能映射到具体行号。

| 编译选项 | 作用 |
|----------|------|
| `--coverage` | 等价于 `-fprofile-arcs -ftest-coverage`，同时启用行、函数、分支插桩 |
| `-fprofile-arcs` | 插入弧度计数器，运行后写 `.gcda` |
| `-ftest-coverage` | 生成 `.gcno` 控制流注记 |
| `-fprofile-update=atomic` | 多线程场景下原子更新 `.gcda`，避免竞态丢数据 |
| `-O0` | 禁用优化，保障覆盖率准确性 |
| `-g` | 保留调试信息，报告可定位源码行 |

### 运行测试

直接执行程序或测试套件，退出时自动写 `.gcda`：

```bash
./my_program                              # 单次运行
./my_tests --gtest_filter='*'            # Google Test

# 多场景运行时覆盖率数据自动累积
./my_program --input test1.txt
./my_program --input test2.txt
./my_program --input test3.txt
```

**程序崩溃时 `.gcda` 不会落盘。** 如需覆盖崩溃场景的覆盖率，在信号处理函数里手动刷写：

```c
#include <signal.h>

// GCC < 11
extern void __gcov_flush(void);
// GCC >= 11（推荐）
extern void __gcov_dump(void);

void crash_handler(int sig) {
    __gcov_dump();   // 刷写计数器到 .gcda
    _exit(1);
}

int main(void) {
    signal(SIGSEGV, crash_handler);
    signal(SIGABRT, crash_handler);
    // ...
}
```

### 生成报告

#### 方式一：gcov 文本报告（快速查看）

```bash
gcov my_program.c          # 基础行覆盖
gcov -b my_program.c       # 启用分支信息
gcov -b -c my_program.c    # 分支信息显示计数（更直观）
gcov -f my_program.c       # 显示每个函数的覆盖情况
```

生成的 `.gcov` 文件中，`#####` 表示从未执行的行，`-` 表示非可执行行：

```
        -:    0:Source:my_program.c
        1:    1:int process(int x, int y) {
branch  0 taken 8 (fallthrough)
branch  1 taken 2
        1:    2:    if (x > 10) {
        8:    3:        return x * y;
        -:    4:    }
        2:    5:    return x + y;
    #####:    6:    // 死代码，永远不会执行
```

#### 方式二：lcov + genhtml HTML 报告（推荐）

**安装：**
```bash
sudo apt-get install lcov      # Ubuntu/Debian
sudo yum install lcov          # RHEL/CentOS
brew install lcov              # macOS
```

**生成报告（含分支覆盖率）：**

> lcov 默认不收集分支数据，需显式启用。`--branch-coverage` 是推荐写法，详见下方「GCC 分支覆盖率专项」章节。

```bash
# 1. 收集覆盖率数据（--branch-coverage 启用分支统计，每步都要加）
lcov --capture \
     --directory . \
     --output-file coverage.info \
     --branch-coverage

# 2. 过滤系统头文件和第三方库
lcov --remove coverage.info '/usr/*' '*/third_party/*' \
     --output-file coverage_filtered.info \
     --branch-coverage

# 3. 打印摘要到终端
lcov --summary coverage_filtered.info --branch-coverage

# 4. 生成 HTML 报告
genhtml coverage_filtered.info \
        --output-directory coverage_html/ \
        --branch-coverage \
        --highlight \
        --legend

# 5. 打开报告
open coverage_html/index.html        # macOS
xdg-open coverage_html/index.html   # Linux
```

**终端摘要示例：**
```
Summary coverage rate:
  lines......: 91.2% (114 of 125 lines)
  functions..: 100.0% (12 of 12 functions)
  branches...: 68.4% (52 of 76 branches)
```

### 清理覆盖率数据

```bash
# 仅重置计数（保留 .gcno，无需重新编译）
lcov --zerocounters --directory .

# 手动删除执行数据
find . -name "*.gcda" -delete

# 删除所有覆盖率产物
find . \( -name "*.gcda" -o -name "*.gcno" -o -name "*.gcov" \) -delete
rm -rf coverage_html/ coverage.info coverage_filtered.info
```

### GCC 常见问题排查

| 现象 | 根因 | 解法 |
|------|------|------|
| 找不到 `.gcda` 文件 | 程序崩溃/被 kill，未正常退出 | 注册信号处理，调用 `__gcov_dump()` |
| 覆盖率始终为 0 | 链接时漏掉 `--coverage` | 链接命令补 `--coverage` 或 `-lgcov` |
| 行号与源码对不上 | 开启了编译优化 | 加 `-O0` |
| `.gcno` 与 `.gcda` 版本不匹配 | 重新编译前未清除旧 `.gcda` | `find . -name "*.gcda" -delete` 后再运行 |
| 多线程数据丢失/损坏 | `.gcda` 写入存在竞态 | 加 `-fprofile-update=atomic` |
| `branches` 行不显示 | lcov 默认关闭分支统计 | 所有 lcov/genhtml 命令加 `--rc branch_coverage=1` / `--branch-coverage` |

---

## GCC 分支覆盖率专项

分支覆盖率是 GCC 覆盖率中**最容易被忽略、也最有价值**的维度——行覆盖 100% 并不意味着所有条件路径都被测试过。

### 核心要点

编译阶段**无需额外标志**，`--coverage` 本身已插入分支计数器。分支数据的开关在**报告阶段**：

```
编译: gcc --coverage -O0 -g ...     ← 无需额外分支标志
运行: ./my_program
报告: lcov --rc branch_coverage=1   ← 关键开关在这里
HTML: genhtml --branch-coverage     ← HTML 中显示分支标记
```

### gcov 分支输出详解

```bash
gcov -b -c my_program.c
```

输出中每个条件语句下方会出现 `branch N taken M` 行：

```
        1:   10:int classify(int x) {
branch  0 taken 7 (fallthrough)    # x > 0 为 true，走 if 体
branch  1 taken 3                  # x > 0 为 false，走 else
        7:   11:    if (x > 0) {
        7:   12:        return 1;
        -:   13:    } else if (x == 0) {
branch  0 taken 2 (fallthrough)    # x == 0 为 true
branch  1 taken 1                  # x == 0 为 false
        2:   14:        return 0;
        1:   15:    } else {
        1:   16:        return -1;
        -:   17:    }
branch  0 taken 0                  # ← taken 0：该分支从未执行！
branch  1 taken 7
```

`taken 0` 就是测试盲区，说明这条路径从未被触发。

### lcov 分支统计

#### 为什么需要 `--rc branch_coverage=1`

lcov 默认**只收集行覆盖率和函数覆盖率，分支数据默认关闭**。官方手册的原文说明：

> *"Branch data is not collected or displayed by default"*
> *（`branch_coverage` 的默认值为 `0`）*

关闭的原因是性能考量——分支数据会显著增加内存占用、CPU 处理时间和 `.info` 文件体积，对于大型项目尤为明显。因此 lcov 将其设计为显式启用。

`--rc name=value` 是 lcov 通用的运行时配置覆盖机制，等价于临时修改配置文件中对应条目。`--rc branch_coverage=1` 就是把 `branch_coverage` 这个配置项从默认的 `0` 改为 `1`。

**版本差异：**

| 版本 | 旧写法（已废弃） | 新写法（推荐） | 等价命令行选项 |
|------|----------------|--------------|--------------|
| lcov 1.x | `--rc lcov_branch_coverage=1` | `--rc branch_coverage=1` | `--branch-coverage` |
| lcov 2.x | 同上，保持兼容 | `--rc branch_coverage=1` | `--branch-coverage` |

`--branch-coverage` 是 `--rc branch_coverage=1` 的命令行简写，两者完全等价——后者是为了与 `genhtml` 接口风格保持一致才加入的。**实际使用推荐统一用 `--branch-coverage`**，更简洁也不容易出错。

#### 三种启用方式对比

**方式 A：每条命令加 `--branch-coverage`（适合临时使用或脚本）**

```bash
lcov --capture --directory . -o coverage.info --branch-coverage
lcov --remove coverage.info '/usr/*' -o coverage_clean.info --branch-coverage
lcov --summary coverage_clean.info --branch-coverage
genhtml coverage_clean.info -o coverage_html/ --branch-coverage
```

**方式 B：每条命令加 `--rc branch_coverage=1`（与旧文档/CI 脚本保持一致）**

```bash
lcov --capture --directory . -o coverage.info --rc branch_coverage=1
lcov --remove coverage.info '/usr/*' -o coverage_clean.info --rc branch_coverage=1
lcov --summary coverage_clean.info --rc branch_coverage=1
genhtml coverage_clean.info -o coverage_html/ --branch-coverage   # genhtml 用 --branch-coverage
```

**方式 C：写入 `~/.lcovrc` 永久生效（适合个人开发环境）**

```ini
# ~/.lcovrc
branch_coverage = 1
```

写入后所有 lcov/genhtml 命令无需再加参数，直接生效。

> **注意**：三种方式中，`--branch-coverage` 和 `--rc branch_coverage=1` 对 `lcov` 完全等价；而 `genhtml` 只识别 `--branch-coverage`，不接受 `--rc branch_coverage=1`。

#### 完整的正确与错误对比

```bash
# ❌ 错误：不加任何分支参数，报告里 branches 行消失或为 0%
lcov --capture --directory . --output-file coverage.info
lcov --summary coverage.info

# ❌ 错误：collect 时加了，但 remove/summary 漏掉
#    .info 文件中的分支数据在 remove 步骤被丢弃
lcov --capture --directory . -o coverage.info --branch-coverage
lcov --remove coverage.info '/usr/*' -o coverage_clean.info   # ← 漏了！
lcov --summary coverage_clean.info                             # ← 漏了！

# ✅ 正确：每步都带上（推荐用 --branch-coverage）
lcov --capture --directory . -o coverage.info --branch-coverage
lcov --remove coverage.info '/usr/*' -o coverage_clean.info --branch-coverage
lcov --summary coverage_clean.info --branch-coverage
genhtml coverage_clean.info -o coverage_html/ --branch-coverage
```

### HTML 报告中的分支标记

`genhtml --branch-coverage` 会在每行左侧显示：

| 标记颜色 | 含义 |
|----------|------|
| 绿色数字 | 该分支已执行，数字为执行次数 |
| 红色 `0` | 该分支**从未执行**，是测试盲区 |
| 无标记 | 该行无分支 |

### 分支覆盖率的常见误区

**误区一：行覆盖 100% = 测试充分**

```c
int safe_div(int a, int b) {
    if (b != 0)          // 行被覆盖
        return a / b;
    return 0;
}

// 测试只写了 safe_div(10, 2)
// 行覆盖：100%（if 那行执行了）
// 分支覆盖：50%（b == 0 的分支从未触发）
```

**误区二：分支数量 = if 语句数 × 2**

实际上 `&&`、`||` 也会产生分支，三目运算符、`switch` 的每个 `case` 也各占一条。一个复杂表达式可能产生十几条分支。

**误区三：分支覆盖 100% 是可行目标**

对于有防御性编程（永远不应发生的错误路径）的代码，追求 100% 分支覆盖率可能得不偿失。通常将核心业务逻辑的分支覆盖率目标设为 80%+ 更务实。

### 崩溃场景完整示例

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>

extern void __gcov_dump(void);  /* GCC >= 11 */

static void flush_coverage(int sig) {
    __gcov_dump();
    fprintf(stderr, "Coverage flushed on signal %d\n", sig);
    _exit(128 + sig);
}

int main(void) {
    signal(SIGSEGV, flush_coverage);
    signal(SIGABRT, flush_coverage);
    signal(SIGTERM, flush_coverage);

    /* 业务代码 */
    return 0;
}
```

### 分支覆盖率一键脚本

```bash
#!/bin/bash
set -e

echo "=== GCC 分支覆盖率报告生成 ==="

# 清理旧的执行数据（.gcno 保留，无需重新编译）
find . -name "*.gcda" -delete
echo "✓ 清理旧数据"

# 编译（如果还没编译）
if [ ! -f my_program ]; then
    g++ --coverage -O0 -g -std=c++17 -o my_program main.cpp utils.cpp
    echo "✓ 编译完成"
fi

# 运行测试
./my_program
echo "✓ 测试运行完成"

# 收集（含分支）
lcov --capture \
     --directory . \
     --output-file coverage.info \
     --branch-coverage \
     --quiet

# 过滤系统文件
lcov --remove coverage.info '/usr/*' '*/test/*' \
     --output-file coverage_clean.info \
     --branch-coverage \
     --quiet

# 打印摘要
echo ""
echo "=== 覆盖率摘要 ==="
lcov --summary coverage_clean.info --branch-coverage

# 生成 HTML
genhtml coverage_clean.info \
        --output-directory coverage_html/ \
        --branch-coverage \
        --highlight \
        --legend \
        --quiet

echo ""
echo "✓ HTML 报告：coverage_html/index.html"
```

---

## Clang/LLVM 覆盖率体系

Clang 提供两种覆盖率模式，按需选择：

| 模式 | 标志 | 中间文件 | 工具链 | 适用场景 |
|------|------|----------|--------|----------|
| **gcov 兼容模式** | `--coverage` | `.gcno` + `.gcda` | lcov + genhtml | 需与 GCC 工具链共存 |
| **Source-based（推荐）** | `-fprofile-instr-generate -fcoverage-mapping` | `.profraw` → `.profdata` | llvm-profdata + llvm-cov | 纯 Clang 项目，精度最高 |

### 模式一：gcov 兼容模式

与 GCC 流程完全相同，直接替换编译器即可：

```bash
clang   --coverage -O0 -g -o my_program my_program.c
clang++ --coverage -O0 -g -std=c++17 -o my_app main.cpp utils.cpp

./my_program

lcov --capture --directory . --output-file coverage.info --branch-coverage
genhtml coverage.info --output-directory coverage_html/ --branch-coverage
```

### 模式二：Source-based Coverage（推荐）

#### 工具链数据流

```
源代码
  │
  └─[编译: -fprofile-instr-generate -fcoverage-mapping]
       │
       ▼
  插桩后的二进制（映射信息嵌入 binary 内）
       │
       └─[运行]──► default.profraw（原始计数）
                       │
              llvm-profdata merge
                       │
                       ▼
                 coverage.profdata（合并后）
                       │
              ┌────────┴────────┐
              ▼                 ▼
         llvm-cov show     llvm-cov export
         HTML / 文本         lcov 格式
```

#### 编译设置

```bash
# C
clang -fprofile-instr-generate -fcoverage-mapping -O0 -g \
      -o my_program my_program.c

# C++
clang++ -fprofile-instr-generate -fcoverage-mapping -O0 -g \
        -std=c++17 -o my_app main.cpp utils.cpp
```

| 选项 | 作用 |
|------|------|
| `-fprofile-instr-generate` | 插入计数器，运行后输出 `.profraw` |
| `-fcoverage-mapping` | 将源码到计数器的映射嵌入二进制 |
| `-O0` | 禁用优化 |
| `-g` | 保留调试信息 |

#### 运行测试

```bash
# 默认输出 default.profraw
./my_program

# 自定义路径（推荐：%p = 进程ID，%m = 二进制哈希，避免多进程互相覆盖）
LLVM_PROFILE_FILE="coverage-%p-%m.profraw" ./my_program

# 多场景分别收集
LLVM_PROFILE_FILE="test1.profraw" ./my_program --scenario a
LLVM_PROFILE_FILE="test2.profraw" ./my_program --scenario b
```

程序崩溃时可手动刷写：
```c
extern int __llvm_profile_write_file(void);
// 在信号处理函数中调用
__llvm_profile_write_file();
```

#### 生成报告

**步骤 1：合并原始数据**

```bash
llvm-profdata merge -sparse default.profraw -o coverage.profdata

# 合并多个文件
llvm-profdata merge -sparse test1.profraw test2.profraw -o coverage.profdata

# 通配符
llvm-profdata merge -sparse *.profraw -o coverage.profdata
```

**步骤 2：生成报告**

```bash
# 终端摘要
llvm-cov report ./my_program \
         -instr-profile=coverage.profdata

# HTML 报告
llvm-cov show ./my_program \
         -instr-profile=coverage.profdata \
         -format=html \
         -output-dir=coverage_html/ \
         -show-line-counts-or-regions \
         -show-instantiations

# 仅统计特定文件
llvm-cov report ./my_program \
         -instr-profile=coverage.profdata \
         src/my_module.cpp
```

**llvm-cov report 输出示例：**
```
Filename         Regions  Missed  Cover    Functions  Missed  Cover    Lines  Missed  Cover
--------------------------------------------------------------------------------------------
src/main.cpp          24       3  87.50%           6       0  100.00%     48       5  89.58%
src/utils.cpp         15       0  100.00%          4       0  100.00%     30       0  100.00%
--------------------------------------------------------------------------------------------
TOTAL                 39       3  92.31%          10       0  100.00%     78       5  93.59%
```

**导出 lcov 格式（对接现有工具）：**
```bash
llvm-cov export ./my_program \
         -instr-profile=coverage.profdata \
         -format=lcov > coverage.lcov

genhtml coverage.lcov --output-directory coverage_html/ --branch-coverage
```

**多目标文件（含共享库）：**
```bash
llvm-cov show ./my_program \
         -object ./libmy_lib.so \
         -instr-profile=coverage.profdata \
         -format=html \
         -output-dir=coverage_html/
```

### Clang 常见问题排查

| 现象 | 根因 | 解法 |
|------|------|------|
| `.profraw` 未生成 | 程序崩溃未刷写 | 调用 `__llvm_profile_write_file()` |
| `llvm-profdata: No such file` | 工具不在 PATH | 用完整路径 `/usr/lib/llvm-15/bin/llvm-profdata` |
| 版本不匹配错误 | profdata 与 llvm-cov 版本不一致 | 确保 clang 和 llvm-cov 版本相同 |
| 覆盖率明显失真 | 开启了优化 | 加 `-O0` |
| 多进程 profraw 互相覆盖 | 默认文件名冲突 | `LLVM_PROFILE_FILE="test-%p.profraw"` |

---

## 两大体系对比

| 维度 | GCC / gcov | Clang / llvm-cov |
|------|-----------|-----------------|
| **覆盖粒度** | 行、函数、分支 | 行、函数、分支、**区域（Region）** |
| **精度** | 中 | 高（宏内部、模板实例各自统计） |
| **中间文件** | `.gcno` + `.gcda` | `.profraw` → `.profdata` |
| **HTML 报告** | 依赖外部 genhtml | llvm-cov 内置生成 |
| **lcov 兼容** | 原生 | 需 `llvm-cov export -format=lcov` 转换 |
| **多线程安全** | 需加 `-fprofile-update=atomic` | 内置支持 |
| **跨语言** | C / C++ / Fortran | C / C++ / ObjC / Swift |
| **工具成熟度** | 极成熟，生态广泛 | 较新，功能持续增强 |
| **适合场景** | 传统项目、兼容性优先 | 现代项目、精度优先 |

---

## 与 CMake 集成

### GCC

```cmake
# CMakeLists.txt
option(ENABLE_COVERAGE "Enable GCC code coverage" OFF)

if(ENABLE_COVERAGE)
    if(NOT CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        message(FATAL_ERROR "GCC coverage requires GCC compiler")
    endif()
    message(STATUS "GCC coverage enabled")
    add_compile_options(--coverage -O0 -g)
    add_link_options(--coverage)
endif()
```

```bash
cmake -B build -DENABLE_COVERAGE=ON
cmake --build build
cd build && ctest --output-on-failure
lcov --capture --directory . \
     --output-file coverage.info --branch-coverage
lcov --remove coverage.info '/usr/*' \
     --output-file coverage_clean.info --branch-coverage
genhtml coverage_clean.info \
        --output-directory coverage_html/ --branch-coverage
```

### Clang Source-based

```cmake
# CMakeLists.txt
option(ENABLE_COVERAGE "Enable Clang source-based coverage" OFF)

if(ENABLE_COVERAGE)
    if(NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        message(FATAL_ERROR "Source-based coverage requires Clang")
    endif()
    message(STATUS "Clang source-based coverage enabled")
    add_compile_options(-fprofile-instr-generate -fcoverage-mapping -O0 -g)
    add_link_options(-fprofile-instr-generate)

    find_program(LLVM_PROFDATA llvm-profdata REQUIRED)
    find_program(LLVM_COV      llvm-cov      REQUIRED)

    add_custom_target(coverage
        COMMAND ${CMAKE_COMMAND} -E env
                LLVM_PROFILE_FILE=${CMAKE_BINARY_DIR}/test-%p.profraw
                $<TARGET_FILE:my_tests>
        COMMAND ${LLVM_PROFDATA} merge -sparse
                ${CMAKE_BINARY_DIR}/test-*.profraw
                -o ${CMAKE_BINARY_DIR}/coverage.profdata
        COMMAND ${LLVM_COV} show $<TARGET_FILE:my_tests>
                -instr-profile=${CMAKE_BINARY_DIR}/coverage.profdata
                -format=html
                -output-dir=${CMAKE_BINARY_DIR}/coverage_html
        DEPENDS my_tests
        COMMENT "Generating Clang coverage report..."
    )
endif()
```

```bash
cmake -B build -DCMAKE_CXX_COMPILER=clang++ -DENABLE_COVERAGE=ON
cmake --build build
cmake --build build --target coverage
open build/coverage_html/index.html
```

---

## 与 Makefile 集成

```makefile
CXX    = g++
SRCS   = main.cpp utils.cpp
TARGET = my_program

$(TARGET): $(SRCS)
	$(CXX) -O2 -o $@ $^

# GCC 覆盖率（含分支）
coverage-gcc: clean-cov
	$(CXX) --coverage -O0 -g -o $(TARGET) $(SRCS)
	./$(TARGET)
	lcov --capture --directory . \
	     --output-file coverage.info --branch-coverage
	lcov --remove coverage.info '/usr/*' \
	     --output-file coverage_clean.info --branch-coverage
	lcov --summary coverage_clean.info --branch-coverage
	genhtml coverage_clean.info \
	        --output-directory coverage_html/ --branch-coverage
	@echo "报告：coverage_html/index.html"

# Clang Source-based 覆盖率
coverage-clang: clean-cov
	clang++ -fprofile-instr-generate -fcoverage-mapping -O0 -g \
	        -o $(TARGET) $(SRCS)
	LLVM_PROFILE_FILE="coverage.profraw" ./$(TARGET)
	llvm-profdata merge -sparse coverage.profraw -o coverage.profdata
	llvm-cov show ./$(TARGET) \
	         -instr-profile=coverage.profdata \
	         -format=html -output-dir=coverage_html/
	@echo "报告：coverage_html/index.html"

clean-cov:
	find . \( -name "*.gcda" -o -name "*.gcno" -o -name "*.gcov" \) -delete
	rm -f *.profraw *.profdata coverage.info coverage_clean.info
	rm -rf coverage_html/

.PHONY: coverage-gcc coverage-clang clean-cov
```

---

## CI/CD 集成实践

### GitHub Actions — GCC + lcov（含分支）

```yaml
# .github/workflows/coverage.yml
name: Code Coverage

on: [push, pull_request]

jobs:
  coverage-gcc:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install lcov
        run: sudo apt-get install -y lcov

      - name: Build with coverage
        run: |
          cmake -B build -DENABLE_COVERAGE=ON
          cmake --build build

      - name: Run tests
        run: cd build && ctest --output-on-failure

      - name: Collect coverage (with branches)
        run: |
          lcov --capture --directory build \
               --output-file coverage.info \
               --branch-coverage
          lcov --remove coverage.info '/usr/*' '*/test/*' \
               --output-file coverage_filtered.info \
               --branch-coverage
          lcov --summary coverage_filtered.info \
               --branch-coverage

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage_filtered.info
          fail_ci_if_error: true
```

### GitHub Actions — Clang Source-based

```yaml
name: Clang Coverage

on: [push, pull_request]

jobs:
  coverage-clang:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Clang & LLVM tools
        run: sudo apt-get install -y clang llvm

      - name: Build with coverage
        env:
          CC: clang
          CXX: clang++
        run: |
          cmake -B build -DENABLE_COVERAGE=ON
          cmake --build build

      - name: Run tests & collect coverage
        run: |
          LLVM_PROFILE_FILE="build/test-%p.profraw" build/my_tests
          llvm-profdata merge -sparse build/test-*.profraw \
                        -o build/coverage.profdata
          llvm-cov export build/my_tests \
                   -instr-profile=build/coverage.profdata \
                   -format=lcov > coverage.lcov

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage.lcov
```

---

## 快速命令速查

### 安装工具

```bash
# Ubuntu/Debian
sudo apt-get install gcc g++ lcov        # GCC 体系
sudo apt-get install clang llvm          # Clang 体系

# macOS
brew install lcov                         # GCC 体系（Xcode 自带 clang）

# 版本确认
gcov --version && lcov --version
llvm-profdata --version && llvm-cov --version
```

### GCC 覆盖率（含分支）

```bash
# 编译
g++ --coverage -O0 -g -o prog main.cpp

# 运行
./prog

# 收集（含分支）
lcov --capture --directory . -o cov.info --branch-coverage
lcov --remove cov.info '/usr/*' -o cov_clean.info --branch-coverage

# 摘要
lcov --summary cov_clean.info --branch-coverage

# HTML
genhtml cov_clean.info -o html/ --branch-coverage
```

### Clang Source-based 覆盖率

```bash
# 编译
clang++ -fprofile-instr-generate -fcoverage-mapping -O0 -g -o prog main.cpp

# 运行
LLVM_PROFILE_FILE="prog.profraw" ./prog

# 合并
llvm-profdata merge -sparse *.profraw -o cov.profdata

# 摘要
llvm-cov report ./prog -instr-profile=cov.profdata

# HTML
llvm-cov show ./prog -instr-profile=cov.profdata \
           -format=html -output-dir=html/

# 导出 lcov 格式
llvm-cov export ./prog -instr-profile=cov.profdata \
           -format=lcov > coverage.lcov
```

### 如何选择

```
使用 GCC 编译器？
  └─► gcc --coverage + lcov --branch-coverage + genhtml --branch-coverage

使用 Clang，需要兼容 lcov 工具链？
  └─► clang --coverage（流程同 GCC）

使用 Clang，追求最高覆盖率精度？
  └─► -fprofile-instr-generate -fcoverage-mapping
      → llvm-profdata merge → llvm-cov show/report
      （支持区域级覆盖，能区分宏展开与模板实例）
```

---

*版本 2.0 | GCC ≥ 9.x，Clang/LLVM ≥ 12.x*