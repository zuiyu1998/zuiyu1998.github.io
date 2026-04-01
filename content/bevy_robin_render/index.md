+++
title = "bevy_robin_render"
date = 2026-03-17

[taxonomies]
tags = ["renderer", "rust", "wgpu", "bevy"]
+++

bevy_robin_render是一个社区实现的基于frame-graph的渲染器。
这是实现渲染器的一系列文章，

<!-- more -->

# Wgpu

bevy_robin_render以wgpu作为基础的图形api。

# RenderPass

一个wgpu的RenderPass构建如下:

1. RenderPassDescriptor
2. RenderPipeline 实例
3. Vertex
4. Indexs
5. BindGroup

RenderPassDescriptor需要TextureView,Vertex和Indexs是Buffer。BindGroup指向Buffer和Texture。

FrameGraph在Execute阶段需要构建如上的元素。RenderPassDescriptor，Vertex,Indexs，BindGroup都和资源关联。
