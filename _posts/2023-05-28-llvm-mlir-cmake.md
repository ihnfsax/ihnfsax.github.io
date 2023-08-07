---
title: "LLVM/MLIR CMake 相关命令解析"
date: 2023-05-28
categories:
  - Compiler
tags:
  - MLIR
  - LLVM
  - CMake
---

我们想要使用 LLVM 和 MLIR 的代码构建自己的项目时，通常会被 LLVM/MLIR 特殊的 CMake 命令搞得一头雾水。本文将对这些 LLVM/MLIR 内的特殊命令进行解析。

## `LLVM_DEFINITIONS` 变量

一个字符串，里面是形如 `-DXXX` 的编译选项（宏定义），按空格分隔。基于 LLVM 开发时，需要通过以下代码，将其添加到编译选项中：

```cmake
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})
```

## `MLIR_DIALECT_LIBS`, `MLIR_CONVERSION_LIBS`, `MLIR_TRANSLATION_LIBS` 属性

`GLOBAL` scope 的属性，都是列表属性，里面全是构建方言、转换、翻译要用到的库。通常，我们通过 `get_property()` 获取这些属性，以便后续使用。

```cmake
get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
get_property(conversion_libs GLOBAL PROPERTY MLIR_CONVERSION_LIBS)
get_property(translation_libs GLOBAL PROPERTY MLIR_TRANSLATION_LIBS)
```

这些库一般用于链接，直接传递给 `target_link_libraries()` 函数或通过 `LINK_LIBS` 传递给 `add_mlir_library()` 等函数。

## `LLVM_RUNTIME_OUTPUT_INTDIR` 和 `LLVM_LIBRARY_OUTPUT_INTDIR`

我们常在顶层 CMakeLists.txt 中看到：

```cmake
set(LLVM_RUNTIME_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/bin)
set(LLVM_LIBRARY_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/lib)
```

这是在为和 LLVM 相关的目标文件设置输出文件夹。使用 `add_clang_executable()`, `add_llvm_executable()`, `llvm_add_library()` 等函数，最终都会执行如下代码来设置输出目录：

```cmake
set_output_directory(${name} BINARY_DIR ${LLVM_RUNTIME_OUTPUT_INTDIR} LIBRARY_DIR ${LLVM_LIBRARY_OUTPUT_INTDIR})
```

但是，普通的 `add_library()` 和 `add_excutable()` 是不受影响的，需要手动设置：

```cmake
set_target_properties(${target} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set_target_properties(${target} PROPERTIES ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
```

## 解析 `add_mlir_dialect_library()` 函数

我们经常能看到如下的调用：

```cmake
add_mlir_dialect_library(MLIRXXXXDialect
  src_file1.cpp
  src_file2.cpp

  ADDITIONAL_HEADER_DIRS
  ${MLIR_MAIN_INCLUDE_DIR}/mlir/Dialect/XXXX

  DEPENDS
  MLIRXXXXOpsIncGen
  MLIRXXXXConversionsIncGen

  LINK_COMPONENTS
  BinaryFormat
  Core

  LINK_LIBS PUBLIC
  MLIRIR
  MLIRSupport
  )
```

在 `mlir/cmake/modules/AddMLIR.cmake` 中，定义了 `add_mlir_dialect_library()`, `add_mlir_conversion_library()`, `add_mlir_translation_library()` 三个函数，我们以 `add_mlir_dialect_library()` 为例，查看 MLIR CMake 如何添加方言库目标的。

```cmake
function(add_mlir_dialect_library name)
  set_property(GLOBAL APPEND PROPERTY MLIR_DIALECT_LIBS ${name})
  add_mlir_library(${ARGV} DEPENDS mlir-headers)
endfunction(add_mlir_dialect_library)
```

该函数第一步是将第一个实参添加到属性 `MLIR_DIALECT_LIBS` 中。由于在 MLIR 内部全部采用了这几个函数来添加库目标，属性 `MLIR_DIALECT_LIBS` 也就包含了 MLIR 所有的方言库。

