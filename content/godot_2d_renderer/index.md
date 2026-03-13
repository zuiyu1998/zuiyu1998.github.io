+++
title = "godot 2d renderer"
date = 2026-03-13

[taxonomies]
tags = ["godot", "renderer", "2d", "游戏引擎"]
+++

这是一个关于godot 2d 渲染实现的专栏。

<!-- more -->

# tick(update)

在游戏引擎中的tick中负责渲染的是RenderServer对象。RenderServer负责实现渲染的是 \_draw 函数。

RendererCanvasCull负责剔除和排序需要显示的2d模块。

RendererCanvasRender负责跟cpu沟通，转化为绘制命令。
