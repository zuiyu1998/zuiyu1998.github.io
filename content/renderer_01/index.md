+++
title = "从零开始渲染器-Pass"
date = 2025-07-31

[taxonomies]
tags = ["renderer", "rust", "wgpu"]
+++

从零开始渲染器.论 Pass 的设计。

<!-- more -->

# 需求

在一般的流程中，需要手动指定要使用的 pipeline，buffer，texture，bindgroup 构建 RenderPass 或者 ComputerPass。手动容易出错，且对人的要求较高。因此实现一个对象自动实现对 wgpu 的操作。

# 实现

使用命令模式抽象对 wgpu 的操作。而 Pass 就是该命令的集合。

```rust
pub trait PassCommand: 'static + Send + Sync {
    fn execute(&self, context: PassContext);
}

pub struct PassContext<'a> {
    pub enconder: &'a mut CommandEncoder,
    pub resource_register_context: &'a mut ResourceRegistryContext,
}

pub struct ResourceRegistryContext {
    pub resources: ResourceRegisterContainer,
}

pub struct ResourceRegisterContainer {}

pub struct Pass(Vec<Box<dyn PassCommand>>);

```

PassCommand trait 负责实际的渲染过程。Pass 保存渲染过程中要执行的指令。PassContext 为 PassCommand 执行过程中的上下文。其中 ResourceRegisterContainer 为当前上下文中使用的实际资源列表。这个资源列表由 frame graph 自动填充。

# 问题

RenderPass 如何设计？ComputerPass 如何设计。
