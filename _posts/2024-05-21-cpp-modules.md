---
title: "C++20 Module 机制详解"
date: 2024-05-21
categories:
  - C++
tags:
  - C++20
---

Module 是 C++20 的新特性，用于替换头文件的功能。C++ 经常需要在多个翻译单元间共享声明和定义，过去使用头文件做到这一点，而 module 是一个实现该功能全新机制。

> 1. Module 和命名空间是正交的。
> 2. 对于 clang 用户而言，module 一词有歧义，它可能指：Objective-C Module, [Clang Module](https://clang.llvm.org/docs/Modules.html) (Clang Header Module), C++20 Module (Standard C++ Module)。

## 名词解释

### Module 和 Module Unit

为了方便表述，我们先引入几个名词。

一个 module 包含一个或多个 module unit。Module unit 是一种特殊的翻译单元。一个 module unit 总是以 **module declaration** 开始：

```c++
[export] module module_name[:partition_name];
```

`[]` 中的内容是可选的。`module_name` 和 `paritition_name` 遵守 C++ 标识符规则，此外还能包含 period 符号 (`.`)，但是注意，符号 `.` 并不包含语义上的特殊含义。

在本文中，module units 分为四大类，按 module declaration 进行区分：

| Module Unit Category            | Module Declaration                          | 说明                                   |
|---------------------------------|---------------------------------------------|----------------------------------------|
| primary module interface unit   | `export module module_name;`                | 一个 module 中只有一个                 |
| module implementation unit      | `module module_name;`                       | 一个 module 中可以有多个               |
| module partition interface unit | `export module module_name:partition_name;` | `partition_name` 必须在 module 内唯一  |
| internal module partition unit  | `module module_name:partition_name;`        | `partition_name` 必须在 module 内唯一  |

> 注意，单独的 `module;` 并不是 module declaration。

还有 module interface unit（包含 `export` 的 module declaration）, importable module unit, module partition unit 三个名词，它们的关系是：

<img src="{{ "/assets/images/cpp-modules/terms.png" | relative_url }}" width=550 alt="terms" />

importable module unit 文件的后缀是 C++ 文件名加上 m (.cppm/.ccm/.cxxm/.c++m)，而 module implementation unit 文件使用普通 C++ 文件名 (.cpp/.cc/.cxx/.c++)。Clang 默认不支持将其他后缀名的 importable module unit 文件，除非显式指定 `-x c++-module` 选项。

### `export` 和 `import`

Module interface units 可以导出声明（以及定义），可以被其他翻译单元导入。下面是一些例子：

```c++
export module A; // declares the primary module interface unit for named module 'A'
 
// hello() will be visible by translations units importing 'A'
export char const* hello() { return "hello"; } 
 
// world() will NOT be visible.
char const* world() { return "world"; }
 
// Both one() and zero() will be visible.
export
{
    int one()  { return 1; }
    int zero() { return 0; }
}
 
// Exporting namespaces also works: hi::english() and hi::french() will be visible.
export namespace hi
{
    char const* english() { return "Hi!"; }
    char const* french()  { return "Salut!"; }
}
```

我们还会见到 `export import ...` 这样的用法，它的意思是：如果 module B export-import A, 则 import B 也将使 A 的所有 export 内容可见。下面是一个例子：

```c++
/////// A.cpp (primary module interface unit of 'A')
export module A;
 
export char const* hello() { return "hello"; }
 
/////// B.cpp (primary module interface unit of 'B')
export module B;
 
export import A;
 
export char const* world() { return "world"; }
 
/////// main.cpp (not a module unit)
#include <iostream>
import B;
 
int main()
{
    std::cout << hello() << ' ' << world() << '\n';
}
```

### Header Unit

头文件可以作为 header unit 被 import declaration 导入。导入后，头文件的所有声明、定义，以及预处理宏都变得可用：

```c++
[export] import header-name
```

但和直接 `#include` 不同的是，import header unit 前已存在的预处理宏不会对该头文件的预处理产生影响。想要利用现有预处理宏，需要使用另一个使用头文件的机制：Global module fragment。

下面是 header unit 的例子：

```c++
/////// A.cpp (primary module interface unit of 'A')
export module A;
 
import <iostream>;
export import <string_view>;
 
export void print(std::string_view message)
{
    std::cout << message << std::endl;
}
 
/////// main.cpp (not a module unit)
import A;
 
int main()
{
    std::string_view message = "Hello, world!";
    print(message);
}
```

### Built Module Interface

一个 Built Module Interface (BMI) 是一个 importable module unit 的预编译结果。BMI 文件后缀是 `.pcm`。生成的 BMI 文件名应该遵守如下规则：

- primary module interface unit: `module_name.pcm`
- module partition unit: `module_name-partition_name.pcm`

我们不应随意修改 BMI 文件的名称，它们对 clang 如何正确找到位置非常重要。

### Global module fragment

Global module fragment (GMF) 是在一个 module unit 内，`module;` 和 module declaration 之间的部分：

```c++
module;
preprocessing-directives (optional)
module-declaration
```

GMF 只能包含预处理指令。我们不允许在 GMF 之外的位置进行 `#include` 操作，因为这样头文件内容就会被当作 module 的一部分，而不是只被 module 使用。

GMF 相比 header unit 的好处是，它支持利用现有的预处理宏对预处理行为产生影响，这在某些依赖宏定义进行配置的头文件中非常重要。

```c++
/////// A.cpp (primary module interface unit of 'A')
module;
#define _POSIX_C_SOURCE 200809L
#include <stdlib.h>
export module A;

import <ctime>; // Header unit

export double weak_random()
{
    std::timespec ts;
    std::timespec_get(&ts, TIME_UTC); // from <ctime>
    srand48(ts.tv_nsec);
    return drand48();
}

/////// main.cpp (not a module unit)
import <iostream>;
import A;

int main()
{
    std::cout << "Random value between 0 and 1: " << weak_random() << '\n';
}
```

## 使用 modules 构建项目

### 完整例子

下面是一个包含四种 module unit 的例子：

```c++
// Hello.cppm
module;
#include <iostream>
export module Hello;
export void hello() {
  std::cout << "Hello World!\n";
}

// use.cpp
import Hello;
int main() {
  hello();
  return 0;
}
```

`Hello.cppm` 是一个 primary module interface unit，而 `use.cpp` 是 module 的使用者，并不是一个 module unit。

编译命令时，使用 `-std=c++20` 选项启用 module：

```bash
$ clang++ -std=c++20 Hello.cppm --precompile -o Hello.pcm
$ clang++ -std=c++20 use.cpp -fmodule-file=Hello=Hello.pcm Hello.pcm -o Hello.out
$ ./Hello.out
Hello World!
```

下面的例子包含了四种不同的 module unit:

```c++
// M.cppm
export module M;
export import :interface_part;
import :impl_part;
export void Hello();

// interface_part.cppm
export module M:interface_part;
export void World();

// impl_part.cppm
module;
#include <iostream>
#include <string>
module M:impl_part;
import :interface_part;

std::string W = "World.";
void World() {
  std::cout << W << std::endl;
}

// Impl.cpp
module;
#include <iostream>
module M;
void Hello() {
  std::cout << "Hello ";
}

// User.cpp
import M;
int main() {
  Hello();
  World();
  return 0;
}
```

注意，这里的 module partition interface unit 和 internal module partition unit 没有使用相同的 partition 名称。

编译命令：

```bash
# Precompiling the module
$ clang++ -std=c++20 interface_part.cppm --precompile -o M-interface_part.pcm
$ clang++ -std=c++20 impl_part.cppm --precompile -fprebuilt-module-path=. -o M-impl_part.pcm
$ clang++ -std=c++20 M.cppm --precompile -fprebuilt-module-path=. -o M.pcm
$ clang++ -std=c++20 Impl.cpp -fprebuilt-module-path=. -c -o Impl.o

# Compiling the user
$ clang++ -std=c++20 User.cpp -fprebuilt-module-path=. -c -o User.o

# Compiling the module and linking it together
$ clang++ -std=c++20 M-interface_part.pcm -fprebuilt-module-path=. -c -o M-interface_part.o
$ clang++ -std=c++20 M-impl_part.pcm -fprebuilt-module-path=. -c -o M-impl_part.o
$ clang++ -std=c++20 M.pcm -fprebuilt-module-path=. -c -o M.o
$ clang++ User.o M-interface_part.o  M-impl_part.o M.o Impl.o -o a.out
```

在 Golang 中，同一个 package 内不同文件中的函数是可以互相调用的，而 C++ module 始终遵循先有声明才能使用的规则。假设现在有另一个 module implementation unit 文件 `Impl2.cpp`，定义了 `Test()` 函数，那么 `Impl.cpp` 想要使用它，最好的做法是在 primary module interface unit 文件 `M.cppm` 声明 `Test()` 函数（不需要导出），就好似 `Impl.cpp` 和 `Impl2.cpp` 都 include 了 `M.cppm` 一样。我们后面会讲到，module implementation unit 实际隐式地 import 了 primary module interface unit。

### 生成 BMI

可以使用 `--precompile` 和 `-fmodule-output` 两种选项生成 importable module unit 对应的 BMI 文件。

就像前面所展示的，`--precompile` 选项将 BMI 看作编译产物，所以输出路径也是由 `-o` 选项来指定。

`-fmodule-output` 选项不同，它将 BMI 作为编译的副产物生成。当 `-fmodule-output=` 被指定时，BMI 生成到等号后的路径，否则生成到 `.o` 文件相同的目录，文件名前缀和 `.cppm` 的前缀一样。

`--precompile` 可以看作将 module unit 编译为目标文件的第一步：生成 BMI，而 `--fmodule-output` 则是两步一次性做完：

```bash
# 第一步：
$ clang++ -std=c++20 Hello.cppm --precompile -o Hello.pcm
# 第二步：
$ clang++ -std=c++20 Hello.pcm -c -o Hello.o
# 如果 Hello.cppm 用到了其他 module unit，还需要 -fprebuilt-module-path= 或 -fmodule-file=

# 一次性生成目标文件：
$ clang++ -std=c++20 -fmodule-output Hello.cppm -c -o Hello.o
```

`--precompile` 这种两阶段的编译方式有助于提高编译的并行度。

### 指定 BMI

我们已经见到了三种方式用来指定 BMI 文件以及对应的 module：

- `-fprebuilt-module-path=<path/to/directory>`.
- `-fmodule-file=<module-name>=<path/to/BMI>`.
- `-fmodule-file=<path/to/BMI>` (Deprecated).

`-fprebuilt-module-path=` 允许 clang 在指定目录寻找 `M.pcm` 和 `M-P.pcm`，其中 M 是 module 名称，P 是 partition 名称。

`-fmodule-file=<module-name>=<path/to/BMI>` 指明 module 名称和文件的对应关系，若要指定 module partition unit 的文件，使用 `M:P` 来表示 module partition 的名称。

前两个选项都支持 BMI 的 lazy load，而 `-fmodule-file=<path/to/BMI>` 不支持，这个选项未来会被 clang 取消。

还有两点需要注意：

- **当编译一个 module implementation unit 时，会隐式地 import 对应的 primary module interface unit**，所以必须在编译 module implementation unit 时，指定 primary module interface unit 对应的文件。
- 使用 `--precompile` 选项两阶段编译 module unit 时，在两个阶段都需要告诉编译器 module unit 文件在哪。

前面举了一个包含所有 module unit 类型的例子，它们对 `.pcm` 的依赖关系如下所示：

<img src="{{ "/assets/images/cpp-modules/dependency.png" | relative_url }}" width=500 alt="dependency" />

在 `Impl.cpp` 中，我们虽然没有显式地像 `User.cpp` 那样 `import M;`，但根据前面的规则，它实际隐式地 import 了 module `M`。

### 使用 CMake

一个例子：

```c++
// a.cppm
module;

#include <iostream>

export module MyModule;

int hidden() {
    return 42;
}

export void printMessage() {
    std::cout << "The hidden value is " << hidden() << "\n";
}
```

```c++
// main.cpp
import MyModule;

int main() {
    printMessage();
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.28)

project(modules-example)

set(CMAKE_CXX_STANDARD 20)

add_executable(demo)
target_sources(demo
    PUBLIC
    main.cpp
)
target_sources(demo
  PUBLIC
    FILE_SET all_my_modules TYPE CXX_MODULES FILES
    a.cppm
)
```

想要在 cmake 中 使用 c++ module，必须有 3.28 以上的 cmake 和 1.11 以上的 ninja。

使用如下命令生成构建文件：

```bash
CXX=clang++ cmake -S . -B build -G Ninja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

查看生成的 `compile_commands.json` 可以发现，实际使用的编译命令并未采用前面所说的 `-fprebuilt-module-path` 或 `-fmodule-file` 选项，而是引入了一个 modmap 文件。

截止到 2024 年 5 月，clangd 对 C++20 module 的支持依旧局限于 `-fprebuilt-module-path` 和 `-fmodule-file` 选项，无法从 cmake 生成的命令中找到 module 的位置。当前 clangd 的 module 支持的进度可关注[论坛](https://discourse.llvm.org/t/rfc-directions-for-modules-support-in-clangd/78072)和[仓库](https://github.com/clangd/clangd/issues/1293)。

## 参考资料

- [Standard C++ Modules - Clang Documentation](https://clang.llvm.org/docs/StandardCPlusPlusModules.html)
- [Modules - cppreference](https://en.cppreference.com/w/cpp/language/modules)
- [How to use c++20 modules with CMake? - Stack overflow](https://stackoverflow.com/q/57300495/12939471)
