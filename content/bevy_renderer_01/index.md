+++
title = "从零开始bevy渲染器设计-01-渲染流程"
date = 2025-11-17

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。这篇文章解析渲染系统的顶层入口。

<!-- more -->

# 渲染流程

render_system 位于 bevy_render/src/renderer.rs 文件中。整体逻辑分为三部分。

- RenderGraph 的更新
- RenderGraphRunner 的 run 函数
- 窗口更新

整体来看。这就是一个 renderer 的整体逻辑。但是这里缺少了从系统窗口到窗口数据这一步。换句话说就是 ExtractedWindows 的创建过程。
