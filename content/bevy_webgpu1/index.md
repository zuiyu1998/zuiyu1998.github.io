+++
title = "从零开始画一个三角形"
date = 2024-04-13

[taxonomies]
tags = ["bevy", "webgpu"]
+++

在 bevy render 中探究 webgpu 的实现。

<!-- more -->

# 正常流程画一个三角形

- 设置顶点数据
- 设置 bindgroup
- 设置 管道
- 提交 gpu 执行

# bevy 案例

在 bevy 中实现一个三角形。代码如下:

```rust
use bevy::{prelude::*, sprite::MaterialMesh2dBundle};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, setup)
        .run();
}

fn setup(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    commands.spawn(Camera2dBundle::default());
    commands.spawn(MaterialMesh2dBundle {
        mesh: meshes.add(Triangle2d::default()).into(),
        transform: Transform::default().with_scale(Vec3::splat(128.)),
        material: materials.add(Color::RED),
        ..default()
    });
}

```

# 设置顶点数据

如何将 cpu 的数据转化为 gpu 的数据?
Triangle2d::default()是 cpu 的顶点数据，它是一个 mesh。但是在 MaterialMesh2dBundle 中 mesh 只是一个 handle。
以 MaterialMesh2dBundle 为切口全局搜索 Mesh2dHandle 的类型。extract_mesh2d 系统将 mesh 转换为 RenderMesh2dInstance。RenderMesh2dInstance 的定义如下:

```rust
pub struct RenderMesh2dInstance {
    pub transforms: Mesh2dTransforms,
    pub mesh_asset_id: AssetId<Mesh>,
    pub material_bind_group_id: Material2dBindGroupId,
    pub automatic_batching: bool,
}
```

mesh_asset_id 是 mesh 的 handle。
这里还没有发现如何将 cpu 数据转换为 gpu 数据。
继续全局搜索 RenderMesh2dInstances，会发现 queue_material2d_meshes 系统。

```rust
let Some(mesh) = render_meshes.get(mesh_instance.mesh_asset_id) else {
        continue;
    };
```

在这段代码中使用了 mesh_asset_id 获取到了 gpu 数据。render_meshes 的定义是 RenderAssets<Mesh>。这个跟 Assets<Mesh>很像。RenderAssets 需要一个实现了 RenderAsset 的 trait。观察 mesh 的 RenderAsset 实现。代码如下:

```rust
impl RenderAsset for Mesh {
    type PreparedAsset = GpuMesh;
    type Param = (
        SRes<RenderDevice>,
        SRes<RenderAssets<Image>>,
        SResMut<MeshVertexBufferLayouts>,
    );

    fn asset_usage(&self) -> RenderAssetUsages {
        self.asset_usage
    }

    /// Converts the extracted mesh a into [`GpuMesh`].
    fn prepare_asset(
        self,
        (render_device, images, ref mut mesh_vertex_buffer_layouts): &mut SystemParamItem<
            Self::Param,
        >,
    ) -> Result<Self::PreparedAsset, PrepareAssetError<Self>> {
        let morph_targets = match self.morph_targets.as_ref() {
            Some(mt) => {
                let Some(target_image) = images.get(mt) else {
                    return Err(PrepareAssetError::RetryNextUpdate(self));
                };
                Some(target_image.texture_view.clone())
            }
            None => None,
        };

        let vertex_buffer_data = self.get_vertex_buffer_data();
        let vertex_buffer = render_device.create_buffer_with_data(&BufferInitDescriptor {
            usage: BufferUsages::VERTEX,
            label: Some("Mesh Vertex Buffer"),
            contents: &vertex_buffer_data,
        });

        let buffer_info = if let Some(data) = self.get_index_buffer_bytes() {
            GpuBufferInfo::Indexed {
                buffer: render_device.create_buffer_with_data(&BufferInitDescriptor {
                    usage: BufferUsages::INDEX,
                    contents: data,
                    label: Some("Mesh Index Buffer"),
                }),
                count: self.indices().unwrap().len() as u32,
                index_format: self.indices().unwrap().into(),
            }
        } else {
            GpuBufferInfo::NonIndexed
        };

        let mesh_vertex_buffer_layout =
            self.get_mesh_vertex_buffer_layout(mesh_vertex_buffer_layouts);

        let mut key_bits = BaseMeshPipelineKey::from_primitive_topology(self.primitive_topology());
        key_bits.set(
            BaseMeshPipelineKey::MORPH_TARGETS,
            self.morph_targets.is_some(),
        );

        Ok(GpuMesh {
            vertex_buffer,
            vertex_count: self.count_vertices() as u32,
            buffer_info,
            key_bits,
            layout: mesh_vertex_buffer_layout,
            morph_targets,
        })
    }
}
```

