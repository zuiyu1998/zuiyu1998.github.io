+++
title = "论bevy render的设计"
date = 2024-01-24

[taxonomies]
tags = ["bevy", "rust"]
+++

bevy 是一个 rust 的开源游戏引擎，在社区备受关注。本篇文章立足 bevy render 库探究 render 渲染器的设计。

<!-- more -->

# 一句话总结 bevy render 库的功能

渲染图工作器推动渲染图中的每个节点使用自己的绘制函数将 cpu 数据转化 gpu 数据，然后将 gpu 数据传输给 gpu。
这句话有两个重要的行为：

- 每个节点使用自己的绘制函数将 cpu 数据转化 gpu 数据。
- 把 gpu 数据传输给 gpu。

理解第一个行为可以自定义渲染内容。
理解第二个行为可以自定义渲染后端。

在大多数情况下我们只关心如何自定义渲染内容，所以我们会更多的介绍第一个行为。

# 如何定义一个渲染图中的节点

在 bevy 的源代码库中我们可以全局搜索 impl Node 这几个关键字，我们能够比较精确的看到已经实现的节点。从搜索的关键字我们就可以得到，node 是一个 trait。当然通过 trait 实现抽象也在意料之中。trait 的定义如下:

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
    fn run(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext,
        world: &World,
    ) -> Result<(), NodeRunError>;
}
```

node 最为重要的函数为 run 函数，我们可以看到，它使用了二个可变的对象，RenderGraphContext 和 RenderContext。从名称也可以得知这个两个对象的作用，一个是渲染图上下文，一个是渲染上文。从这个函数定义可以看到，只要我们实现了这个 trait，我们其实就可以自定义渲染内容，但是这个实现还太抽象，我们需要明白 gpu 需要一系列的数据，才能更好的操作。直接使用这个还是一头雾水。所以需要从源码看一个实现，才能更好的理解。

# bevy ui UiPassNode

UiPassNode 的定义如下:

```rust
pub struct UiPassNode {
    ui_view_query: QueryState<
        (
            &'static RenderPhase<TransparentUi>,
            &'static ViewTarget,
            &'static ExtractedCamera,
        ),
        With<ExtractedView>,
    >,
    default_camera_view_query: QueryState<&'static DefaultCameraView>,
}
```

UiPass 的 node 实现如下:

```rust
impl Node for UiPassNode {
    fn update(&mut self, world: &mut World) {
        self.ui_view_query.update_archetypes(world);
        self.default_camera_view_query.update_archetypes(world);
    }

    fn run(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext,
        world: &World,
    ) -> Result<(), NodeRunError> {
        let input_view_entity = graph.view_entity();

        let Ok((transparent_phase, target, camera)) =
            self.ui_view_query.get_manual(world, input_view_entity)
        else {
            return Ok(());
        };
        if transparent_phase.items.is_empty() {
            return Ok(());
        }

        // use the "default" view entity if it is defined
        let view_entity = if let Ok(default_view) = self
            .default_camera_view_query
            .get_manual(world, input_view_entity)
        {
            default_view.0
        } else {
            input_view_entity
        };
        let mut render_pass = render_context.begin_tracked_render_pass(RenderPassDescriptor {
            label: Some("ui_pass"),
            color_attachments: &[Some(target.get_unsampled_color_attachment())],
            depth_stencil_attachment: None,
            timestamp_writes: None,
            occlusion_query_set: None,
        });
        if let Some(viewport) = camera.viewport.as_ref() {
            render_pass.set_camera_viewport(viewport);
        }
        transparent_phase.render(&mut render_pass, world, view_entity);

        Ok(())
    }
}
```

这里面最重要的逻辑就是下面这句：

```rust
transparent_phase.render(&mut render_pass, world, view_entity);
```

transparent_phase 是一个泛型，由 PhaseItem trait 约束，在这里它实际的类型是 RenderPhase<TransparentUi>。render_pass 是 TrackedRenderPass 的实例，它用于设置管道，顶点等 gpu 需要的数据。
一句话总结，node 的具体实现依赖于实现了 PhaseItem trait 的泛型 RenderPhase<TransparentUi>。这里我们需要探究 RenderPhase 和 PhaseItem trait 的作用。
transparent_phase.render 的实现如下:

```rust
pub fn render<'w>(
    &self,
    render_pass: &mut TrackedRenderPass<'w>,
    world: &'w World,
    view: Entity,
) {
    self.render_range(render_pass, world, view, ..);
}

pub fn render_range<'w>(
    &self,
    render_pass: &mut TrackedRenderPass<'w>,
    world: &'w World,
    view: Entity,
    range: impl SliceIndex<[I], Output = [I]>,
) {
    let items = self
        .items
        .get(range)
        .expect("`Range` provided to `render_range()` is out of bounds");

    let draw_functions = world.resource::<DrawFunctions<I>>();
    let mut draw_functions = draw_functions.write();
    draw_functions.prepare(world);

    let mut index = 0;
    while index < items.len() {
        let item = &items[index];
        let batch_range = item.batch_range();
        if batch_range.is_empty() {
            index += 1;
        } else {
            let draw_function = draw_functions.get_mut(item.draw_function()).unwrap();
            draw_function.draw(world, render_pass, view, item);
            index += batch_range.len();
        }
    }
}

