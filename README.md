# SmallVMP: 一个基于 LLVM 的教学性 VMP 实现

**“不造轮子，何以知轮之精髓？” — To truly understand the wheel, you must build one.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 目录

- [关于项目](#关于项目)
- [核心特性](#核心特性)
- [工作原理](#工作原理)
- [环境准备](#环境准备)
- [如何使用](#如何使用)
- [效果展示](#效果展示)
- [局限性](#局限性)
- [未来展望](#未来展望)
- [致谢](#致谢)

## 关于项目

`SmallVMP` 是一个从零开始构建的、用于学习和研究的虚拟机保护（Virtual Machine Protection）项目。它并非一个生产级的安全解决方案，而是我个人在探索 VMP 技术原理过程中的产物。

项目的核心目标是：
1.  亲手实现一个微型虚拟机（VM），包括其指令集、解释器和字节码格式。
2.  利用 **LLVM Pass** 技术，在编译期间自动将被保护函数的 LLVM IR 转换为我们自定义的字节码。
3.  解决在 VMP 中处理外部函数调用和全局变量访问等关键问题。

本项目深度集成了强大的 **Hikari Obfuscator** 框架，不仅实现了基础的 VMP 功能，还能利用 Hikari 对 VMP 的核心——解释器本身——进行混淆，从而极大提升了分析难度。

## 核心特性

- **基于 LLVM Pass**: 在编译时自动完成虚拟化，对源代码无侵入。
- **自定义指令集 (ISA)**: 包含一套精简的 指令集，用于算术、逻辑、访存和控制流操作。
- **Thunk 机制**: 巧妙地通过生成原生“桥接函数 (Thunk)”来处理对外部函数和全局变量的引用，避免了手动模拟链接器的复杂性。
- **与 Hikari 深度集成**: 可以轻松地将 Hikari 的多种混淆策略（如 BCF, CFF, SPL, SUB）应用于 VMP 解释器和项目中的其他原生代码。
- **跨平台**: 作为 LLVM 的一部分，理论上支持所有 LLVM/Clang 支持的目标平台。

## 工作原理

`SmallVMP` 的保护流程可以概括为以下几个步骤：

1.  **标记**: 开发者使用 `IRVM_SECTION` 宏标记需要保护的 C/C++ 函数。
2.  **IR 转换 (LLVM Pass)**:
    - 在编译过程中，`SmallVMP` 的 Pass 会识别出被标记的函数。
    - 它遍历函数的 LLVM IR，并将其逐条翻译成我们自定义的字节码。
    - 所有对外部函数或全局变量的引用都会被记录下来，并为之生成一个原生代码的**桥接函数 (Thunk)**。字节码中将只保留对这个 Thunk 的调用 ID。
3.  **函数替换**:
    - 原始函数体被清空。
    - 替换为一个**跳板 (Stub)** 函数。这个 Stub 的唯一作用就是调用 VM 解释器 (`vm_exec`)，并将字节码、寄存器上下文等信息传递给它。
4.  **运行时执行**:
    - 当程序调用被保护的函数时，实际上是在调用这个 Stub。
    - Stub 启动 VM 解释器，解释器开始逐条执行字节码，模拟原始函数的逻辑。
    - 当遇到需要调用外部函数（如 `printf`）的指令时，解释器会通过 ID 查找到对应的 Thunk 函数并执行它。

## 环境准备

`SmallVMP` 是作为 LLVM 的一个 Pass 实现的，并依赖于特定的 Hikari 混淆框架。因此，您需要先编译和配置好集成了 `SmallVMP` 的 LLVM/Clang 工具链。

1.  **获取工具链**:
    - 克隆或下载集成了 `SmallVMP` 的 LLVM/Hikari 源码。
    - `[(https://github.com/NiTianErXing666/SmallVmp)]`
2.  **编译 LLVM/Clang**:
    - 根据该仓库的说明文档编译工具链。
3.  **配置环境变量**:
    - 将编译好的 `bin` 目录（包含 `clang`, `clang++` 等）添加到您的系统 `PATH` 中，以确保后续命令能调用这个定制版的编译器。

## 如何使用

使用 `SmallVMP` 非常简单：

1.  **引入头文件**: 在您的 C/C++ 代码中，包含 `VMP.h`。

    ```c
    #include "VMP.h"
    ```

2.  **标记目标函数**: 使用 `IRVM_SECTION` 宏来标记您希望进行 VMP 保护的函数。

    ```c
    // 示例代码，包含全局变量和外部函数调用
    int gArr[10] = {0};
    const char* gMsg = "Hello, VMP!";
    
    IRVM_SECTION
    int test_calls(int a, int b) {
        puts("call puts(const)");
        int *p = &gArr[4];
        *p = a + b;
        printf("gArr[4]=%d, gMsg=%s
    ", gArr[4], gMsg);
        return a + b;
    }
    ```

3.  **编译代码**: 使用定制的 `clang` 进行编译。`SmallVMP` 默认启用，无需额外参数。

    ```bash
    /path/to/your/custom/clang test.c -o app
    ```

    编译时，您会看到类似如下的日志，表明 `SmallVMP` 已成功处理目标函数：
    ```
    [irvm] emit code global: test_calls_code (266 bytes)
    [irvm]   + bytes initialized
    [irvm]   + rewritten to vm_exec stub: test_calls
    ```

4.  **(可选) 叠加 Hikari 混淆**: 您可以附加 Hikari 的 `-mllvm` 参数来混淆 VM 解释器本身，或项目中的其他原生代码，以达到双重保护。

    ```bash
    # 开启伪控制流 (Bogus Control Flow)
    clang test.c -o app -mllvm -enable-bcfobf
    
    # 开启控制流平坦化 (Control Flow Flattening)
    clang test.c -o app -mllvm -enable-cffobf
    
    # 开启所有混淆
    clang test.c -o app -mllvm -enable-allobf
    ```

## 效果展示

 可以去看雪论坛看效果：https://bbs.kanxue.com/thread-288435-1.html


##  局限性
SmallVMP 目前仍是一个实验性项目，存在一些局限：

指令支持不完整: 对某些复杂的 LLVM IR 指令（如 select, phi 的复杂用法等）尚未支持。为保证编译稳定性，遇到不支持的指令时，Pass 会自动放弃对该函数的虚拟化。
性能开销: 和所有 VMP方案一样，解释执行会带来显著的性能下降。不适合用于性能敏感的热点函数。
安全性: 字节码目前是明文存储的，容易被 Dump 和分析。
## 未来展望
字节码加密: 对生成的字节码进行加密，在解释器执行前动态解密，对抗静态分析。
动态分发器 (Dynamic Dispatcher): 使操作码 (Opcode) 与处理函数 (Handler) 的映射关系在运行时动态生成，对抗基于特征码的分析。
嵌套 VM (Nested VM): 实现二级 VM，用一个 VM 来保护另一个 VM 的解释器，形成层层嵌套，将分析难度提升到新的高度。
## 致谢
本项目深受 Hikari Obfuscator 项目的启发，并直接基于其框架进行构建。感谢 Hikari 团队的杰出工作。https://github.com/HikariObfuscator/Hikari
感谢 LLVM 社区提供的强大而灵活的编译器基础设施。
感谢 Xvmp 的思路！感谢 Xvmp 团队的杰出工作 https://github.com/GANGE666/xVMP
## 贡献
本项目旨在学习和交流，代码尚不完美，欢迎任何形式的贡献！如果您有任何建议、发现 Bug 或希望添加新功能，请随时提交 Pull Request 或创建 Issue。