可以看到这里的 mesh 转换为了 gpuMesh。但是这个 trait 是什么时候执行的？
全局搜索::<Mesh>，观察 prepare_assets 系统，代码如下:

```rust
pub fn prepare_assets<A: RenderAsset>(
    mut extracted_assets: ResMut<ExtractedAssets<A>>,
    mut render_assets: ResMut<RenderAssets<A>>,
    mut prepare_next_frame: ResMut<PrepareNextFrameAssets<A>>,
    param: StaticSystemParam<<A as RenderAsset>::Param>,
) {
    let mut param = param.into_inner();
    let queued_assets = std::mem::take(&mut prepare_next_frame.assets);
    for (id, extracted_asset) in queued_assets {
        if extracted_assets.removed.contains(&id) {
            continue;
        }

        match extracted_asset.prepare_asset(&mut param) {
            Ok(prepared_asset) => {
                render_assets.insert(id, prepared_asset);
            }
            Err(PrepareAssetError::RetryNextUpdate(extracted_asset)) => {
                prepare_next_frame.assets.push((id, extracted_asset));
            }
        }
    }

    for removed in extracted_assets.removed.drain(..) {
        render_assets.remove(removed);
    }

    for (id, extracted_asset) in extracted_assets.extracted.drain(..) {
        match extracted_asset.prepare_asset(&mut param) {
            Ok(prepared_asset) => {
                render_assets.insert(id, prepared_asset);
            }
            Err(PrepareAssetError::RetryNextUpdate(extracted_asset)) => {
                prepare_next_frame.assets.push((id, extracted_asset));
            }
        }
    }
}

```

这个系统将 ExtractedAssets 转化为 RenderAssets，全局搜 ExtractedAssets，可以发现这个 extract_render_asset 系统，代码如下：

```rust
fn extract_render_asset<A: RenderAsset>(mut commands: Commands, mut main_world: ResMut<MainWorld>) {
    main_world.resource_scope(
        |world, mut cached_state: Mut<CachedExtractRenderAssetSystemState<A>>| {
            let (mut events, mut assets) = cached_state.state.get_mut(world);

            let mut changed_assets = HashSet::default();
            let mut removed = Vec::new();

            for event in events.read() {
                #[allow(clippy::match_same_arms)]
                match event {
                    AssetEvent::Added { id } | AssetEvent::Modified { id } => {
                        changed_assets.insert(*id);
                    }
                    AssetEvent::Removed { .. } => {}
                    AssetEvent::Unused { id } => {
                        changed_assets.remove(id);
                        removed.push(*id);
                    }
                    AssetEvent::LoadedWithDependencies { .. } => {
                        // TODO: handle this
                    }
                }
            }

            let mut extracted_assets = Vec::new();
            for id in changed_assets.drain() {
                if let Some(asset) = assets.get(id) {
                    let asset_usage = asset.asset_usage();
                    if asset_usage.contains(RenderAssetUsages::RENDER_WORLD) {
                        if asset_usage == RenderAssetUsages::RENDER_WORLD {
                            if let Some(asset) = assets.remove(id) {
                                extracted_assets.push((id, asset));
                            }
                        } else {
                            extracted_assets.push((id, asset.clone()));
                        }
                    }
                }
            }

            commands.insert_resource(ExtractedAssets {
                extracted: extracted_assets,
                removed,
            });
            cached_state.state.apply(world);
        },
    );
}

```

这个将 Assets 转换为 ExtractedAssets。
现在回顾一下 bevy render subapp 的调度:

- ExtractSchedule
- Render

将以上的系统划入调度：

- ExtractSchedule
  - extract_render_asset
- Render
  - Queue
    - prepare_assets
  - QueueMeshes
    - queue_material2d_meshes

好，cpu 的数据转换为 gpu 数据了。

# 设置 bindgroup

在 queue_material2d_meshes 绑定了 bind_group,代码如下：

```rust
mesh_instance.material_bind_group_id = material2d.get_bind_group_id();

```

而 material2d 来源于 PreparedMaterial2d,它有一个泛型，在这里是 MaterialMesh2dBundle 的 material 字段。查找 get_bind_group_id 的实现，全局搜 prepare_materials_2d，这个系统是唯一用到 PreparedMaterial2d。这里主要关注如下函数。代码如下:

