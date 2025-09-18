+++
title = "从零开始draft渲染器设计-00"
date = 2025-09-10

[taxonomies]
tags = ["draft", "renderer"]
+++

这是设计 draft renderer 渲染器的一系列文章。

<!-- more -->

# 总述

draft renderer 是一个基于 webgpu 的渲染器。

# webgpu 的三角形渲染流程

1. 申请 gpu 的资源，并且通过申请到的资源从 cpu 端向 gpu 填充数据
2. 申请 gpu 的绑定组布局和管线
3. 根据绑定组布局和资源生成绑定组
4. 获取 encoder 等资源，生成 commmand buffer

# 常见概念

- 材质
  用于管线的生成
  用于填充对应的资源

# 流程

1. 获取可以向 gpu 渲染的所有 cpu 对象
2. 申请 cpu 对象对应的全局 gpu 资源
3. 获取 cpu 对象的局部 gpu 资源

# 问题

draft 渲染器是如何处理上述步骤的？

如何处理同一类材质，如何处理同一材质？以及扩展。

如何判定局部的 uniform 生成？
在渲染同一类型的时候，相同名称的资源应该指向同一个 uniform，避免申请多个。
