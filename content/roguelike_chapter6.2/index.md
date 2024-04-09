+++
title = "roguelike_chapter6.2 游戏状态机"
date = 2024-03-30

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 添加一个游戏状态机

在 src/logic.rs 中新增一个 States，代码如下:

```rust
#[derive(States, Default, Clone, Eq, PartialEq, Debug, Hash)]
pub enum RunTurnState {
    #[default]
    PreRun,
    //等待输入
    AwaitingInput,
    PlayerTurn,
    MonsterTurn,
}
```

RunTurnState 的默认状态为 PreRun，添加一个系统 change_to_awaiting_input 让其运行到 AwaitingInput。代码如下:

```rust
pub fn change_to_awaiting_input(mut next_state: ResMut<NextState<RunTurnState>>) {
    next_state.set(RunTurnState::AwaitingInput);
}
```

将 change_to_awaiting_input 系统放入 LogicPlugin。代码如下:

```rust
app.add_systems(
    Update,
    (change_to_awaiting_input,).run_if(in_state(RunTurnState::PreRun)),
);

```

更改 user_input 系统，如果输入是有效的，则更改系统进入 PlayerTurn。代码如下:

```rust
pub fn user_input() {
    ...
    if x != 0 || y != 0 {
        next_state.set(RunTurnState::PlayerTurn);
    }
}

```

添加一个系统 change_to_monster_turn 让其运行到 MonsterTurn。代码如下:

```rust
pub fn change_to_monster_turn(mut next_state: ResMut<NextState<RunTurnState>>) {
    next_state.set(RunTurnState::MonsterTurn);
}
```

同样添加到 LogicPlugin，只是在 PlayerTurn 状态下运行。代码如下:

```rust
app.add_systems(
        Update,
        (change_to_monster_turn,).run_if(in_state(RunTurnState::PlayerTurn)),
    );
```

在 MonsterTurn 状态下添加 change_to_awaiting_input 系统，以便可以在怪物行动之后监控输入，代码如下：

```rust
app.add_systems(
    Update,
    (change_to_awaiting_input,).run_if(in_state(RunTurnState::MonsterTurn)),
);
```

更改 src/logic.rs 中 monster_ai 系统的调度，放入 RunTurnState::MonsterTurn 的状态中，代码如下:

```rust
app.add_systems(
    Update,
    (change_to_awaiting_input, monster_ai).run_if(in_state(RunTurnState::MonsterTurn)),
);
```

清除 FixUpdate 相关的代码。

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路

```

```
