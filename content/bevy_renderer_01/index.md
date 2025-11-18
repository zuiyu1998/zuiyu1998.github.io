+++
title = "从零开始bevy渲染器设计-01-渲染流程"
date = 2025-11-17

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。这篇文章解析渲染流程。

<!-- more -->

# 渲染流程

每个摄像机摄像。
将摄像机写入窗口中。

render_system 位于 bevy_render/src/renderer.rs 文件中。整体逻辑分为三部分。

- RenderGraph 的更新
- RenderGraphRunner 的 run 函数
- 窗口更新