第二步，调用 `add_mlir_library()` 函数，传递了所有实参，外加 `DEPENDS` 和 `mlir-headers` 两个参数。

在 `add_mlir_library()` 中，首先会解析参数：

```cmake
cmake_parse_arguments(ARG
    "SHARED;INSTALL_WITH_TOOLCHAIN;EXCLUDE_FROM_LIBMLIR;DISABLE_INSTALL;ENABLE_AGGREGATION"
    ""
    "ADDITIONAL_HEADERS;DEPENDS;LINK_COMPONENTS;LINK_LIBS"
    ${ARGN})
```

`ARG` 是前缀。第一个字符串解析为布尔型（选项型）变量，第二个字符串解析为单值变量，第三个字符串解析为列表变量。具体地，比如：

```cmake
add_mlir_library(arg1 DEPENDS hello world SHARED DEPENDS platform)
```

- `${ARG_SHARED}` 为 `ON`；
- `${ARG_DEPENDS}` 为 `hello;world;platform`。

### `ADDITIONAL_HEADER_DIRS`

通过一层一层查找，可以发现 `ADDITIONAL_HEADER_DIRS` 的使用过程：

```text
add_mlir_dialect_library
  └add_mlir_library
    └llvm_add_library
       └llvm_process_sources
```

首先，`ADDITIONAL_HEADER_DIRS` 后面一般跟的是方言对应的源码 include 目录，如果是官方方言，就会是 `${MLIR_MAIN_INCLUDE_DIR}/mlir/Dialect/XXXX`，其中变量 `MLIR_MAIN_INCLUDE_DIR` 表示 `llvm-project/mlir/include` 目录。如果是我们实现的方言，一般跟的是 `${CMAKE_CURRENT_SOURCE_DIR}/include/mlir/Dialect/XXXX`。

查看 `llvm_add_library()` 函数，发现 `ADDITIONAL_HEADER_DIRS` 指示目录里的头文件会一起传递给 `add_library()`。**给 `add_library()` 传递头文件不是必须的，似乎只是为了某些 IDE 能更好识别这些头文件。**

### `DEPENDS`

在 `add_mlir_library()` 中，`DEPENDS` 首先会 APPEND 一个目标 `mlir-generic-headers`，然后传递给 `llvm_add_library()`。

最终，因为 `add_mlir_library()` 强制将 `OBJECT` 选项传递给 `llvm_add_library()`，最后会运行如下代码：

```cmake
if(ARG_DEPENDS)
  add_dependencies(${obj_name} ${ARG_DEPENDS})
endif()
```

`add_dependencies()` 的作用是：确保一个顶层目标在依赖的所有目标构建完了后再构建。所以，`DEPENDS` 后面跟的一般是 TableGen 有关的目标，因为源码编译需要 TableGen 生成的 `.inc` 文件。

### `LINK_COMPONENTS`

`LINK_COMPONENTS` 在 `add_mlir_library()` 中默认 APPEND `Support`，然后传递给 `llvm_add_library()`。

查看源码，发现 `ARG_LINK_COMPONENTS` 的使用过程：

```cmake
# llvm_add_library()
llvm_map_components_to_libnames(llvm_libs
    ${ARG_LINK_COMPONENTS}
    ${LLVM_LINK_COMPONENTS}
    )

target_link_libraries(${name} ${libtype}
    ${ARG_LINK_LIBS}
    ${lib_deps}
    ${llvm_libs}
    )
```

可以看到，说明 `LINK_COMPONENTS` 里的组件名会被转换为对应的库名，然后传递给 `target_link_libraries()`。比如将 `Core` 组件名转换为 `LLVMCore` 库名。

查看所有组件名：

```bash
llvm-config --components
```

### `LINK_LIBS PUBLIC`

这个就比较简单了，主要做了两件事：

