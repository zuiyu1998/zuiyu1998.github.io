+++
title = "基于frame-graph的bevy渲染器设计思路"
date = 2025-03-17

[taxonomies]
tags = ["frame_graph", "renderer", "bevy"]
+++

这是探究渲染器实现的一系列文章。本篇文章探究 frame-graph 的设计。

<!-- more -->

# frame-graph

frame-graph 中最重要的三个概念。

- 渲染节点
- 资源节点
- 资源

frame-graph 的三个阶段

- Setup
- Compile
- Execute

# 阶段

将 frame-graph 的三个阶段对应 RenderApp 中的 SystemSet。

```rust
#[derive(Debug, Hash, PartialEq, Eq, Clone, SystemSet)]
pub enum FrameGraphSet {
    Setup,
    Compile,
    //Render
    Execute,
}
```

# 资源

frame-graph 管理的 gpu 资源。它根据 frame-graph 的阶段有多个状态：

- 外部导入，这种的资源生命周期不受 framegraph 管理，因此只能读取。
- Setup，资源未被创建。
- Compile，资源已被创建

资源节点的功能如下:

- 唯一标识
- 渲染节点记录
- 版本管理(同一个资源节点可以被多个渲染节点操作)

并不是所有的 gpu 资源都由 frame-graph 管理。定义一个 trait 声明可以被 frame-graph 管理的资源，代码如下:

```rust

#[derive(Debug)]
pub enum AnyFGResource {}

#[derive(Debug, Clone, Hash, PartialEq, Eq)]
pub enum AnyFGResourceDescriptor {}


pub trait FGResource: 'static + Debug {
    type Descriptor: FGResourceDescriptor;

    fn borrow_resource(res: &AnyFGResource) -> &Self;
}

pub trait FGResourceDescriptor: 'static + Clone + Debug + Into<AnyFGResourceDescriptor> {
    type Resource: FGResource;
}

pub struct Resource {
    state: ResourceState,
    info: ResourceInfo,
}

pub struct ResourceInfo {
    ///唯一的资源名称
    pub name: String,
    ///资源索引
    pub handle: TypeHandle<Resource>,
    /// 资源版本
    version: u32,
    ///首次使用此资源的渲染节点
    pub first_pass_node_handle: Option<TypeHandle<PassNode>>,
    ///最后使用此资源的渲染节点
    pub last_pass_node_handle: Option<TypeHandle<PassNode>>,
}


pub struct ImportedResourceState {}

pub enum ResourceState {
    Imported(ImportedResourceState),
    Setup(AnyFGResourceDescriptor),
    Compile(AnyFGResource),
}
```

# 资源节点

资源节点是 frame-graph 中管理资源的节点，资源节点功能如下:

- 指向资源
- 保留对资源的操作性
- 记录最后操作的渲染节点

定义如下:

```rust
pub struct ResourceNode {
    handle: TypeHandle<ResourceNode>,
    resource_handle: TypeHandle<Resource>,
    version: u32,
    writer_handle: TypeHandle<PassNode>,
}
```

# 渲染节点

渲染节点是 frame-graph 最重要的节点，它承载着整个渲染器的主体功能。渲染节点的功能如下:

- 使用涉及到的资源进行渲染。
- 渲染节点排序
- 记录资源的生命周期
- 读取资源节点
- 写入资源节点

定义一个对象 DynRenderFn，它的主要功能是使用资源进行渲染，代码如下:

```rust
pub type DynRenderFn = dyn FnOnce(&mut RenderPass, &mut RenderContext) -> Result<(), RendererError>;

```

RenderPass 是实际 RenderPass 的包装层，RenderContext 负责从各种索引处获取实际的资源实例。它的工作流程是:通过闭包获取索引和写入的数据，根据索引获取资源实例并写入 RenderPass。

读取和写入资源节点都会返回资源节点的索引，这里与普通索引不同的是需要保留写入和读取的属性。定义如下:

```rust
pub struct ResourceNodeRef<ResourceType, ViewType> {
    handle: TypeHandle<ResourceNode>,
    resource_handle: TypeHandle<Resource>,
    _marker: PhantomData<(ResourceType, ViewType)>,
}
```

为渲染节点添加两个方法,代码如下:

```rust
impl PassNode {
    pub fn write<ResourceType>(
            &mut self,
            fg: &mut FrameGraph,
            resource_handle: ResourceNodeHandle<ResourceType>,
        ) -> ResourceNodeRef<ResourceType, GpuWrite>;

    pub fn read<ResourceType>(
        &mut self,
        fg: &mut FrameGraph,
        resource_handle: ResourceNodeHandle<ResourceType>,
    ) -> ResourceNodeRef<ResourceType, GpuRead> {
        todo!()
    }
}
```

# 索引

索引的种类很多，它和指向的资源一一对应。常见的有资源索引，渲染管道索引
