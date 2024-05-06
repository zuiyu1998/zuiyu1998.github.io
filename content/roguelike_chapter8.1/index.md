+++
title = "roguelike_chapter8 道具"
date = 2024-04-28

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)
参考仓库: [BevyRoguelike](https://github.com/thephet/BevyRoguelike.git)

<!-- more -->

# 更换图片资源

现在使用的图片资源空白部分会显示黑色，这是因为图片资源空白部分未做透明处理，这里替换为新的资源。地址为:https://github.com/thephet/BevyRoguelike/blob/main/assets/terminal8x8_transparent.png。
将下载后的图片放入 src/assets/texturs 中。
更改 src/loading.rs 中的 TextureAssets 资源，代码如下:

```rust
#[derive(AssetCollection, Resource)]
pub struct TextureAssets {
    #[asset(path = "textures/bevy.png")]
    pub bevy: Handle<Image>,
    #[asset(path = "textures/github.png")]
    pub github: Handle<Image>,
    #[asset(path = "textures/terminal8x8_transparent.png")]
    pub terminal: Handle<Image>,
}
```

将 TextureAssets 的视线迁入 src/theme.rs 中，实现更好的代码结构。

# 使用有限状态机管理游戏状态

将原先的 GameState 重命名 AppState，并使用全局替换。将 AppState 中的 Playing 改为 InGame，并再次使用全局替换。更改后的代码如下：

```rust
#[derive(States, Default, Clone, Eq, PartialEq, Debug, Hash)]
enum AppState {
    // During the loading State the LoadingPlugin will load our assets
    #[default]
    Loading,
    // During this State the actual game logic is executed
    InGame,
    // Here the menu is drawn and waiting for player interaction
    Menu,
}
```

在 src/state.rs 中新增 GameState。代码如下:

```rust
#[derive(States, Default, Clone, Eq, PartialEq, Debug, Hash)]
enum GameState {
    #[default]
    None,
    Playing,
    Pause,
    ToolTip,
}

```

在 src/state.rs 中新增 StatePlugin，用于设置整个项目的 state。代码如下:

```rust
pub struct StatePlugin;

impl Plugin for StatePlugin {
    fn build(&self, app: &mut App) {
        app.init_state::<GameState>();
    }
}
```

在 src/lib.rs 的 GamePlugin 中添加 StatePlugin，并将 AppState 的代码迁入 state.rs 中。
在 src/state.rs 的中新增一个 参数 AppStateManager，用来管理整个 app 的状态。代码如下:

```rust
#[derive(SystemParam)]
pub struct AppStateManager<'w> {
    app_next_state: ResMut<'w, NextState<AppState>>,
    game_next_state: ResMut<'w, NextState<GameState>>,
}
```

为 AppStateManager 添加两个函数 start_game 和 end_game。
start_game 的定义如下:

```rust
impl<'w> AppStateManager<'w> {
    pub fn start_game(&mut self) {
        self.app_next_state.set(AppState::InGame);
        self.game_next_state.set(GameState::Playing);
    }
}
```

在 start_game 函数调用后，更改 AppState 为 InGame，GameState 为 Playing，项目进入游戏界面;

end_game 函数的定义如下:

```rust
   pub fn end_game(&mut self) {
        self.app_next_state.set(AppState::Menu);
        self.game_next_state.set(GameState::None);
    }
```

在 end_game 函数调用后，更改 AppState 为 Menu，GameState 为 None,项目进入主菜单;

更改 src/menu.rs 的 click_play_button 系统，使用 AppStateManager 管理游戏状态。代码如下:

```rust
fn click_play_button(
    mut app_state_manager: AppStateManager,
    mut interaction_query: Query<
        (
            &Interaction,
            &mut BackgroundColor,
            &ButtonColors,
            Option<&ChangeState>,
            Option<&OpenLink>,
        ),
        (Changed<Interaction>, With<Button>),
    >,
) {
    for (interaction, mut color, button_colors, change_state, open_link) in &mut interaction_query {
        match *interaction {
            Interaction::Pressed => {
                if let Some(_) = change_state {
                    app_state_manager.start_game();
                } else if let Some(link) = open_link {
                    if let Err(error) = webbrowser::open(link.0) {
                        warn!("Failed to open link {error:?}");
                    }
                }
            }
            Interaction::Hovered => {
                *color = button_colors.hovered.into();
            }
            Interaction::None => {
                *color = button_colors.normal.into();
            }
        }
    }
}


```

更改 src/commom.rs 中 delete_the_dead 系统，使用 AppStateManager 管理游戏状态。

```rust
pub fn delete_the_dead(
    mut commands: Commands,
    q_combat_stats: Query<(&CombatStats, Entity, &Name)>,
    player_entity: Res<PlayerEntity>,
    mut log: ResMut<GameLog>,
    mut app_state_manager: AppStateManager,
) {
    for (combat_stats, entity, name) in q_combat_stats.iter() {
        if combat_stats.hp <= 0 {
            if entity == player_entity.0 {
                app_state_manager.end_game();
            } else {
                commands.entity(entity).despawn_recursive();

                log.entries.push(format!("{} is dead", &name));
            }
        }
    }
}
```

# 使用有限状态机管理敌人状态

在 src/Cargo.html 中添加有限状态机的依赖，代码如下:

```toml
seldom_state = { version = "0.10" }
```

在 src/lib.rs 中新增 StateMachinePlugin。StateMachinePlugin 为 seldom_state 中定义的 plugin。
在 src/common/state_machine.rs 中新增 Idle 和 Follow 组件，代码如下:

```rust
#[derive(Debug, Component, Clone)]
#[component(storage = "SparseSet")]
pub struct Idle;

#[derive(Debug, Component, Clone)]
#[component(storage = "SparseSet")]
pub struct Follow;
```

SparseSet 声明 是为了频繁插入删除组件的一种优化。

在 src/enemy.rs 中添加一个函数用来添加敌人的状态机。代码如下:

```rust
pub fn add_state_machine(commands: &mut EntityCommands, _enemy: EnemyType) {
    commands.insert((
        StateMachine::default()
            .trans::<Idle, _>(look_player, Follow)
            .trans::<Follow, _>(look_player.not(), Idle)
            .set_trans_logging(true),
        Idle,
        EnemyTimer::default(),
    ));
}
```

这里添加了一个状态机组件，该状态机组件有两个转换条件。定义 idle -> follw 和 follow->idle 的转换条件。look_player 的定义如下:

```rust
fn look_player(
    In(entity): In<Entity>,
    q_enemy: Query<&Viewshed, With<Enemy>>,
    player_position: Res<PlayerPosition>,
) -> bool {
    if let Ok(viewshed) = q_enemy.get(entity) {
        if viewshed
            .visible_tiles
            .contains(&Point::new(player_position.0.x, player_position.0.y))
        {
            return true;
        } else {
            return false;
        }
    } else {
        return false;
    }
}
```

look_player 用于判断敌人是否已经察觉玩家。如果是，返回 true。

添加一个敌人行动频率的组件 EnemyTimer，用于执行敌人 ai 的逻辑。代码如下:

```rust
#[derive(Debug, Component, Deref, DerefMut)]
pub struct EnemyTimer(Timer);
```

更改 enemy_ai 的实现。代码如下:

```rust
fn enemy_ai(
    mut commands: Commands,
    mut q_enemy: Query<
        (&mut Viewshed, &mut Position, &Name, Entity, &mut EnemyTimer),
        (With<Enemy>, With<Follow>),
    >,
    player_position: Res<PlayerPosition>,
    player_entity: Res<PlayerEntity>,
    mut map: ResMut<Map>,
    time: Res<Time>,
) {
    for (mut viewshed, mut position, name, entity, mut timer) in q_enemy.iter_mut() {
        timer.tick(time.delta());

        if !timer.just_finished() {
            continue;
        }

        info!("{} shouts insults", name);

        let distance = DistanceAlg::Pythagoras.distance2d(
            Point::new(position.x, position.y),
            Point::new(player_position.0.x, player_position.0.y),
        );

        if distance < 1.5 {
            let player = player_entity.0.clone();

            commands.entity(entity).with_children(|parent| {
                parent.spawn(WantsToMelee { target: player });
            });

            return;
        }

        let path = a_star_search(
            map.xy_idx(position.x, position.y) as i32,
            map.xy_idx(player_position.0.x, player_position.0.y) as i32,
            &mut *map,
        );
        if path.success && path.steps.len() > 1 {
            position.x = path.steps[1] as i32 % (map.width as i32);
            position.y = path.steps[1] as i32 / (map.width as i32);
            viewshed.dirty = true;
        }
    }
}
```

该系统现在只会在 enemy 实体有 follow 组件的时候执行。清除 RunTurnState 的相关代码，现在已经不需要 RunTurnState 执行敌人的 ai。

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路
- [BevyRoguelike](https://github.com/thephet/BevyRoguelike) bevy 实现版本
