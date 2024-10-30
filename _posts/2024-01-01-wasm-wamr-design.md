---
layout: post
title: WAMR整体设计
categories: [basic]
description: wamr design
keywords: wasm wamr
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

#### WAMR 的核心模块
WAMR 是由几个核心模块组成的，每个模块都有自己的分工。下面是这些模块的简单介绍：

1. **解释器模块**：负责逐条执行 WebAssembly 的字节码。这个模式资源需求少，适合内存小、计算能力弱的设备。解释器是 WAMR 的基础模块，无论是 JIT 还是 AOT，都需要先有解释器的支持。

2. **JIT 编译器**：JIT 就是即时编译器，它会在运行过程中发现某些代码被频繁执行，就把这些“热点”代码编译成本地机器码，让执行速度快很多。JIT 编译比解释执行性能更高，但需要更多内存。

3. **AOT 编译器**：AOT（提前编译）模式则是提前把 WebAssembly 模块直接编译成目标机器码，这样在运行时不需要解释执行了。AOT 让应用启动得更快，适合启动时间要求严格的环境。

4. **内存管理模块**：负责管理 WebAssembly 模块的内存分配和回收，确保模块之间的内存隔离，防止越界访问等潜在的安全问题。

5. **文件系统模块**：为运行时提供文件访问功能。这个模块在运行时需要读取外部文件或资源时就能派上用场。

6. **API 支持模块**：这个模块为 WebAssembly 模块提供访问宿主系统的接口，比如文件系统、网络、硬件接口等。这让 WebAssembly 能和宿主环境更好地交互。

#### 模块结构与调用流程
下面是 WAMR 的模块结构图。每个模块通过 `Runtime Core` 来协作，`Runtime Core` 就像一个“中枢神经”，让各个模块彼此配合、互相调用。

```
 ┌───────────────────────────┐
 │       Wasm Module         │
 └───────────────────────────┘
             │
 ┌───────────┴───────────┐
 │    Runtime Core       │
 └───────────┬───────────┘
             │
 ┌───────────┼───────────┐
 │           │           │
 │   Interpreter         │
 │     Module            │
 │                       │
 │  JIT        AOT       │
 │ Engine    Engine      │
 │                       │
 └───────────┬───────────┘
             │
 ┌───────────┼───────────┐
 │           │           │
 │   Memory  │   File    │
 │  Manager  │  System   │
 │           │   Module  │
 └───────────┴───────────┘
```

#### 各模块的具体作用

1. **解释器模块**：这是整个 WAMR 的“基础模块”，负责逐条解码并执行 WebAssembly 字节码。这种方式启动快，比较省内存，但性能相对较低。

2. **JIT 编译器**：JIT 编译器负责“即时”把热点代码块（就是那些被频繁执行的部分）翻译成本地机器码，以提高执行效率。JIT 编译的流程大致是：
   - **热点检测**：计数器用来识别频繁执行的代码块。
   - **即时编译**：将热点代码块编译成机器码并缓存。
   - **直接执行**：缓存好机器码后再执行，不用重复编译。

3. **AOT 编译器**：在部署前就把 WebAssembly 模块编译成机器码，这样运行时直接执行编译好的代码，不用解释执行了。AOT 编译可以显著缩短启动时间，提高运行效率。

4. **内存管理模块**：负责运行时的堆和栈内存管理，还带有垃圾回收功能，保证了内存隔离，防止 WebAssembly 模块之间的内存冲突和潜在的安全问题。

5. **文件系统模块**：为外部文件访问提供接口，WebAssembly 模块可以通过它访问资源，特别适合需要外部资源的场景。

6. **API 支持模块**：为 WebAssembly 模块提供访问宿主环境的接口，如文件、网络等，让 WebAssembly 和宿主平台间的交互更加顺畅。


#### 示例代码：WAMR 初始化与模块加载

为了更好地理解 WAMR 的架构，我们来看一个简单的示例代码，展示如何在 C 程序中初始化 WAMR、加载 WebAssembly 模块并执行模块中的函数。这个流程基本上涉及 WAMR 中多个核心模块的协作。

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

    // 查找并执行函数
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

#### 代码说明

1. **初始化 WAMR 运行时**：通过 `wasm_runtime_init` 启动 WAMR。该方法会初始化内存管理模块，为后续操作做准备。
2. **加载 WebAssembly 字节码**：通过 `load_wasm_file` 加载字节码文件 `example.wasm`，并将字节码存储在 `wasm_bytes` 中。
3. **加载模块**：调用 `wasm_runtime_load` 将字节码加载为一个模块（module），这个模块还未准备好执行，需要进一步实例化。
4. **实例化模块**：通过 `wasm_runtime_instantiate` 创建模块实例（module instance），在实例化过程中分配栈和堆内存，建立与宿主平台的 API 接口。
5. **函数查找与调用**：使用 `wasm_runtime_lookup_function` 查找名为 `add` 的函数，并使用 `wasm_runtime_call_wasm` 进行调用，这里传递两个参数 `5` 和 `3`，返回值存储在 `results` 中。
6. **清理资源**：模块执行完成后，清理资源并释放内存。

#### 总结

WAMR 的模块化架构设计非常灵活，它不仅适用于资源受限的嵌入式环境，还适合需要高性能的场景。在这种架构下，不同模块之间通过 `Runtime Core` 协同工作，每个模块独立负责特定的功能，既确保了运行时的效率，又提高了易用性。通过这个代码示例，可以看出 WAMR 的初始化、加载和执行的基本流程，为后续进一步的模块解析打下了基础。