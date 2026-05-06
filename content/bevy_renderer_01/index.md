+++
title = "从零开始bevy渲染器设计-01-数据传输"
date = 2026-05-06

[taxonomies]
tags = ["bevy", "renderer"]
+++

从零开始渲染器-01.
分析bevy引擎中主世界到渲染世界的数据传输。

<!-- more -->

# 多世界设计

bevy的渲染流程位于独立主世界的渲染世界上。渲染世界和主世界是两个不同的ecs实例，因此渲染世界在实际执行渲染前需要从主世界获取对应的数据。
渲染世界获取主世界的数据的所有行为被规定在ExtractSchedule中。

# SyncWorldPlugin

SyncWorldPlugin通过给实体添加SyncToRenderWorld组件来自动绑定RenderEntity.RenderEntity为渲染世界的实体。
SyncComponentPlugin自动给组件添加SyncToRenderWorld组件。

# ExtractResourcePlugin

ExtractResourcePlugin用来同步主世界的资源到渲染世界。

# ExtractComponentPlugin

ExtractComponentPlugin用来同步主世界的组件

# ExtractInstancesPlugin

ExtractInstancesPluginy用来提取主世界的数据。