```rust
fn prepare_material2d<M: Material2d>(
    material: &M,
    render_device: &RenderDevice,
    images: &RenderAssets<Image>,
    fallback_image: &FallbackImage,
    pipeline: &Material2dPipeline<M>,
) -> Result<PreparedMaterial2d<M>, AsBindGroupError> {
    let prepared = material.as_bind_group(
        &pipeline.material2d_layout,
        render_device,
        images,
        fallback_image,
    )?;
    Ok(PreparedMaterial2d {
        bindings: prepared.bindings,
        bind_group: prepared.bind_group,
        key: prepared.data,
        depth_bias: material.depth_bias(),
    })
}

```

这里主要关注的是 as_bind_group 函数，这个函数来自于 AsBindGroup trait。这里关注 ColorMaterial 的 AsBindGroup trait 实现。

```rust
/// A [2d material](Material2d) that renders [2d meshes](crate::Mesh2dHandle) with a texture tinted by a uniform color
#[derive(Asset, AsBindGroup, Reflect, Debug, Clone)]
#[reflect(Default, Debug)]
#[uniform(0, ColorMaterialUniform)]
pub struct ColorMaterial {
    pub color: Color,
    #[texture(1)]
    #[sampler(2)]
    pub texture: Option<Handle<Image>>,
}

```

这里是直接派生的，所以直接看下 AsBindGroup 的定义。

```rust
pub trait AsBindGroup {
    /// Data that will be stored alongside the "prepared" bind group.
    type Data: Send + Sync;

    /// label
    fn label() -> Option<&'static str> {
        None
    }

    /// Creates a bind group for `self` matching the layout defined in [`AsBindGroup::bind_group_layout`].
    fn as_bind_group(
        &self,
        layout: &BindGroupLayout,
        render_device: &RenderDevice,
        images: &RenderAssets<Image>,
        fallback_image: &FallbackImage,
    ) -> Result<PreparedBindGroup<Self::Data>, AsBindGroupError> {
        let UnpreparedBindGroup { bindings, data } =
            Self::unprepared_bind_group(self, layout, render_device, images, fallback_image)?;

        let entries = bindings
            .iter()
            .map(|(index, binding)| BindGroupEntry {
                binding: *index,
                resource: binding.get_binding(),
            })
            .collect::<Vec<_>>();

        let bind_group = render_device.create_bind_group(Self::label(), layout, &entries);

        Ok(PreparedBindGroup {
            bindings,
            bind_group,
            data,
        })
    }

    /// Returns a vec of (binding index, `OwnedBindingResource`).
    /// In cases where `OwnedBindingResource` is not available (as for bindless texture arrays currently),
    /// an implementor may define `as_bind_group` directly. This may prevent certain features
    /// from working correctly.
    fn unprepared_bind_group(
        &self,
        layout: &BindGroupLayout,
        render_device: &RenderDevice,
        images: &RenderAssets<Image>,
        fallback_image: &FallbackImage,
    ) -> Result<UnpreparedBindGroup<Self::Data>, AsBindGroupError>;

    /// Creates the bind group layout matching all bind groups returned by [`AsBindGroup::as_bind_group`]
    fn bind_group_layout(render_device: &RenderDevice) -> BindGroupLayout
    where
        Self: Sized,
    {
        render_device.create_bind_group_layout(
            Self::label(),
            &Self::bind_group_layout_entries(render_device),
        )
    }

    /// Returns a vec of bind group layout entries
    fn bind_group_layout_entries(render_device: &RenderDevice) -> Vec<BindGroupLayoutEntry>
    where
        Self: Sized;
}

```

# 设置 pipeline 渲染管道

pipeline 的绑定同样是在 queue_material2d_meshes 系统中进行，这是注意 pipeline 同样是资源。

# 提交 gpu 执行

在 RenderMesh2dInstances 中保存了 mesh_id，bindgroupid，最后需要将这些数据发送给 gpu。这里面重要的是 RenderMesh2dInstances 的 draw_func id。这是实际写入的实现。定义如下:

```rust
type DrawMaterial2d<M> = (
    SetItemPipeline,
    SetMesh2dViewBindGroup<0>,
    SetMesh2dBindGroup<1>,
    SetMaterial2dBindGroup<M, 2>,
    DrawMesh2d,
);
```

依次是设置 Pipeline，Mesh2dViewBindGroup，Mesh2dBindGroup，Material2dBindGroup，DrawMesh2d。将这些写入后，bevy 绘制三角形的流程也完成了。
