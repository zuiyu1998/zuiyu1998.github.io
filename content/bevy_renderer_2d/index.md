+++
title = "从零开始bevy 2d渲染"
date = 2026-03-09

[taxonomies]
tags = ["bevy", "renderer", "2d"]
+++

这是探究 bevy 2d 渲染的一系列文章。从bevy 2d_shapes的例子开始。

<!-- more -->

# 2d 网格渲染

主世界的2d网格实体如下：

- Mesh2d
- MeshMaterial2d

其中MeshMaterial2d中的材质为ColorMaterial。

# 2d渲染

1. 从主世界提取ColorMaterial实体
2. 将ColorMaterial转换为ColorMaterialUniform
