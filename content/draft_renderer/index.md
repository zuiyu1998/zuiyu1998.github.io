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

# 渲染器

渲染器本质上是一个将世界中的物体渲染到屏幕上的工具。每个渲染器都要回到的两个问题？一个是如何渲染？一个是支持渲染的物体有哪些。

# 如何渲染

这是图形学的领域，这里暂且不去深入，而是使用一句简单的话来表示：使用图形 api 来进行渲染。常见的图形 api 有 opengl,vulkan,directx,metal。
draft 使用 wgpu 进行渲染。
