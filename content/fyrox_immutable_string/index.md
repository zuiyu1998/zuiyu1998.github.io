+++
title = "论Fyrox中ImmutableString的实现"
date = 2025-08-05

[taxonomies]
tags = ["Fyrox", "ImmutableString"]
+++

论 Fyrox 中 ImmutableString 的实现

<!-- more -->

# ImmutableString

这是 String 类型的替代。它表示的是一种不可变的 String 类型。并且只有实际字符串的内容相同，就指向同一个对象，在大量需要字符串复制，且不需要改变字符串的内容时，就可以使用 ImmutableString 替代。

ImmutableString的实际内容存储在一个全局的map中，其本身只是一个智能指针的包装。
