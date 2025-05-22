+++
title = "基于frame-graph的渲染器设计思路"
date = 2025-05-19

[taxonomies]
tags = ["frame_graph", "renderer"]
+++

这是探究使用 frame graph 实现渲染器的一系列文章。本篇文章探究 frame-graph 的设计和一系列问题。

<!-- more -->

# frame-graph

frame graph 是一个有向无环图。它由两类节点构成，渲染节点和资源节点。
渲染节点负责使用实际的 gpu 资源进行渲染。
资源节点是实际 gpu 资源的占位。

frame graph 对渲染分为三个阶段：

- Setup， 该阶段构建 frame graph。
- Compile, 该阶段计算渲染节点使用资源的生命周期
- Execute，该阶段实现进行渲染

# Transient Resources

由 Frame Graph 管理的资源。一个 Transient Resource 描述了对应 gpu 资源 的使用和创造。通常情况下是 Buffer 和 Texture

# PassNode

渲染节点最重要的功能是对 gpu 的资源使用。在一般的实现中，这里都是借助闭包捕获对应的索引。但是更好的设计是手动收集这些索引。
