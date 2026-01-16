+++
title = "论frame graph"
date = 2026-01-15

[taxonomies]
tags = ["frame_graph", "renderer"]
+++

论 frame graph。

<!-- more -->

# frame graph

Frame Graph（帧图）是现代游戏引擎中用于管理渲染管线的核心架构，通过有向无环图（DAG）抽象渲染流程，实现资源自动管理、跨平台优化与高效渲染调度。

frame graph 共有两个主要优点:

- 声明式渲染
- 复用 gpu 资源

# 主要流程

1. 渲染管线构建帧图。
2. 帧图编译
3. 帧图执行
