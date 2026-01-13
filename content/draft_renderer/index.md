+++
title = "从零开始draft渲染器设计-00"
date = 2025-09-10

[taxonomies]
tags = ["draft", "renderer"]
+++

这是设计 draft renderer 渲染器的一系列文章。

<!-- more -->

# 渲染器

渲染器是一个将世界中的物体渲染到屏幕上的工具。

- 物体有哪些
- 怎么获取世界的物体
- 怎么渲染物体

# 物体有哪些

- 网格
- 材质

# 怎么获取世界的物体

世界是多类的，渲染器无法获取知晓世界。因此应该让世界告诉给渲染器有哪些物体。

# 怎么渲染物体

渲染器当然可以通过软渲染实现 cpu 渲染物体，但是这种方式并不实用，只适合学习。
渲染器正确渲染物体的方式是借助 gpu 这种额外的硬件。各种图形 api 实现了对 gpu 硬件的抽象。通常情况下只需要将物体转换为图形 api 能识别的对象就可以实现渲染了。
常见的图形 api 有 Vulkan，Metal,DirectX,OpenGl。当前渲染器使用的图形 api 是 Wegbgpu。
一般渲染器是怎么渲染物体？

1. 将物体转化为图形 api 所能识别的 gpu 对象。
2. 通过图形 api 将 gpu 对象传递给 gpu。
3. gpu 实现物体的渲染。

# 怎么管理 gpu 对象

1. 使用 frame graph 有向无环图来管理 gpu 对象的声明周期。
2. 对于 frame graph 之外的 gpu 对象通过 gc 来管理。
   - 使用 Buffer Allocator 来管理 gpu buffer

# 将物体转化为图形 api 所能识别的 gpu 对象

1. 将 mesh 转化为图形 api 所能识别的 gpu 对象
   - meshcache
