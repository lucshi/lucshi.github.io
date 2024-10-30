---
layout: post
title: 为何选择轻量级 WebAssembly Runtime
categories: [wasm]
description: wamr initiate
keywords: wasm wamr
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---


#### 背景和动机
WebAssembly（Wasm）起初作为一种跨平台字节码语言被设计用于浏览器环境，能够将高性能的代码引入网页。然而，WebAssembly 的潜力远超浏览器，它也逐渐被用于嵌入式系统、云计算和边缘设备中。随着非浏览器环境对 WebAssembly 支持的需求增加，Bytecode Alliance 开发了 WebAssembly Micro Runtime（WAMR），一个为资源受限环境优化的 Wasm 运行时。

WAMR 的设计目标是尽可能减少内存和 CPU 占用，适用于 IoT 设备、微控制器和其他嵌入式应用。相比于其他 WebAssembly 运行时，WAMR 提供了更灵活的模块化架构，以适应不同硬件和操作系统的需求。

#### WAMR 的核心设计目标
WAMR 的设计目标可以归纳为以下几点：

1. **资源效率**：最大限度减少内存和计算资源使用，使得 WAMR 能够在 32 位 MCU（Microcontroller Units）甚至低于 1MB RAM 的设备上运行。
2. **多模式支持**：提供了解释模式、JIT（即时编译）模式、和 AOT（提前编译）模式，以平衡性能和资源使用。
3. **模块化架构**：用户可以裁剪或添加模块，从而定制化功能并优化内存使用。
4. **跨平台性**：支持多种操作系统，包括嵌入式系统、RTOS（实时操作系统）和裸机平台，且能在不同的硬件架构上运行。

#### WAMR 核心模块及功能

WAMR 的核心模块构成了它的整体架构，每个模块都被设计为独立的可选组件：

- **解释器**：这是 WAMR 的核心组件，负责按顺序解析和执行 WebAssembly 字节码。解释器模式提供了较低的启动时间，但运行速度较慢。
- **JIT 编译器**：在 JIT 模式下，WAMR 将字节码编译为本地机器码并缓存，以提升执行性能。该模式适合需要频繁执行的热点代码。
- **AOT 编译器**：AOT 编译器在部署前将 WebAssembly 模块编译为机器码，从而在运行时减少启动延迟。这对嵌入式系统和高性能环境尤其有用。
- **内存管理模块**：负责动态分配、管理和回收 WebAssembly 模块使用的内存，确保内存隔离和访问安全。
- **文件系统模块**：提供文件 I/O 支持，用于在运行时访问外部文件或资源。

#### WAMR 的架构示例
以下架构图展示了 WAMR 的核心模块及其交互：

```
 ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
 │   Wasm Module │       │   JIT Engine  │       │   AOT Engine  │
 └───────────────┘       └───────────────┘       └───────────────┘
          │                       │                       │
          ├────────────────────────┼──────────────────────┤
          │                        │                      │
 ┌────────▼──────────┐     ┌───────▼──────────┐     ┌────▼─────┐
 │  Interpreter Core │     │  Memory Manager  │     │  FS Mgr  │
 └────────┬──────────┘     └──────────────────┘     └──────────┘
          │
    ┌─────▼─────┐
    │  Runtime  │
    └───────────┘
```

#### WAMR 实际应用场景

1. **IoT 设备**：通过 WAMR，开发者可以在有限硬件上执行 Wasm 模块，实现动态更新、无缝扩展。
2. **边缘计算**：在边缘设备中运行 WAMR，可以减少对网络带宽的依赖，并通过本地计算提高响应速度。
3. **浏览器之外的 Wasm 执行**：在 CLI、桌面应用等场景中，WAMR 提供了灵活的 Wasm 运行时支持，允许开发者在非浏览器环境中运行 WebAssembly 模块。

#### 示例代码：WAMR 的基本初始化与运行

以下代码展示了 WAMR 的初始化和模块执行流程，从文件加载到函数调用，展示了 WAMR 如何通过解释器模式执行 WebAssembly 模块。

```c
#include "wasm_export.h"
#include <stdio.h>

// 错误缓冲区，用于存储加载和实例化时的错误信息
#define ERROR_BUF_SIZE 128

int main(int argc, char *argv[]) {
    // 初始化 WAMR 运行时
    if (!wasm_runtime_init()) {
        printf("Failed to initialize WAMR runtime.\n");
        return -1;
    }

    // 加载 WebAssembly 字节码
    const char *wasm_file = "example.wasm";
    uint8_t *wasm_bytes;
    uint32_t wasm_size = load_wasm_file(wasm_file, &wasm_bytes);
    
    if (wasm_size == 0) {
        printf("Failed to load wasm file.\n");
        return -1;
    }

    // 加载模块
    char error_buf[ERROR_BUF_SIZE];
    wasm_module_t module = wasm_runtime_load(wasm_bytes, wasm_size, error_buf, ERROR_BUF_SIZE);
    if (!module) {
        printf("Failed to load wasm module: %s\n", error_buf);
        return -1;
    }

    // 实例化模块
    wasm_module_inst_t module_inst = wasm_runtime_instantiate(module, 8192, 8192, error_buf, ERROR_BUF_SIZE);
    if (!module_inst) {
        printf("Failed to instantiate wasm module: %s\n", error_buf);
        wasm_runtime_unload(module);
        return -1;
    }

    // 查找函数并执行
    wasm_function_inst_t func = wasm_runtime_lookup_function(module_inst, "add", NULL);
    if (func) {
        int32_t params[2] = {5, 3}; // 两个参数，分别为5和3
        int32_t results[1];
        
        if (wasm_runtime_call_wasm(module_inst, NULL, func, 2, params, results)) {
            printf("Result of 'add': %d\n", results[0]);
        } else {
            printf("Function call failed.\n");
        }
    } else {
        printf("Function 'add' not found.\n");
    }

    // 清理资源
    wasm_runtime_deinstantiate(module_inst);
    wasm_runtime_unload(module);
    wasm_runtime_destroy();
    free(wasm_bytes);

    return 0;
}
```

**代码解释**：
1. **初始化 WAMR**：调用 `wasm_runtime_init` 初始化 WAMR 运行时环境。
2. **加载和实例化模块**：通过 `wasm_runtime_load` 加载 WebAssembly 模块，并通过 `wasm_runtime_instantiate` 创建模块实例。
3. **函数查找和调用**：使用 `wasm_runtime_lookup_function` 查找模块中的 `add` 函数，并通过 `wasm_runtime_call_wasm` 进行调用。此时我们传递了两个整数参数 `5` 和 `3`，并打印出返回值 `8`。
4. **资源释放**：释放实例化的模块和 WAMR 运行时资源。

#### 总结
WAMR 的轻量化设计、解释器和 JIT/AOT 模式支持，以及模块化架构，使其成为嵌入式和边缘设备的理想选择。今后我们将进一步探索 WAMR 的具体模块，如解释器核心设计、内存管理和安全性等，为开发者提供更深入的技术剖析和代码实现。