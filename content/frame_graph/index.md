+++
title = "frame graph"
date = 2024-01-24

[taxonomies]
tags = ["frame graph", "game engine"]
+++

frame graph 是一种新的渲染器实现方式。

<!-- more -->

# frame graph

frame graph 是一个有向无环图。它分为两类节点，一类是 pass 节点。一类是 resource 节点。

# pass 节点

- 描述涉及到的资源节点
  - read
  - write
- 对资源进行操作

# resource 节点

- 对资源的描述
- 多帧资源

frame graph 每一帧都会生成，setup 阶段描述渲染的所有信息。compile 阶段进行 pass 排序，资源生成。execute 阶段 执行所有 pass。
