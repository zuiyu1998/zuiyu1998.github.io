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

渲染器本质上是一个将世界中的物体渲染到屏幕上的工具。

# 如何将场景的物体渲染到屏幕上

通常情况下，这个功能分为两个步骤。

- 通过摄像机观察世界，获取世界的截图。
- 将所有的截图合成到窗口。