```

transparent_phase 从 DrawFunction 中取出 draw_functions,根据 PhaseItem 取出 id，由 id 获取渲染函数，渲染函数负责具体的绘制。
这里我们还不清楚是渲染函数怎么和 PhaseItem 联系起来。渲染函数同样是实现了 draw trait 的对象。bevy 引入了一个另一个 trait 自动实现 draw，并且通过这个 trait 将 PhaseItem 联系起来，这个 trait 就是 RenderCommand<P: PhaseItem>。我们可以看下 bevy ui render 具体的实现。

```rust
pub type DrawUi = (
    SetItemPipeline,
    SetUiViewBindGroup<0>,
    SetUiTextureBindGroup<1>,
    DrawUiNode,
);

pub struct SetUiViewBindGroup<const I: usize>;
impl<P: PhaseItem, const I: usize> RenderCommand<P> for SetUiViewBindGroup<I> {
    type Param = SRes<UiMeta>;
    type ViewQuery = Read<ViewUniformOffset>;
    type ItemQuery = ();

    fn render<'w>(
        _item: &P,
        view_uniform: &'w ViewUniformOffset,
        _entity: (),
        ui_meta: SystemParamItem<'w, '_, Self::Param>,
        pass: &mut TrackedRenderPass<'w>,
    ) -> RenderCommandResult {
        pass.set_bind_group(
            I,
            ui_meta.into_inner().view_bind_group.as_ref().unwrap(),
            &[view_uniform.offset],
        );
        RenderCommandResult::Success
    }
}
pub struct SetUiTextureBindGroup<const I: usize>;
impl<P: PhaseItem, const I: usize> RenderCommand<P> for SetUiTextureBindGroup<I> {
    type Param = SRes<UiImageBindGroups>;
    type ViewQuery = ();
    type ItemQuery = Read<UiBatch>;

    #[inline]
    fn render<'w>(
        _item: &P,
        _view: (),
        batch: &'w UiBatch,
        image_bind_groups: SystemParamItem<'w, '_, Self::Param>,
        pass: &mut TrackedRenderPass<'w>,
    ) -> RenderCommandResult {
        let image_bind_groups = image_bind_groups.into_inner();
        pass.set_bind_group(I, image_bind_groups.values.get(&batch.image).unwrap(), &[]);
        RenderCommandResult::Success
    }
}
pub struct DrawUiNode;
impl<P: PhaseItem> RenderCommand<P> for DrawUiNode {
    type Param = SRes<UiMeta>;
    type ViewQuery = ();
    type ItemQuery = Read<UiBatch>;

    #[inline]
    fn render<'w>(
        _item: &P,
        _view: (),
        batch: &'w UiBatch,
        ui_meta: SystemParamItem<'w, '_, Self::Param>,
        pass: &mut TrackedRenderPass<'w>,
    ) -> RenderCommandResult {
        pass.set_vertex_buffer(0, ui_meta.into_inner().vertices.buffer().unwrap().slice(..));
        pass.draw(batch.range.clone(), 0..1);
        RenderCommandResult::Success
    }
}

```

这里的 DrawUi 就是实际的绘制函数实现。好，我们这里已经弄清楚 node 的具体实现，这里我们总结一下这个流程。

- node trait 靠实现了 PhaseItem 的 RenderPhase<T>的驱动。
- RenderPhase<T> 从 DrawFunction<T>获得具体的渲染函数。
- 渲染函数由 RenderCommand<T>定义。
- 每个 RenderCommand 负责向 gpu 发送数据。

对应 beve ui render 就是

- UiPassNode 靠 RenderPhase<TransparentUi>驱动
- RenderPhase<TransparentUi> 从 DrawFunction<TransparentUi>获得具体的渲染函数
- DrawUi 为实际的渲染函数，
- SetItemPipeline，SetUiViewBindGroup，SetUiTextureBindGroup，DrawUiNode 负责向 gpu 发送数据

# bevy ui plugin

通过 bevy ui plugin 可以看下具体的实现。代码如下:

```rust
pub fn build_ui_render(app: &mut App) {
    load_internal_asset!(app, UI_SHADER_HANDLE, "ui.wgsl", Shader::from_wgsl);

    let Ok(render_app) = app.get_sub_app_mut(RenderApp) else {
        return;
    };

    render_app
        .init_resource::<SpecializedRenderPipelines<UiPipeline>>()
        .init_resource::<UiImageBindGroups>()
        .init_resource::<UiMeta>()
        .init_resource::<ExtractedUiNodes>()
        .allow_ambiguous_resource::<ExtractedUiNodes>()
        .init_resource::<DrawFunctions<TransparentUi>>()
        .add_render_command::<TransparentUi, DrawUi>()
        .add_systems(
            ExtractSchedule,
            (
                extract_default_ui_camera_view::<Camera2d>,
                extract_default_ui_camera_view::<Camera3d>,
                extract_uinodes.in_set(RenderUiSystem::ExtractNode),
                extract_uinode_borders.after(RenderUiSystem::ExtractAtlasNode),
                #[cfg(feature = "bevy_text")]
                extract_text_uinodes.after(RenderUiSystem::ExtractAtlasNode),
                extract_uinode_outlines.after(RenderUiSystem::ExtractAtlasNode),
            ),
        )
        .add_systems(
            Render,
            (
                queue_uinodes.in_set(RenderSet::Queue),
                sort_phase_system::<TransparentUi>.in_set(RenderSet::PhaseSort),
                prepare_uinodes.in_set(RenderSet::PrepareBindGroups),
            ),
        );

}

```

在 ui plugin 中可以看到手动添加了 DrawFunctions 资源和 TransparentUi，DrawUi 的 rendercommand。

# 简陋架构图

![bevy ui render 架构图](./images/bevy_ui_render.jpg 'bevy ui render 架构图')
图中红线为我们实现自定义渲染需要自己实现的 trait。

# 结语

该文章并未涉及从 ecs 中获取组件数据和渲染 app 的相关解释。
