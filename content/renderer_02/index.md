+++
title = "从零开始渲染器-TransientResource"
date = 2025-07-31

[taxonomies]
tags = ["renderer", "rust", "wgpu"]
+++

从零开始渲染器.论 TransientResource。

<!-- more -->

# TransientResource

在上一篇中谈到 ResourceRegisterContainer 是渲染命令上下文中资源列表，并且 ResourceRegisterContainer 中资源是自动填充的。这一篇就是谈论 ResourceRegisterContainer 如何被填充。

ResourceRegisterContainer 的资源是 frame graph 资源概念的最终阶段。这个阶段的资源和实际的 gpu 资源一一对应，通常情况下它是 buffer 或者 texture。
那么 frame graph 中资源一共有几个阶段?。资源分为两个阶段，资源标记和已注册资源。资源标记用于 frame graph 的 Setup 阶段和 Compile 阶段。注册资源用于 frame graph 的实际渲染。

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

