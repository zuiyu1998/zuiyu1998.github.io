+++
title = "bevy_render_graph"
date = 2024-01-24

[taxonomies]
tags = ["bevy", "render", "render-graph"]
+++

Render Graph 或者说 Frame graph 是对复杂渲染管线的一个高度抽象，以图（Graph）的形式呈现渲染过程中的各个步骤，不同的渲染任务之间的依赖关系，以及它们对资源（如纹理、缓冲区等）的使用。

<!-- more -->

# 设计目标

Render Graph 或者说 Frame graph 是对复杂渲染管线的一个高度抽象，以图（Graph）的形式呈现渲染过程中的各个步骤，不同的渲染任务之间的依赖关系，以及它们对资源（如纹理、缓冲区等）的使用。

Render Graph 的主要思想是将渲染过程表示为一个有向无环图（DAG），其中节点表示渲染通道（Render pass），边表示依赖关系。每个渲染通道执行特定的渲染操作，可具有输入和输出资源。

Render Graph 的目标是为了解决大型渲染引擎里复杂渲染管线中的一些问题。如资源生命周期管理、渲染效率、渲染过程的可视化调试等等。

Render Graph 的优点如下:

- 高可复用性。通过将渲染过程分解为模块化的独立节点，使得渲染代码更易于维护和扩展。同时多个渲染阶段能够共用节点、复用资源，提高渲染效率。
- 高效资源管理。确地追踪每个资源在何时何地被创建、使用和销毁。自动的资源生命周期管理。
- 并行优化。更激进的资源异步处理，分离子图(Subgraph)到多个 CommandQueue 或者将没有依赖关系的节点分别提交到不同的 CommandBuffer 并行执行，充分压榨多核。
- 高度灵活。上层可以基于这个机制自由定义和组织渲染管线，更好地适应不同的渲染需求和平台限制。
- 易于调试。基于 DAG，很方便将渲染图可视化，用于分析渲染阶段之间的依赖关系和执行顺序，以及节点状态、性能分析和统计信息等等。

# 主要设计

node trait 如下:

```rust
pub trait Node: Downcast + Send + Sync + 'static {
    /// Specifies the required input slots for this node.
    /// They will then be available during the run method inside the [`RenderGraphContext`].
    fn input(&self) -> Vec<SlotInfo> {
        Vec::new()
    }

    /// Specifies the produced output slots for this node.
    /// They can then be passed one inside [`RenderGraphContext`] during the run method.
    fn output(&self) -> Vec<SlotInfo> {
        Vec::new()
    }

    /// Updates internal node state using the current render [`World`] prior to the run method.
    fn update(&mut self, _world: &mut World) {}

    /// Runs the graph node logic, issues draw calls, updates the output slots and
    /// optionally queues up subgraphs for execution. The graph data, input and output values are
    /// passed via the [`RenderGraphContext`].
    fn run<'w>(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext<'w>,
        world: &'w World,
    ) -> Result<(), NodeRunError>;
}
```

input 输入资源，output 输出资源。update 更新内部状态，run 为 node 的主要逻辑。

RenderGraph 定义如下:

```rust
#[derive(Resource, Default)]
pub struct RenderGraph {
    nodes: HashMap<InternedRenderLabel, NodeState>,
    sub_graphs: HashMap<InternedRenderSubGraph, RenderGraph>,
}

pub struct NodeState {
    pub label: InternedRenderLabel,
    /// The name of the type that implements [`Node`].
    pub type_name: &'static str,
    pub node: Box<dyn Node>,
    pub input_slots: SlotInfos,
    pub output_slots: SlotInfos,
    pub edges: Edges,
}

```

node 实际节点。但是 NodeState。这里重要的是 NodeState 的 new 方法。

```rust
pub fn new<T>(label: InternedRenderLabel, node: T) -> Self
where
    T: Node,
{
    NodeState {
        label,
        input_slots: node.input().into(),
        output_slots: node.output().into(),
        node: Box::new(node),
        type_name: std::any::type_name::<T>(),
        edges: Edges {
            label,
            input_edges: Vec::new(),
            output_edges: Vec::new(),
        },
    }
}
```

注意 RenderGraph 的 run_graph 方法。这是整个渲染器的主要逻辑。这个函数将会一步步按照依赖关系展平节点，并且通过节点的 run 方法去渲染。这是 render graph 中的 Execute 阶段。

ViewNode trait,这是 node trait 的特化 trait，ViewNodeRunner 使用 ViewNode trait。

```rust
pub trait ViewNode {
    /// The query that will be used on the view entity.
    /// It is guaranteed to run on the view entity, so there's no need for a filter
    type ViewQuery: ReadOnlyQueryData;

    /// Updates internal node state using the current render [`World`] prior to the run method.
    fn update(&mut self, _world: &mut World) {}

    /// Runs the graph node logic, issues draw calls, updates the output slots and
    /// optionally queues up subgraphs for execution. The graph data, input and output values are
    /// passed via the [`RenderGraphContext`].
    fn run<'w>(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext<'w>,
        view_query: QueryItem<'w, Self::ViewQuery>,
        world: &'w World,
    ) -> Result<(), NodeRunError>;
}
```

ViewNodeRunner 是用于 view 渲染的叶子节点，没有 input，也没有 out.

# 参考资料

- ![Learning Render Graph](https://yrom.net/blog/2023/08/13/learning-render-graph/)
