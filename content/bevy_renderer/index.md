+++
title = "论bevy的渲染器设计-00"
date = 2025-10-13

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。

<!-- more -->

# 渲染器(渲染系统)

渲染器负责将渲染世界中可渲染的对象渲染。

# 主体流程

1. 从主世界提取数据到渲染世界。
2. 从摄像头判断可见性
3. 将cpu数据传递gpu
4. gpu负责执行渲染
