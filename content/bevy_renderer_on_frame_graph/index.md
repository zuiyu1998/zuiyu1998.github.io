+++
title = "bevy_renderer_on_frame_graph"
date = 2025-03-17

[taxonomies]
tags = ["bevy", "renderer", "frame-graph"]
+++

这是基于 frame graph 实现 renderer 的一篇 rfc。

<!-- more -->

# 基于 frame graph 的 bevy renderer 设计

## Summary

基于 frame graph 的渲染设计。

## Motivation

相比起现行的 RenderGraph 设计，添加 frame graph 会增加两个功能:

- 高效资源管理：准确地追踪每个资源在何时何地被创建、使用和销毁。自动的资源生命周期管理。
- 易于调试：基于 DAG，很方便将渲染图可视化，用于分析渲染阶段之间的依赖关系和执行顺序，以及节点状态、性能分析和统计信息等等。

## User-facing explanation

frame graph 是一个有向无环图。它由渲染节点和资源节点构成。渲染节点描述了如何将资源绘制到 gpu 上。资源节点描述了如何使用资源。
frame graph 将一帧的渲染分为三个阶段: Setup,Compile,Execute。在 Setup 阶段，用户可以自由的构建 frame graph。在 Compile 阶段，frame graph 会确认每一个资源的生命周期。在 Execute 阶段，frame graph 会将资源和 gpu 资源一一对应，并执行渲染节点描述的渲染。

### 资源

frame graph 最重要的功能是自动管理资源的创建、使用和销毁。这里的资源并不是 gpu 资源。而是 frame graph 自定义的资源。资源通常情况下不对应实际的 gpu 资源，只有处于 frame graph 的 Execute 阶段，才会和 gpu 资源一一对应，因此资源通常情况下有两个状态: 资源描述符和 资源实例。在 rust 中定义如下：

```rust
use std::{borrow::Cow, sync::Arc};

use crate::renderer::RenderDevice;

pub trait FrameGraphResourceCreator {
    fn create_texture(&self, desc: &TextureInfo) -> FrameGraphTexture;

    fn create_buffer(&self, desc: &BufferInfo) -> FrameGraphBuffer;

    fn create_resource(&self, desc: &AnyFrameGraphResourceDescriptor) -> AnyFrameGraphResource {
        match desc {
            AnyFrameGraphResourceDescriptor::Texture(info) => {
                let texture = self.create_texture(info);
                AnyFrameGraphResource::OwnedTexture(texture)
            }
            AnyFrameGraphResourceDescriptor::Buffer(info) => {
                let buffer = self.create_buffer(info);
                AnyFrameGraphResource::OwnedBuffer(buffer)
            }
        }
    }
}

impl FrameGraphResourceCreator for RenderDevice {
    fn create_texture(&self, desc: &TextureInfo) -> FrameGraphTexture {
        let resource = self.wgpu_device().create_texture(&desc.get_texture_desc());
        FrameGraphTexture {
            resource,
            desc: desc.clone(),
        }
    }

    fn create_buffer(&self, desc: &BufferInfo) -> FrameGraphBuffer {
        let resource = self.wgpu_device().create_buffer(&desc.get_buffer_desc());

        FrameGraphBuffer {
            resource,
            desc: desc.clone(),
        }
    }
}

pub enum AnyFrameGraphResource {
    OwnedBuffer(FrameGraphBuffer),
    ImportedBuffer(Arc<FrameGraphBuffer>),
    OwnedTexture(FrameGraphTexture),
    ImportedTexture(Arc<FrameGraphTexture>),
}

pub struct FrameGraphBuffer {
    pub resource: wgpu::Buffer,
    pub desc: BufferInfo,
}

#[derive(Clone, Hash, PartialEq, Eq)]
pub struct BufferInfo {
    pub label: Option<Cow<'static, str>>,
    pub size: wgpu::BufferAddress,
    pub usage: wgpu::BufferUsages,
    pub mapped_at_creation: bool,
}

impl BufferInfo {
    pub fn get_buffer_desc(&self) -> wgpu::BufferDescriptor {
        wgpu::BufferDescriptor {
            label: self.label.as_deref(),
            size: self.size,
            usage: self.usage,
            mapped_at_creation: self.mapped_at_creation,
        }
    }
}

pub struct FrameGraphTexture {
    pub resource: wgpu::Texture,
    pub desc: TextureInfo,
}

#[derive(Clone, Hash, PartialEq, Eq)]
pub struct TextureInfo {
    pub label: Option<Cow<'static, str>>,
    pub size: wgpu::Extent3d,
    pub mip_level_count: u32,
    pub sample_count: u32,
    pub dimension: wgpu::TextureDimension,
    pub format: wgpu::TextureFormat,
    pub usage: wgpu::TextureUsages,
    pub view_formats: Vec<wgpu::TextureFormat>,
}

impl TextureInfo {
    pub fn get_texture_desc(&self) -> wgpu::TextureDescriptor {
        wgpu::TextureDescriptor {
            label: self.label.as_deref(),
            size: self.size,
            mip_level_count: self.mip_level_count,
            sample_count: self.sample_count,
            dimension: self.dimension,
            format: self.format,
            usage: self.usage,
            view_formats: &self.view_formats,
        }
    }
}

pub enum AnyFrameGraphResourceDescriptor {
    Texture(TextureInfo),
    Buffer(BufferInfo),
}

```

这里定义了两种资源: buffer、texture。如果需要可以轻松的扩展 AnyFrameGraphResourceDescriptor，AnyFrameGraphResource

## Implementation strategy

This is the technical portion of the RFC.
Try to capture the broad implementation strategy,
and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following [repo guidelines](https://github.com/bevyengine/rfcs):

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing Bevy features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected.

## Drawbacks

Why should we _not_ do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What objections immediately spring to mind? How have you addressed them?
- What is the impact of not doing this?
- Why is this important to implement as a feature of Bevy itself, rather than an ecosystem crate?

## \[Optional\] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
This can include:

- Does this feature exist in other libraries and what experiences have their community had?
- Papers: Are there any published papers or great posts that discuss this?

This section is intended to encourage you as an author to think about the lessons from other tools and provide readers of your RFC with a fuller picture.

Note that while precedent set by other engines is some motivation, it does not on its own motivate an RFC.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect Bevy as a whole in a holistic way.
Try to use this section as a tool to more fully consider other possible
interactions with the engine in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
If a feature or change has no direct value on its own, expand your RFC to include the first valuable feature that would build on it.
