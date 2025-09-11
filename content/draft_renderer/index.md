+++
title = "从零开始draft渲染器设计-00"
date = 2025-09-10

[taxonomies]
tags = ["draft", "renderer"]
+++

这是设计 draft renderer 渲染器的一系列文章。

<!-- more -->

# 总述

bevy renderer 是一个基于 webgpu 的渲染器。

# webgpu 的三角形渲染流程

1. 申请 gpu 的资源，从 cpu 端向 cpu 填充数据
2. 申请 gpu 的绑定组布局和管线
3. 根据绑定组布局和资源生成绑定组
4. 获取 encoder，生成 commmand buffer

# 问题

draft 渲染器是如何处理上述步骤的？

如何判定局部的 uniform 生成？
在渲染同一类型的时候，相同名称的资源应该指向同一个 uniform，避免申请多个。

# 常见概念

- 材质，描述了一个几何体的涉及的光学信息
- 几何体，描述了一个物体的性状
-
