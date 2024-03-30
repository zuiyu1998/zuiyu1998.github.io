+++
title = "linux内核模块构建"
date = 2024-03-29

[taxonomies]
tags = ["rust", "linux"]
+++

关于 linux 模块构建的只是。

<!-- more -->

# 内核模块

```Kconfig
config mod_versions
    bool string
    depends on mods
    help
        string
```

一个内核模块的 Kconfig 声明需要三个部分，模块名，编译选项，依赖列表，帮助信息。
Kbuild Makefile 需要根据编译选项实际生成 config

# Kbuild Makefile

一个简易的 Kbuild Makefile 声明如下:

```makefile
obj-y += foo.o
```

y 为 Kconfig 声明的编译选项，通常它是一个变量，在编译的时候指定。foo.o 为生成的文件。编译的文件同名。

> y 为内核,

除了可以声明文件，还可以声明对应的子目录,代码如下。

```makefile
obj-y += foo\
```

或者另一个形式的声明。

```makefile
subdir-y += foo
```
