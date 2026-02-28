+++
title = "从零开始bevy渲染器设计-01-图形api"
date = 2026-02-28

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。这篇文章谈论如何构建图形api。

<!-- more -->

# 图形api

图形api通常情况下指的是opengl、metal、direct12这种对硬件抽象的sdk。
游戏引擎的使用者不需要关注这些图形api，因此需要有一个中间层对图形api进行抽象。