- `add_dependencies(${obj_name} ${link_libs})`：将 `LINK_LIBS` 添加到依赖里。
- `target_link_libraries(${name} ${libtype} ${ARG_LINK_LIBS} ${lib_deps} ${llvm_libs})`：添加为要链接的库。

## TableGen 相关命令解析

我们可以通过 [tblgen](https://llvm.org/docs/CommandGuide/tblgen.html) 工具来看可以进行哪些 TableGen 生成，比如：

```bash
mlir-tblgen -help
# Output:
# ...
# --gen-op-decls
# --gen-op-defs
# ...
```

在 CMake 代码中，我们经常能见到这样的代码：

```cmake
set(LLVM_TARGET_DEFINITIONS GPUBase.td)
mlir_tablegen(GPUOpInterfaces.h.inc -gen-op-interface-decls)
mlir_tablegen(GPUOpInterfaces.cpp.inc -gen-op-interface-defs)
add_public_tablegen_target(MLIRGPUOpInterfacesIncGen)
```

其中，设置 `LLVM_TARGET_DEFINITIONS` 是为了运行 `mlir-tblgen` 命令时能找到 `GPUBase.td` 文件。

`add_public_tablegen_target` 就是添加一个目标。

比较难理解的是如下的代码：

```cmake
add_mlir_dialect(GPUOps gpu)
add_mlir_doc(GPUOps GPUOps Dialects/ -gen-op-doc)
```

其中，`add_mlir_dialect()` 函数定义如下：

```cmake
function(add_mlir_dialect dialect dialect_namespace)
  set(LLVM_TARGET_DEFINITIONS ${dialect}.td)
  mlir_tablegen(${dialect}.h.inc -gen-op-decls)
  mlir_tablegen(${dialect}.cpp.inc -gen-op-defs)
  mlir_tablegen(${dialect}Types.h.inc -gen-typedef-decls -typedefs-dialect=${dialect_namespace})
  mlir_tablegen(${dialect}Types.cpp.inc -gen-typedef-defs -typedefs-dialect=${dialect_namespace})
  mlir_tablegen(${dialect}Dialect.h.inc -gen-dialect-decls -dialect=${dialect_namespace})
  mlir_tablegen(${dialect}Dialect.cpp.inc -gen-dialect-defs -dialect=${dialect_namespace})
  add_public_tablegen_target(MLIR${dialect}IncGen)
  add_dependencies(mlir-headers MLIR${dialect}IncGen)
endfunction()
```

一些方言会用 `add_mlir_dialect()` 函数替代上面一大串代码。

类似的还有 `add_mlir_interface()` 函数：

```cmake
function(add_mlir_interface interface)
  set(LLVM_TARGET_DEFINITIONS ${interface}.td)
  mlir_tablegen(${interface}.h.inc -gen-op-interface-decls)
  mlir_tablegen(${interface}.cpp.inc -gen-op-interface-defs)
  add_public_tablegen_target(MLIR${interface}IncGen)
  add_dependencies(mlir-generic-headers MLIR${interface}IncGen)
endfunction()
```

`add_mlir_doc()` 函数：

```cmake
function(add_mlir_doc doc_filename output_file output_directory command)
  set(LLVM_TARGET_DEFINITIONS ${doc_filename}.td)
  tablegen(MLIR ${output_file}.md ${command} ${ARGN})
  set(GEN_DOC_FILE ${MLIR_BINARY_DIR}/docs/${output_directory}${output_file}.md)
  add_custom_command(
          OUTPUT ${GEN_DOC_FILE}
          COMMAND ${CMAKE_COMMAND} -E copy
                  ${CMAKE_CURRENT_BINARY_DIR}/${output_file}.md
                  ${GEN_DOC_FILE}
          DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${output_file}.md)
  add_custom_target(${output_file}DocGen DEPENDS ${GEN_DOC_FILE})
  add_dependencies(mlir-doc ${output_file}DocGen)
endfunction()
```
