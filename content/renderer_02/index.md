+++
title = "从零开始渲染器-TransientResource"
date = 2025-07-31

[taxonomies]
tags = ["renderer", "rust", "wgpu"]
+++

从零开始渲染器.论 TransientResource。

<!-- more -->

# TransientResource
TransientResource分为两个阶段，资源标记和已注册资源。资源标记用于 frame graph 的 Setup 阶段和 Compile 阶段。注册资源用于 frame graph 的实际渲染。

```rust
#[derive(Clone)]
pub enum AnyTransientResource {}

#[derive(Clone)]
pub enum AnyTransientResourceDescriptor {}

#[derive(Clone)]
pub enum AnyArcTransientResource {}

#[derive(Clone)]
pub enum VirtualResource {
    Setuped(AnyTransientResourceDescriptor),
    Imported(AnyArcTransientResource),
}
```
AnyTransientResource代表已经注册的资源，这个和实际的gpu资源一一对应。VirtualResource代表资源标记。

定义TransientResource trait和TransientResourceDescriptor  trait。代码如下:
```rust
pub trait TransientResource: 'static {
    type Descriptor: TransientResourceDescriptor;

    fn borrow_resource(res: &AnyTransientResource) -> &Self;

    fn get_desc(&self) -> &Self::Descriptor;
}

pub trait TransientResourceDescriptor:
    'static + Clone + Into<AnyTransientResourceDescriptor>
{
    type Resource: TransientResource;

    fn borrow_resource_descriptor(res: &AnyTransientResourceDescriptor) -> &Self;
}
```

# ResourceRegisterContainer
ResourceRegisterContainer的主要功能有三点：
- 创建实际的资源
- 销毁实际的资源
- 获取实际的资源
