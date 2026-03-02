+++
title = "从零开始bevy渲染器设计-02-schedule"
date = 2026-03-01

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。这篇文章谈论渲染世界的调度

<!-- more -->

# Render

渲染世界的顶层调度为Render。该调度如下:

- ExtractCommands 从主世界获取数据
- 其他
