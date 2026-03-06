+++
title = "从零开始bevy渲染器设计-01-获取世界数据"
date = 2026-03-06

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。这篇文章讨论如何从主世界获取数据到渲染世界。

<!-- more -->
# ExtractPlugin

ExtractPlugin是一个用于从bevy主世界获取数据到渲染数据的plugin.
在RenderSystems中ExtractCommands中执行ExtractSchedule。
- extract_component (提取组件)
- extract_resource (提取资源)
- extract_param
- 其他