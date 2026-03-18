+++
title = "从零开始bevy渲染器设计-01-具体案例"
date = 2026-03-01

[taxonomies]
tags = ["bevy", "renderer"]
+++

从零开始渲染器-01.
分析bevy引擎的具体案例。确认几何体，材质与纹理，光照，摄像机等对象。

<!-- more -->

# 代码

bevy examples下关于shapes的案例如下:

```rust
#[cfg(not(target_arch = "wasm32"))]
use bevy::{
    input::common_conditions::input_just_pressed,
    sprite_render::{Wireframe2dConfig, Wireframe2dPlugin},
};
use bevy::{input::common_conditions::input_toggle_active, prelude::*};

fn main() {
    let mut app = App::new();
    app.add_plugins((
        DefaultPlugins,
        #[cfg(not(target_arch = "wasm32"))]
        Wireframe2dPlugin::default(),
    ))
    .add_systems(Startup, setup);
    #[cfg(not(target_arch = "wasm32"))]
    app.add_systems(
        Update,
        toggle_wireframe.run_if(input_just_pressed(KeyCode::Space)),
    );
    app.add_systems(
        Update,
        rotate.run_if(input_toggle_active(false, KeyCode::KeyR)),
    );
    app.run();
}

const X_EXTENT: f32 = 1000.;
const Y_EXTENT: f32 = 150.;
const THICKNESS: f32 = 5.0;

fn setup(
    mut commands: Commands,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<ColorMaterial>>,
) {
    commands.spawn(Camera2d);

    let shapes = [
        meshes.add(Circle::new(50.0)),
        meshes.add(CircularSector::new(50.0, 1.0)),
        meshes.add(CircularSegment::new(50.0, 1.25)),
        meshes.add(Ellipse::new(25.0, 50.0)),
        meshes.add(Annulus::new(25.0, 50.0)),
        meshes.add(Capsule2d::new(25.0, 50.0)),
        meshes.add(Rhombus::new(75.0, 100.0)),
        meshes.add(Rectangle::new(50.0, 100.0)),
        meshes.add(RegularPolygon::new(50.0, 6)),
        meshes.add(Triangle2d::new(
            Vec2::Y * 50.0,
            Vec2::new(-50.0, -50.0),
            Vec2::new(50.0, -50.0),
        )),
        meshes.add(Segment2d::new(
            Vec2::new(-50.0, 50.0),
            Vec2::new(50.0, -50.0),
        )),
        meshes.add(Polyline2d::new(vec![
            Vec2::new(-50.0, 50.0),
            Vec2::new(0.0, -50.0),
            Vec2::new(50.0, 50.0),
        ])),
    ];
    let num_shapes = shapes.len();

    for (i, shape) in shapes.into_iter().enumerate() {
        // Distribute colors evenly across the rainbow.
        let color = Color::hsl(360. * i as f32 / num_shapes as f32, 0.95, 0.7);

        commands.spawn((
            Mesh2d(shape),
            MeshMaterial2d(materials.add(color)),
            Transform::from_xyz(
                // Distribute shapes from -X_EXTENT/2 to +X_EXTENT/2.
                -X_EXTENT / 2. + i as f32 / (num_shapes - 1) as f32 * X_EXTENT,
                Y_EXTENT / 2.,
                0.0,
            ),
        ));
    }

    let rings = [
        meshes.add(Circle::new(50.0).to_ring(THICKNESS)),
        // this visually produces an arc segment but this is not technically accurate
        meshes.add(Ring::new(
            CircularSector::new(50.0, 1.0),
            CircularSector::new(45.0, 1.0),
        )),
        meshes.add(CircularSegment::new(50.0, 1.25).to_ring(THICKNESS)),
        meshes.add({
            // This is an approximation; Ellipse does not implement Inset as concentric ellipses do not have parallel curves
            let outer = Ellipse::new(25.0, 50.0);
            let mut inner = outer;
            inner.half_size -= Vec2::splat(THICKNESS);
            Ring::new(outer, inner)
        }),
        // this is equivalent to the Annulus::new(25.0, 50.0) above
        meshes.add(Ring::new(Circle::new(50.0), Circle::new(25.0))),
        meshes.add(Capsule2d::new(25.0, 50.0).to_ring(THICKNESS)),
        meshes.add(Rhombus::new(75.0, 100.0).to_ring(THICKNESS)),
        meshes.add(Rectangle::new(50.0, 100.0).to_ring(THICKNESS)),
        meshes.add(RegularPolygon::new(50.0, 6).to_ring(THICKNESS)),
        meshes.add(
            Triangle2d::new(
                Vec2::Y * 50.0,
                Vec2::new(-50.0, -50.0),
                Vec2::new(50.0, -50.0),
            )
            .to_ring(THICKNESS),
        ),
    ];
    // Allow for 2 empty spaces
    let num_rings = rings.len() + 2;

    for (i, shape) in rings.into_iter().enumerate() {
        // Distribute colors evenly across the rainbow.
        let color = Color::hsl(360. * i as f32 / num_rings as f32, 0.95, 0.7);

        commands.spawn((
            Mesh2d(shape),
            MeshMaterial2d(materials.add(color)),
            Transform::from_xyz(
                // Distribute shapes from -X_EXTENT/2 to +X_EXTENT/2.
                -X_EXTENT / 2. + i as f32 / (num_rings - 1) as f32 * X_EXTENT,
                -Y_EXTENT / 2.,
                0.0,
            ),
        ));
    }

    let mut text = "Press 'R' to pause/resume rotation".to_string();
    #[cfg(not(target_arch = "wasm32"))]
    text.push_str("\nPress 'Space' to toggle wireframes");

    commands.spawn((
        Text::new(text),
        Node {
            position_type: PositionType::Absolute,
            top: px(12),
            left: px(12),
            ..default()
        },
    ));
}

#[cfg(not(target_arch = "wasm32"))]
fn toggle_wireframe(mut wireframe_config: ResMut<Wireframe2dConfig>) {
    wireframe_config.global = !wireframe_config.global;
}

fn rotate(mut query: Query<&mut Transform, With<Mesh2d>>, time: Res<Time>) {
    for mut transform in &mut query {
        transform.rotate_z(time.delta_secs() / 2.0);
    }
}
```

