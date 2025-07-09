+++
title = "基于frame-graph的渲染器设计思路"
date = 2025-07-07

[taxonomies]
tags = ["frame_graph", "renderer"]
+++

这是探究使用 frame graph 实现渲染器的一系列文章。本篇文章探究了 frame-graph 的设计和一系列问题。

<!-- more -->

# frame-graph

frame graph 是一个有向无环图。它由两类节点构成，渲染节点和资源节点。
渲染节点负责使用实际的 gpu 资源进行渲染。
资源节点是实际 gpu 资源的占位。

frame graph 对一帧的渲染分为三个阶段：

- Setup， 该阶段构建 frame graph 实例。
- Compile, 该阶段计算渲染节点使用资源的生命周期
- Execute，该阶段获取 gpu 资源，并进行渲染

# Transient Resources

由 Frame Graph 管理的资源。一个 Transient Resource 描述了对应 gpu 资源。通常情况下是 Buffer 和 Texture

# PassNode

渲染节点保存了将要使用资源的索引，这些索引指向了实际的 gpu 资源。

- pipeline 和 layout
- 数据和 bind group
