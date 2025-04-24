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

通过对现行的 RenderGraph 进行拆分实现基于 frame graph 的渲染设计。

## Motivation

相比起现行的 RenderGraph 设计，添加 frame graph 会增加两个功能:

- 高效资源管理：准确地追踪每个资源在何时何地被创建、使用和销毁。自动的资源生命周期管理。
- 易于调试：基于 DAG，很方便将渲染图可视化，用于分析渲染阶段之间的依赖关系和执行顺序，以及节点状态、性能分析和统计信息等等。

## User-facing explanation

RenderGraph 通过 Node trait 实现渲染功能。在新的设计中将 Node trait 的功能分为 Setup trait and Pass trait。Setup trait 用于构建 FrameGraph, PassData trait 用于保存 Setup 中的各种的资源索引， 同时用这些索引实现渲染功能。

```rust

```

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
