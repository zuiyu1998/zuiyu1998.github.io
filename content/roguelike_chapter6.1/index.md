+++
title = "roguelike_chapter6.1 stage和states"
date = 2024-03-30

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# stage

bevy 是一个 ecs 架构的游戏的引擎。stage 是系统（s）调度的最小单元。bevy 中系统并不是同时运行的，只有在同一个 stage 中的系统且没有排序的系统才会并行。
在默认的调度器（Schedule）中提供了许多默认的 state,它们有着明确的先后顺序。如下:

- Startup (首次运行)
  - PreStartup
  - Startup
  - PostStartup
- Main (loop)
  - First
  - PreUpdate
  - StateTransition (状态转换)
  - RunFixedMainLoop （fixed）
    - FixedFirst
    - FixedPreUpdate
    - FixeUpdate
    - FixePostUpdate
    - FixeLast
  - Update
  - SpawnScene
  - PostUpdate
  - Last

这里面比较重要的两个 stage 是 StateTransition 和 FixeUpdate，这个是不太常用，属于进阶的内容。FixeUpdate 是根据帧率运行的 stage，StateTransition 是用于状态转换的 stage。

# states

states 是和 StateTransition 紧密联系的。专门用于状态转换的系统在 StateTransition 中运行。

states 可以选择性的运行某些系统，是有限状态机的一种实现。这是 bevy 的官方支持，当然也可以自己实现。
这个系统的主要功能是可以批量的自定义 stage，每一个 state 会定义三个自定义 stage，这个极大的减轻我们的工作量。实现这个的功能的代码如下:

```rust
pub fn apply_state_transition<S: States>(world: &mut World) {
    // We want to take the `NextState` resource,
    // but only mark it as changed if it wasn't empty.
    let Some(mut next_state_resource) = world.get_resource_mut::<NextState<S>>() else {
        return;
    };
    if let Some(entered) = next_state_resource.bypass_change_detection().0.take() {
        next_state_resource.set_changed();
        match world.get_resource_mut::<State<S>>() {
            Some(mut state_resource) => {
                if *state_resource != entered {
                    let exited = mem::replace(&mut state_resource.0, entered.clone());
                    world.send_event(StateTransitionEvent {
                        before: exited.clone(),
                        after: entered.clone(),
                    });
                    // Try to run the schedules if they exist.
                    world.try_run_schedule(OnExit(exited.clone())).ok();
                    world
                        .try_run_schedule(OnTransition {
                            from: exited,
                            to: entered.clone(),
                        })
                        .ok();
                    world.try_run_schedule(OnEnter(entered)).ok();
                }
            }
            None => {
                world.insert_resource(State(entered.clone()));
                world.try_run_schedule(OnEnter(entered)).ok();
            }
        };
    }
}
```

states 的每一个状态会添加三个 stage，onEnter，onExit,OnTransition 分别是三个状态，进入时，离开时，和状态转换。StateTransitionEvent 则是状态转换的事件。
通过 states，可以方便的实现在游戏中，在主菜单等状态的 stage,并为对应的 stage 实现系统。

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路