在这个案例中Mesh2d是几何体，MeshMaterial2d是材质。几何体和材质是如何被bevy 2d renderer所渲染。
Mesh2d存储的是Mesh的句柄，Mesh2d指向的几何体数据存储在Mesh的资源中。

RenderAssetPlugin负责从主世界获取资源到渲染世界。
在RenderPlugin中引入的MeshRenderAssetPlugin负责将Mesh资源转化为RenderMesh。

MeshMaterial2d存储的ColorMaterial的句柄，MeshMaterial2d指向的材质数据存储在ColorMaterial资源中。 Material2dPlugin负责处理类似ColorMaterial的资源。它将ColorMaterial转换为PreparedMaterial2d。

MeshMaterial2dPlugin中queue_material2d_meshes系统创建摄像机所观察到的所有要渲染的实体，并创建phase.
Transparent2d定义在bevy_core_pipelie中。main_opaque_pass_2d系统使用这些phase创建实际的pass，向gpu通信。
main_opaque_pass_2d系统的代码如下:

```rust
pub fn main_opaque_pass_2d(
    world: &World,
    view: ViewQuery<(
        &ExtractedCamera,
        &ExtractedView,
        &ViewTarget,
        &ViewDepthTexture,
    )>,
    opaque_phases: Res<ViewBinnedRenderPhases<Opaque2d>>,
    alpha_mask_phases: Res<ViewBinnedRenderPhases<AlphaMask2d>>,
    mut ctx: RenderContext,
) {
    let view_entity = view.entity();
    let (camera, extracted_view, target, depth) = view.into_inner();

    let (Some(opaque_phase), Some(alpha_mask_phase)) = (
        opaque_phases.get(&extracted_view.retained_view_entity),
        alpha_mask_phases.get(&extracted_view.retained_view_entity),
    ) else {
        return;
    };

    if opaque_phase.is_empty() && alpha_mask_phase.is_empty() {
        return;
    }

    #[cfg(feature = "trace")]
    let _span = info_span!("main_opaque_pass_2d").entered();

    let diagnostics = ctx.diagnostic_recorder();
    let diagnostics = diagnostics.as_deref();

    let color_attachments = [Some(target.get_color_attachment())];
    let depth_stencil_attachment = Some(depth.get_attachment(StoreOp::Store));

    let mut render_pass = ctx.begin_tracked_render_pass(RenderPassDescriptor {
        label: Some("main_opaque_pass_2d"),
        color_attachments: &color_attachments,
        depth_stencil_attachment,
        timestamp_writes: None,
        occlusion_query_set: None,
        multiview_mask: None,
    });
    let pass_span = diagnostics.pass_span(&mut render_pass, "main_opaque_pass_2d");

    if let Some(viewport) = camera.viewport.as_ref() {
        render_pass.set_camera_viewport(viewport);
    }

    if !opaque_phase.is_empty() {
        #[cfg(feature = "trace")]
        let _opaque_span = info_span!("opaque_main_pass_2d").entered();
        if let Err(err) = opaque_phase.render(&mut render_pass, world, view_entity) {
            error!("Error encountered while rendering the 2d opaque phase {err:?}");
        }
    }

    if !alpha_mask_phase.is_empty() {
        #[cfg(feature = "trace")]
        let _alpha_mask_span = info_span!("alpha_mask_main_pass_2d").entered();
        if let Err(err) = alpha_mask_phase.render(&mut render_pass, world, view_entity) {
            error!("Error encountered while rendering the 2d alpha mask phase {err:?}");
        }
    }

    pass_span.end(&mut render_pass);
}

```
