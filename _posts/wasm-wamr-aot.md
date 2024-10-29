---
layout: post
title: WAMR AOT 设计
categories: [wasm]
description: wamr aot
keywords: wasm wamr aot
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false

---

AOT（Ahead-Of-Time）编译是一种在部署前将代码提前编译成机器码的技术。对于 WAMR 来说，AOT 编译器将 WebAssembly 字节码转换成可直接执行的本地机器码，从而省去了运行时解释的开销。这种模式特别适合启动时间要求高且需要高性能的场景，如嵌入式系统、IoT 设备等。

#### AOT 编译的主要步骤

WAMR 的 AOT 编译器包含几个主要步骤：

1. **字节码解析**：分析和解析 WebAssembly 字节码，提取模块结构和函数内容。
2. **机器码生成**：将字节码转换为本地机器码，具体到目标平台的汇编指令。
3. **代码优化**：在生成机器码过程中进行多种优化，例如常量折叠、循环展开等，以提高执行效率。
4. **二进制文件输出**：将生成的机器码存储为二进制文件，便于在不同平台直接加载和执行。

#### AOT 编译示例流程

下面是 WAMR AOT 编译器执行的一个示例流程，展示如何将简单的 `i32.add` 和 `i32.const` 操作编译成机器码。

##### 步骤 1：字节码解析

首先，AOT 编译器会分析字节码，提取模块和函数的结构。假设我们有一个包含 `i32.add` 和 `i32.const` 的简单 WebAssembly 函数，解析过程会提取指令流。

```wasm
(i32.const 5)
(i32.const 3)
(i32.add)
```

##### 步骤 2：生成机器码

解析完成后，编译器将字节码逐一转化为相应的机器码。以 x86-64 为例，AOT 编译器将 `i32.const 5`、`i32.const 3` 和 `i32.add` 依次转化为机器码。

```c
void aot_generate_code(aot_block_t *block) {
    // 编译 `i32.const 5`
    *block->machine_code++ = 0xB8; // "mov eax, imm" 指令操作码
    *((int32_t *)block->machine_code) = 5;
    block->machine_code += sizeof(int32_t);

    // 编译 `i32.const 3`
    *block->machine_code++ = 0xBB; // "mov ebx, imm" 指令操作码
    *((int32_t *)block->machine_code) = 3;
    block->machine_code += sizeof(int32_t);

    // 编译 `i32.add`
    *block->machine_code++ = 0x01; // "add eax, ebx" 指令操作码
    *block->machine_code++ = 0xD8;
}
```

在这个例子中：

- `i32.const 5` 被编译为 `mov eax, 5`
- `i32.const 3` 被编译为 `mov ebx, 3`
- `i32.add` 被编译为 `add eax, ebx`

生成的机器码可以直接在目标平台上运行，无需再次解释。

##### 步骤 3：代码优化

在机器码生成过程中，AOT 编译器可以应用多种优化，例如：

- **常量折叠**：在生成机器码时，将常量计算提前完成，直接用结果替换。
- **指令合并**：将多个相邻指令合并为单个高效的指令。
- **死代码消除**：去掉不会被执行的代码。

以下是一个示例代码，展示了常量折叠和指令合并：

```c
// 示例：常量相加的指令合并
void optimize_add_constant(aot_block_t *block) {
    int32_t lhs = 5;
    int32_t rhs = 3;
    int32_t result = lhs + rhs;

    // 直接生成 mov eax, result
    *block->machine_code++ = 0xB8;
    *((int32_t *)block->machine_code) = result;
    block->machine_code += sizeof(int32_t);
}
```

这样，在遇到常量相加时，AOT 编译器直接将计算结果写入机器码，避免了运行时再执行。

##### 步骤 4：二进制文件输出

生成的机器码可以保存为二进制文件，便于部署在不同平台上。WAMR 的 AOT 模块提供 API，可以直接加载和执行这个二进制文件。

```c
void save_aot_binary(aot_block_t *block, const char *filename) {
    FILE *file = fopen(filename, "wb");
    fwrite(block->machine_code, 1, block->code_size, file);
    fclose(file);
}
```

保存后的二进制文件可以在运行时加载，无需重复编译，从而大幅减少启动时间。

#### AOT 优化策略

WAMR 的 AOT 编译器应用了多种优化策略，提升了生成代码的效率：

1. **常量折叠**：直接计算常量结果，减少运行时的计算负担。
2. **死代码消除**：去掉无用的代码，精简机器码体积。
3. **循环展开**：针对小规模循环，直接展开循环体，减少循环跳转的开销。

#### 小结

WAMR AOT 编译器通过将字节码提前编译为本地机器码，极大缩短了启动时间，并提高了执行效率。它的优化策略，特别是常量折叠和指令合并，使生成的机器码更加高效。这种编译方式特别适合资源受限的设备，在保证运行速度的同时，减轻了运行时的内存占用。