+++
title = "论bevy的渲染器设计-00"
date = 2025-10-13

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。

<!-- more -->

# 渲染器(渲染系统)

渲染器渲染可渲染的对象。

# 功能

1. 将可渲染的对象转化为图形api可识别的对象
2. 剔除哪些不必要渲染的渲染对象
