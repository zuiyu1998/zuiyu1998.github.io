+++
title = "godot 2d renderer"
date = 2026-03-16

[taxonomies]
tags = ["godot", "renderer", "2d", "游戏引擎"]
+++

这是一个关于godot 2d 渲染实现的专栏。

<!-- more -->

# tick(update)

在游戏引擎中的tick中负责渲染的是RenderServer对象。RenderServer负责实现渲染的是 \_draw 函数。

\_draw_viewport

# RendererCanvasCull

RendererCanvasCull负责剔除和排序需要显示的2d模块。

RendererCanvasCull中创建了每一个CanvasItem对应的渲染数据。

RendererCanvasRender负责跟gpu沟通，转化为绘制命令。
