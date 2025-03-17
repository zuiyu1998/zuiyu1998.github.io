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
渲染节点、资源节点、资源定义如下:

```rust
pub struct PassNode {
}

pub struct ResourceNode {
}

pub struct Resource {
}

pub trait FGResource: 'static + Debug;

```

渲染节点使用资源节点指向的资源构建 RenderPass。
RenderPass 是一个薄的抽象层，通过定义 RenderPassTrait 可以以支持更多的渲染器后端。代码如下:

```rust
pub trait RenderPassTrait: 'static + Debug + Sync + Send + Any {
    fn draw(&mut self, vertices: Range<u32>, instances: Range<u32>);

    fn set_render_pipeline(&mut self, render_pipeline: &RenderPipeline);
}

pub struct RenderPass(Box<dyn RenderPassTrait>);

```

渲染节点使用的资源需要存储在一个数据结构中，定义一个结构体存储需要的资源。它主要提供的功能如下:

- 通过资源节点的索引获取资源的引用，读索引获取&引用，写索引获取&mut 引用。

定义如下:

```rust

pub type DynRenderFn = dyn FnOnce(&mut RenderPass, &mut RenderContext) -> Result<(), RendererError>;


///各类资源的上下文，用于存储资源
pub struct RenderContext {
}

impl RenderContext {
    pub fn get_resource_mut<ResourceType: FGResource>(
        &mut self,
        handle: &ResourceRef<ResourceType, GpuWrite>,
    ) -> Option<&mut ResourceType>;

   pub fn get_resource_mut<ResourceType: FGResource>(
        &self,
        handle: &ResourceRef<ResourceType, GpuRead>,
    ) -> Option<&ResourceType>;
}

///资源节点的索引, ViewType决定获取对应的引用
pub struct ResourceRef<ResourceType, ViewType> {
    handle: ResourceNodeHandle<ResourceType>,
    _marker: PhantomData<ViewType>,
}

pub trait GpuViewType {
    const IS_WRITABLE: bool;
}

pub struct GpuRead;

impl GpuViewType for GpuRead {
    const IS_WRITABLE: bool = false;
}

pub struct GpuWrite;

impl GpuViewType for GpuWrite {
    const IS_WRITABLE: bool = true;
}
```
