---
title: "Linux 的共享库搜索规则"
date: 2023-07-27
categories:
  - Compile and Link
tags:
  - GCC
  - ELF
---

我们在 Linux 平台上开发 C/C++ 程序时，容易遇到一个问题：在编译时，我们通过 `-L<dir> -l<lib_name>` 选项指定了共享库，告诉编译器某些符号可以在哪个共享库中找到，但是在运行时，动态链接器却告诉我们无法找到所需的共享库。下面就让我们看看 Linux 平台上动态链接器搜索共享库的规则。

## 共享库搜索顺序

在共享库和可执行文件中，`.dynamic` 段保存了动态链接时需要的信息，其中包括了所依赖的共享库列表, RPATH, RUNPATH 等。我们可以通过 `readelf -d` 查看：

```bash
$ readelf -d ./executable

Dynamic section at offset 0x5c18 contains 34 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libcustom.so.3]
 0x0000000000000001 (NEEDED)             Shared library: [libstdc++.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000001d (RUNPATH)            Library runpath: [/home/jack/project_1/lib:/home/jack/project_1/third_party/custom/lib]
 0x000000000000000c (INIT)               0x3000
 ...
```

其中，NEEDED 代表当前文件**直接**依赖的共享库。RPATH 和 RUNPATH 都表示共享库的搜索路径，多个路径用 `:` 隔开。动态链接器为了找到共享库，会按照如下顺序搜索：

1. `RPATH`：链接到可执行文件的目录列表，在大多数 UNIX 系统上都支持。如果 `RUNPATH` 存在，则被忽略。
2. `LD_LIBRARY_PATH`：保存目录列表的环境变量。
3. `RUNPATH`：与 `RPATH` 相同，但在 `LD_LIBRARY_PATH` 之后搜索。仅在最新的 UNIX 系统上支持，例如在大多数当前的 Linux 系统上。
4. `/etc/ld.so.cache`：列出其他库目录的配置文件。
5. 内置目录，基本上就是 `/lib` 和 `/usr/lib`。

RPATH 和 RUNPATH 在编译时嵌入到二进制文件当中，而 LD_LIBRARY_PATH 环境变量随时都可以设置。

`/etc/ld.so.cache` 是一个缓存文件，由 ldconfig 生成。我们可以查看 `/etc/ld.so.conf.d/` 目录下的文件，它们相当于 `/etc/ld.so.cache` 的文本形式：

```bash
$ cat /etc/ld.so.conf.d/x86_64-linux-gnu.conf 
# Multiarch support
/usr/local/lib/x86_64-linux-gnu
/lib/x86_64-linux-gnu
/usr/lib/x86_64-linux-gnu
```

像 `libstdc++.so` 这种库，都是利用 `/etc/ld.so.cache` 找到的。我们可以在这些文本文件中添加路径，然后执行 `ldconfig` 刷新 `/etc/ld.so.cache`。

我们可以通过设置 `LD_DEBUG=libs` 来**查看共享库的查找过程**：

```bash
LD_DEBUG=libs ./executable
```

## 设置 RPATH 或 RUNPATH

使用 gcc/g++ 时，我们通常使用 `-Wl,option` 将选项传递给链接器。前面提到的 RPATH 和 RUNPATH 也是一样：

- 使用 `-Wl,--disable-new-dtags -Wl,-rpath=/tmp` 添加 `/tmp` 为 RPATH。
- 使用 `-Wl,--enable-new-dtags -Wl,-rpath=/tmp` 添加 `/tmp` 为 RUNPATH。

在我的 Ubuntu 22.04 上，虽然 ld 的 man page 说了 `--disable-new-dtags` 是默认选项，但是实际上使用了 RUNPATH 作为默认选择。

除了搜索优先级，RPATH 和 RUNPATH 还有一个重要的区别：**RPATH 适用于所有的依赖，而 RUNPATH 只适用于直接依赖。** 例如，执行文件 A 依赖于 B.so，B.so 依赖于 C.so，则 A 的 RUNPATH 只在搜索 B.so 时使用，而 RPATH 在搜索 B.so 和 C.so 时都会使用。

为什么有 RPATH 和 RUNPATH 之分呢？在过去，还只有 RPATH，它的优先级又比 LD_LIBRARY_PATH 高，程序员无法通过改变 LD_LIBRARY_PATH 来覆盖掉 RPATH，导致无法在不重新编译的情况下改变某些共享库的选择。因此，推出了 RUNPATH，它的优先级比 LD_LIBRARY_PATH 低，程序员就能通过修改 LD_LIBRARY_PATH 来改变共享库的选择。

> 不过现在也有 `patchelf --set-rpath <path> <file>` 这样的工具可以修改编译好的文件的 RUNPATH。

想要在 CMake 中设置使用 RPATH，可以采用如下命令：

```cmake
target_link_options(<target> PRIVATE "-Wl,--disable-new-dtags")
```

CMake 不会直接调用链接器，而是先编译出对象文件，再使用**编译器**将对象文件链接为可执行文件。所以 `target_link_options` 不是将选项传递给链接器，而是在编译器链接阶段传递给编译器，因此需要使用 `-Wl` 选项。
