+++
title = "反射"
date = 2024-01-24

[taxonomies]
tags = ["reflect", "bevy", "fyrox"]
+++

探究 rust 反射机制的实现。

<!-- more -->

# 反射是什么

能够改变和观察自己的动态类型。

# rust 中反射

rust 中使用 Any Trait 来模拟动态类型，因此在 rust 中动态类型就是 dyn Any。dyn Any 的功能并不强大，社区有更强大的实现。这里讨论的是 fyrox 和 bevy 的反射库。
反射的基本功能:

- 具体类型到动态类型
- 动态类型的修改
- 序列化和反序列化
- 动态类型到具体类型
- 动态类型的类型信息

社区库的实现反射的 trai 为 Reflect Trait。这个库都实现了对 Any trait 的兼容。dyn Reflect 可以向上转换为 dyn Any。

# bevy 和 fyrox 的相同点

- 向上转换
- 向下转换
- 类型信息
- 动态类型的修改

# 不同点

- 动态类型的转换
  - bevy 通过 TypeRegistry 实现
  - fyrox 在 Reflect 指定
