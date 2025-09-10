+++
title = "从零开始bevy渲染器设计-01"
date = 2025-09-10

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。

<!-- more -->

# 总述

bevy renderer 是一个基于 webgpu、适配 ecs 的渲染器设计。

# webgpu 的三角形渲染流程

1. 申请 gpu 的资源，从 cpu 端向 cpu 填充数据
2. 申请 gpu 的绑定组布局和管线
3. 根据绑定组布局和资源生成绑定组
4. 获取 encoder，生成 commmand buffer

# 问题

bevy 渲染器是如何处理上述步骤的？
