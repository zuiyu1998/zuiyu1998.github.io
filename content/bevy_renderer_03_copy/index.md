+++
title = "从零开始bevy渲染器设计-窗口与摄像机"
date = 2025-09-10

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。

<!-- more -->

# 窗口

在各平台上的窗口实例中获取对应的 surface，提取对应的 TextureView。摄像机获取指向的 surface 资源。
