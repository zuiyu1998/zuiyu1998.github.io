+++
title = "基于frame-graph的渲染器设计思路"
date = 2025-03-17

[taxonomies]
tags = ["frame_graph", "renderer", "bevy"]
+++

这是探究使用 frame graph 实现渲染器的一系列文章。本篇文章探究 frame-graph 的设计和一系列问题。

<!-- more -->

# frame-graph

frame graph 是一个典型的图数据结构。它由两类节点构成，渲染节点和资源节点。构建 frame graph 就是构建渲染节点和资源节点的关系图。

在每帧渲染的时候一共有三个阶段。

- Setup
  负责构建渲染节点和和资源节点的关系图
- Compile
  优化各个渲染节点的关系，并且填充渲染节点的资源依赖
- Execute
  申请 gpu 资源并且执行渲染

面临的问题：

- 填入资源的数据在什么时候获取？比如顶点数据和索引数据，图片的字节数据。
  这些 gpu 资源的创建和销毁都大于 frame graph 的生命周期。

  也就是在 Setup 阶段，除了声明用到的资源，还要从外部导入已创建完成的资源。

- Execute 如何运作？
  在渲染节点运行的时候，所有的资源都会替换为索引，然后有一个资源表可以根据索引查找到实际的资源
