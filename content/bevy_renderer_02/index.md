+++
title = "从零开始bevy渲染器设计-02-世界通信"
date = 2026-04-13

[taxonomies]
tags = ["bevy", "renderer"]
+++

这是探究 bevy renderer 渲染器的一系列文章。这篇文章谈论主世界和渲染世界的通信。

<!-- more -->

# ExtractPlugin

渲染世界和主世界通信的代码在ExtractPlugin的实现中。主要代码如下:

```rust
pub fn extract(main_world: &mut World, render_world: &mut World) {
    // temporarily add the app world to the render world as a resource
    let scratch_world = main_world.remove_resource::<ScratchMainWorld>().unwrap();
    let inserted_world = core::mem::replace(main_world, scratch_world.0);
    render_world.insert_resource(MainWorld(inserted_world));
    render_world.run_schedule(ExtractSchedule);

    // move the app world back, as if nothing happened.
    let inserted_world = render_world.remove_resource::<MainWorld>().unwrap();
    let scratch_world = core::mem::replace(main_world, inserted_world.0);
    main_world.insert_resource(ScratchMainWorld(scratch_world));
}
```

渲染世界与主世界的通信在与调度ExtractSchedule。在ExtractSchedule中渲染世界会复制主世界的数据。

- 提取资源
- 提取组件

这些分别实现在extract_resource, extract_param中。
