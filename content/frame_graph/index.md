+++
title = "基于frame-graph的渲染器设计思路"
date = 2025-03-17

[taxonomies]
tags = ["frame_graph", "renderer",]
+++

这是探究渲染器实现的一系列文章。本篇文章探究 frame-graph 的设计。

<!-- more -->

# frame-graph

frame-graph 是一个典型的图结构。它分为两大类，一个是渲染节点，一个是资源节点。
渲染节点使用资源节点指向的资源构建 RenderPass。
RenderPass 是一个薄的抽象层，以支持更多的渲染器后端。代码如下:

```rust
pub trait RenderPassTrait: 'static + Debug + Sync + Send + Any {
    fn draw(&mut self, vertices: Range<u32>, instances: Range<u32>);

    fn set_render_pipeline(&mut self, render_pipeline: &RenderPipeline);
}

pub struct RenderPass(Box<dyn RenderPassTrait>);
```

渲染节点指向的资源存储在 RenderContext 中。资源节点指向为ResourceNodeHandle。
