+++
title = "roguelike_chapter7 ui"
date = 2024-04-11

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 添加字体

在https://timothyqiu.itch.io/vonwaon-bitmap地址下下载所需字体，解压缩放入src/assets/fonts中。

在 src/ui.rs 中定义 FontManager 管理项目用到的字体，代码如下:

```rust
#[derive(Resource, AssetCollection)]
pub struct FontManager {
    #[asset(path = "fonts/VonwaonBitmap-16pxLite.ttf")]
    pub font: Handle<Font>,
}
```

更改 src/loading.rs 中 LoadingPlugin 的 plugin 实现，在 load state 自动加载字体。代码如下:

```rust
impl Plugin for LoadingPlugin {
    fn build(&self, app: &mut App) {
        app.add_loading_state(
            LoadingState::new(GameState::Loading)
                .continue_to_state(GameState::Menu)
                .load_collection::<AudioAssets>()
                .load_collection::<TextureAssets>()
                .load_collection::<FontManager>(),
        );

        app.add_systems(Startup, setup);
    }
}
```

# 添加 hud

新增一个 TopUINode 组件，用来标识 hub 的顶层 node。代码如下:

```rust
#[derive(Component)]
pub struct TopUINode;
```

在 src/ui/hub.rs 中新增 spwan_bottom_hud 系统，用来生成整个 hub 结构。代码如下:

```rust
pub fn spwan_bottom_hud(mut commands: Commands, font_manager: Res<FontManager>) {
    commands
        // root node, just a black rectangle where the UI will be
        .spawn((
            NodeBundle {
                style: Style {
                    width: Val::Percent(100.0),
                    height: Val::Px(100.0),
                    position_type: PositionType::Absolute,
                    left: Val::Px(0.0),
                    bottom: Val::Px(0.0),
                    ..Default::default()
                },
                background_color: BackgroundColor(Color::rgb(0.0, 0.0, 0.0)),
                ..Default::default()
            },
            TopUINode,
        ));
}

```

在 src/ui/mod.rs 新增 InternalUiPlugin,代码如下:

```rust
pub struct InternalUiPlugin;

impl Plugin for InternalUiPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(GameState::Playing), (hub::spwan_bottom_hud,));
        app.add_systems(OnExit(GameState::Playing), (hub::clear_bottom_hud,));
    }
}
```

记得将 InternalUiPlugin 放入 src/lib.rs 的 GamePlugin 中。

# 添加血量条

定义两个组件 HPBar 和 HPText 标识要操作的进度条和血量文字。代码如下:

```rust
#[derive(Component)]
pub struct HPText;

#[derive(Component)]
pub struct HPBar;
```

更改 spwan_bottom_hud 系统，在子节点下生成血量文字和进度条。代码如下:

```rust
pub fn spwan_bottom_hud(mut commands: Commands, font_manager: Res<FontManager>) {
    commands
        // root node, just a black rectangle where the UI will be
        .spawn((
            NodeBundle {
                style: Style {
                    width: Val::Percent(100.0),
                    height: Val::Px(100.0),
                    position_type: PositionType::Absolute,
                    left: Val::Px(0.0),
                    bottom: Val::Px(0.0),
                    ..Default::default()
                },
                background_color: BackgroundColor(Color::rgb(0.0, 0.0, 0.0)),
                ..Default::default()
            },
            TopUINode,
        ))
        .with_children(|parent| {
            parent
                .spawn(NodeBundle {
                    style: Style {
                        width: Val::Percent(50.0),
                        height: Val::Percent(100.0),
                        border: UiRect::all(Val::Px(5.0)),
                        ..Default::default()
                    },
                    background_color: Color::rgb(0.65, 0.65, 0.65).into(),
                    ..Default::default()
                })
                // now inner rectangle
                .with_children(|parent| {
                    parent
                        .spawn(NodeBundle {
                            style: Style {
                                width: Val::Percent(100.0),
                                height: Val::Percent(100.0),
                                flex_direction: FlexDirection::Column,
                                ..Default::default()
                            },
                            background_color: Color::rgb(0.0, 0.0, 0.0).into(),
                            ..Default::default()
                        })
                        // top level with HP information
                        // here we will place both the HP text and the HP bar
                        .with_children(|parent| {
                            parent
                                .spawn(NodeBundle {
                                    style: Style {
                                        width: Val::Percent(100.0),
                                        height: Val::Percent(33.0),
                                        flex_direction: FlexDirection::Row,
                                        ..Default::default()
                                    },
                                    background_color: Color::rgb(0.0, 0.0, 0.0).into(),
                                    ..Default::default()
                                })
                                // container where to place the HP text
                                .with_children(|parent| {
                                    parent
                                        .spawn(NodeBundle {
                                            style: Style {
                                                width: Val::Percent(35.0),
                                                height: Val::Percent(100.0),
                                                ..Default::default()
                                            },
                                            background_color: Color::rgb(0.0, 0.0, 0.0).into(),
                                            ..Default::default()
                                        })
                                        // the actual HP text
                                        .with_children(|parent| {
                                            parent.spawn((
                                                TextBundle {
                                                    style: Style {
                                                        height: Val::Px(20. * 1.),
                                                        // Set height to font size * number of text lines
                                                        margin: UiRect {
                                                            left: Val::Auto,
                                                            right: Val::Auto,
                                                            bottom: Val::Auto,
                                                            top: Val::Auto,
                                                        },
                                                        ..Default::default()
                                                    },
                                                    text: Text::from_section(
                                                        "HP: 17 / 20".to_string(),
                                                        TextStyle {
                                                            font_size: 20.0,
                                                            font: font_manager.font.clone(),
                                                            color: Color::rgb(0.99, 0.99, 0.99),
                                                        },
                                                    ),
                                                    ..Default::default()
                                                },
                                                HPText,
                                            ));
                                        });
                                    // outside HP bar
                                    parent
                                        .spawn(NodeBundle {
                                            style: Style {
                                                width: Val::Percent(63.0),
                                                height: Val::Px(20. * 1.),
                                                border: UiRect::all(Val::Px(5.0)),
                                                margin: UiRect {
                                                    left: Val::Auto,
                                                    right: Val::Auto,
                                                    bottom: Val::Auto,
                                                    top: Val::Auto,
                                                },
                                                ..Default::default()
                                            },
                                            background_color: Color::rgb(0.5, 0.1, 0.1).into(),
                                            ..Default::default()
                                        })
                                        // inside HP bar
                                        .with_children(|parent| {
                                            parent.spawn((
                                                NodeBundle {
                                                    style: Style {
                                                        width: Val::Percent(50.0),
                                                        height: Val::Percent(100.0),
                                                        ..Default::default()
                                                    },
                                                    background_color: Color::rgb(0.99, 0.1, 0.1)
                                                        .into(),
                                                    ..Default::default()
                                                },
                                                HPBar,
                                            ));
                                        });
                                });
                        });
                });
        });
}

```

添加 update_hp_text_and_bar 系统，在玩家实体的 CombatStats 改变时更改 hub。代码如下：

```rust
fn update_hp_text_and_bar(
    mut text_query: Query<&mut Text, With<HPText>>,
    mut bar_query: Query<&mut Style, With<HPBar>>,
    q_combat_stats: Query<&CombatStats, (With<Player>, Changed<CombatStats>)>,
) {
    for combat_stats in q_combat_stats.iter() {
        let (current, max) = (combat_stats.hp, combat_stats.max_hp);

        // update HP text
        for mut text in text_query.iter_mut() {
            text.sections[0].value = format!("HP: {} / {}", current, max);
        }

        // update HP bar
        let bar_fill = (current as f32 / max as f32) * 100.0;
        for mut bar in bar_query.iter_mut() {
            bar.width = Val::Percent(bar_fill);
        }
    }
}
```

在 src/ui/hub.rs 下新增 HudPlugin,将上述用到的系统放入 HudPlugin 的 plugin 实现。代码如下:

```rust
pub struct HudPlugin;

impl Plugin for HudPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(GameState::Playing), (spwan_bottom_hud,));
        app.add_systems(OnExit(GameState::Playing), (clear_bottom_hud,));

        app.add_systems(
            Update,
            (update_hp_text_and_bar,).run_if(in_state(GameState::Playing)),
        );
    }
}

```

记得更改 InternalUiPlugin 的实现，代码如下:

```rust
impl Plugin for InternalUiPlugin {
    fn build(&self, app: &mut App) {
        app.add_plugins((HudPlugin,));
    }
}

```

# 添加消息日志

在 src/commom/mod.rs 中新增 GameLog 资源用来记录游戏中消息，比如人物攻击和人物死亡的消息。

```rust
#[derive(Resource, Default)]
pub struct GameLog {
    pub entries: Vec<String>,
}
```

在 src/logic.rs 的 setup_game 和 clear_game 系统分别生成和剔除 GameLog 资源。此处代码已省略。
在 src/ui/hub.rs 新增 LogUI 组件，用来标识操作 gamelogui 的组件。代码如下:

```rust
#[derive(Component)]
pub struct LogUI;
```

在 src/ui/hub.rs 中更改 spwan_bottom_hud 系统，添加 gamelog 消息的容器。代码如下:

```rust
        parent
                .spawn(NodeBundle {
                    style: Style {
                        width: Val::Percent(50.0),
                        height: Val::Percent(100.0),
                        border: UiRect::all(Val::Px(5.0)),
                        ..Default::default()
                    },
                    background_color: BackgroundColor(Color::rgb(0.65, 0.65, 0.65)),
                    ..Default::default()
                })
                // now inner rectangle
                .with_children(|parent| {
                    parent
                        .spawn(NodeBundle {
                            style: Style {
                                width: Val::Percent(100.0),
                                height: Val::Percent(100.0),
                                // align_items: AlignItems::Stretch,
                                ..Default::default()
                            },
                            background_color: BackgroundColor(Color::rgb(0.0, 0.0, 0.0)),
                            ..Default::default()
                        })
                        // text
                        .with_children(|parent| {
                            parent.spawn((
                                TextBundle {
                                    style: Style {
                                        margin: UiRect::all(Val::Px(5.0)),
                                        ..Default::default()
                                    },
                                    // Use `Text` directly
                                    text: Text {
                                        // Construct a `Vec` of `TextSection`s
                                        sections: vec![
                                            TextSection {
                                                value: "Log...\n".to_string(),
                                                style: TextStyle {
                                                    font: font_manager.font.clone(),
                                                    font_size: 20.0,
                                                    color: Color::YELLOW,
                                                },
                                            },
                                            TextSection {
                                                value: "Use the arrow keys to move.\n".to_string(),
                                                style: TextStyle {
                                                    font: font_manager.font.clone(),
                                                    font_size: 20.0,
                                                    color: Color::YELLOW,
                                                },
                                            },
                                            TextSection {
                                                value: "Bump into the enemies to attack them.\n"
                                                    .to_string(),
                                                style: TextStyle {
                                                    font: font_manager.font.clone(),
                                                    font_size: 20.0,
                                                    color: Color::YELLOW,
                                                },
                                            },
                                            TextSection {
                                                value: "Find the amulet to win the game.\n"
                                                    .to_string(),
                                                style: TextStyle {
                                                    font: font_manager.font.clone(),
                                                    font_size: 20.0,
                                                    color: Color::YELLOW,
                                                },
                                            },
                                        ],
                                        justify: JustifyText::Left,
                                        ..Default::default()
                                    },
                                    ..Default::default()
                                },
                                LogUI,
                            ));
                        });
                });
```

注意这段代码的顺序。它是和 health bar 同级的容器。
在 src/ui/hub.rs 添加 update_game_log，用来显示 gameLog 的更新，代码如下:

```rust
fn update_game_log(game_log: Res<GameLog>, mut text_query: Query<&mut Text, With<LogUI>>) {
    if !game_log.is_changed() {
        return;
    }

    let message_len = game_log.entries.len();
    let mut start = 0;
    let end = message_len;
    if message_len > 4 {
        start = end - 4;
    }

    for mut text in text_query.iter_mut() {
        for (i, entry) in game_log.entries.as_slice()[start..end].iter().enumerate() {
            text.sections[i].value = entry.clone();
        }
    }
}
```

# 记录攻击

更改 melee_combat 系统中 info!代码，这里将信息不打印在日志中，而是记录 GameLog 中，代码如下:

```rust
pub fn melee_combat(
    mut commands: Commands,
    q_wants_to_melee: Query<(&WantsToMelee, &Parent, Entity)>,
    mut q_combat_stats: Query<(&CombatStats, &Name, Option<&mut SufferDamage>)>,
    mut log: ResMut<GameLog>,
) {
    let mut damage_map: HashMap<Entity, Vec<i32>> = HashMap::default();

    for (wants_to_melee, parent, entity) in q_wants_to_melee.iter() {
        let (active, active_name, _) = q_combat_stats.get(parent.get()).unwrap();
        if active.hp < 0 {
            continue;
        }

        let (unactive, unactive_name, _) = q_combat_stats.get(wants_to_melee.target).unwrap();
        if unactive.hp < 0 {
            continue;
        }

        let damage = i32::max(0, active.power - unactive.defense);

        if damage == 0 {
            log.entries.push(format!(
                "{} is unable to hurt {}",
                active_name, unactive_name
            ))
        } else {
            log.entries.push(format!(
                "{} hits {}, for {} hp.",
                &active_name, &unactive_name, damage
            ));

            if let Some(tmp_damages) = damage_map.get_mut(&wants_to_melee.target) {
                tmp_damages.push(damage)
            } else {
                damage_map.insert(wants_to_melee.target, vec![damage]);
            }
        }

        commands.entity(entity).despawn_recursive();
    }

    for (entity, damages) in damage_map.into_iter() {
        let (_, _, suffer_damage) = q_combat_stats.get_mut(entity).unwrap();

        if let Some(mut suffer_damage) = suffer_damage {
            suffer_damage.amount.extend_from_slice(&damages);
        } else {
            commands
                .entity(entity)
                .insert(SufferDamage { amount: damages });
        }
    }
}

```

# 记录死亡

更改 delete_the_dead 在删除实体的同时，记录角色死亡。代码如下:

```rust
pub fn delete_the_dead(
    mut commands: Commands,
    q_combat_stats: Query<(&CombatStats, Entity, &Name)>,
    player_entity: Res<PlayerEntity>,
    mut next_state: ResMut<NextState<GameState>>,
    mut log: ResMut<GameLog>,
) {
    for (combat_stats, entity, name) in q_combat_stats.iter() {
        if combat_stats.hp <= 0 {
            if entity == player_entity.0 {
                next_state.set(GameState::Menu);
            } else {
                commands.entity(entity).despawn_recursive();

                log.entries.push(format!("{} is dead", &name));
            }
        }
    }
}
```

# 添加 Tooltips

ToolTips 显示最重要的参数因素有三个，一个是显示的位置,一个是整个地图的大小，一个是自身的长度。
先在 src/ui/tooltip.rs 添加两个组件用来标识要操作的 ui，代码如下:

```rust

#[derive(Component)]
struct ToolTipText;

#[derive(Component)]
struct ToolTipBox;
```

同时新增一个系统添加 TooltTip ui，代码如下:

```rust
fn spawn_tooltip_ui(mut commands: Commands, font_manager: Res<FontManager>) {
    commands
        // root node, just a black rectangle where the text will be
        .spawn((
            NodeBundle {
                // by default we set visible to false so it starts hidden
                visibility: Visibility::Hidden,
                style: Style {
                    width: Val::Px(300.0),
                    height: Val::Px(30.0),
                    position_type: PositionType::Absolute,
                    ..Default::default()
                },
                background_color: BackgroundColor(Color::rgb(0.0, 0.0, 0.0)),
                ..Default::default()
            },
            ToolTipBox,
        ))
        .with_children(|parent| {
            // text
            parent.spawn((
                TextBundle {
                    visibility: Visibility::Hidden,
                    style: Style {
                        height: Val::Px(20. * 1.),
                        margin: UiRect::all(Val::Auto),
                        ..Default::default()
                    },
                    text: Text::from_section(
                        "Goblin. HP: 2 / 2",
                        TextStyle {
                            font: font_manager.font.clone(),
                            font_size: 20.0,
                            color: Color::WHITE,
                        },
                    ),
                    ..Default::default()
                },
                ToolTipText,
            ));
        });
}


```

在同一目录下新增 TooltipsPlugin，负责相关的调度。代码如下：

```rust
pub struct TooltipsPlugin;

impl Plugin for TooltipsPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(GameState::Playing), (spawn_tooltip_ui,));
    }
}
```

在 src/ui.rs 的 InternalUiPlugin 添加 TooltipsPlugin，代码如下:

```rust
impl Plugin for InternalUiPlugin {
    fn build(&self, app: &mut App) {
        app.add_plugins((HudPlugin, TooltipsPlugin));
    }
}
```

在 src/map.rs 中新增一个组件标识 map。代码如下:

```rust
#[derive(Component)]
pub struct MapInstance;
```

记得在 map 的 spawn_tile 方法中生成 map 时插入 MapInstance 组件。
添加一个系统 update_tooltip 用来显示和更新 tooltip 的内容。代码如下:

```rust
fn update_tooltip(
    // need to get window dimensions
    wnds: Query<&Window, With<PrimaryWindow>>,
    // to get the mouse clicks
    buttons: Res<ButtonInput<MouseButton>>,
    // query to get camera transform
    q_camera: Query<(&Camera, &GlobalTransform), With<Camera2d>>,
    // query to get all the entities with Name component
    q_names: Query<(&Position, &Name, &CombatStats)>,
    // // query to get tooltip text and box
    mut text_box_query: ParamSet<(
        Query<(&mut Text, &mut Visibility), With<ToolTipText>>,
        Query<(&mut Style, &mut Visibility), With<ToolTipBox>>,
    )>,
    // query to get the player field of view
    player_fov_q: Query<&Viewshed, With<Player>>,
    q_map: Query<&GlobalTransform, With<MapInstance>>,
) {
    // if the user left clicks
    if buttons.just_pressed(MouseButton::Left) {
        // get the primary window
        let wnd = wnds.get_single().unwrap();

        // check if the cursor is in the primary window
        if let Some(pos) = wnd.cursor_position() {
            // assuming there is exactly one main camera entity, so this is OK
            let (camera, camera_transform) = q_camera.single();

            let map_wld = q_map.single().translation().truncate();

            // apply the camera transform
            let point_wld = camera.viewport_to_world_2d(camera_transform, pos).unwrap();

            // transform world coordinates to our grid
            let grid_x = (point_wld.x - map_wld.x) / SPRITE_SIZE[0] as f32;
            let grid_y = (point_wld.y - map_wld.y) / SPRITE_SIZE[1] as f32;

            let grid_position = Position {
                x: grid_x as i32,
                y: grid_y as i32,
            };

            // now we go through all the entities with name to see which one is the nearest
            // some variables placeholders to save the entity name and its health
            let mut good_click = false;
            let mut s = String::new();
            let mut maxh = 0;
            let mut currenth = 0;
            // obtain also player fov
            let player_fov = player_fov_q.single();

            q_names
                .iter()
                .filter(|(pos, _, _)| {
                    **pos == grid_position
                        && player_fov
                            .visible_tiles
                            .contains(&(Point::new(grid_position.x, grid_position.y)))
                })
                .for_each(|(_, name, combat_stats)| {
                    s = name.as_str().to_string();
                    good_click = true;
                    // if it also has health component

                    maxh = combat_stats.max_hp;
                    currenth = combat_stats.hp;
                });

            // update tooltip text
            for (mut text, mut visible) in text_box_query.p0().iter_mut() {
                if currenth > 0 {
                    text.sections[0].value = format!("{} HP: {} / {}", s, currenth, maxh);
                } else {
                    text.sections[0].value = format!("{}", s);
                }
                *visible = Visibility::Visible;
            }

            // update box position
            for (mut boxnode, mut visible) in text_box_query.p1().iter_mut() {
                if good_click {
                    boxnode.left = Val::Px(pos.x - 100.0);
                    boxnode.top = Val::Px(pos.y - 40.0);
                    *visible = Visibility::Visible;
                } else {
                    *visible = Visibility::Hidden;
                }
            }
        }
    }
}

```

添加 update_tooltip 的调度如下:

```rust
impl Plugin for TooltipsPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(GameState::Playing), (spawn_tooltip_ui,));
        app.add_systems(
            Update,
            (update_tooltip,).run_if(in_state(RunTurnState::AwaitingInput)),
        );
    }
}
```

添加一个自动关闭 Tooltip 的系统，在不是用户输入的时候主动隐藏弹窗。

```rust
fn hide_tooltip(
    mut text_box_query: ParamSet<(
        Query<&mut Visibility, With<ToolTipText>>,
        Query<&mut Visibility, With<ToolTipBox>>,
    )>,
) {
    // update tooltip visibility
    for mut visible in text_box_query.p0().iter_mut() {
        *visible = Visibility::Hidden;
    }

    // update box visibility
    for mut visible in text_box_query.p1().iter_mut() {
        *visible = Visibility::Hidden;
    }
}
```

添加相应的调度。代码如下:

```rust
impl Plugin for TooltipsPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(GameState::Playing), (spawn_tooltip_ui,));
        app.add_systems(
            Update,
            (update_tooltip,).run_if(in_state(RunTurnState::AwaitingInput)),
        );

        app.add_systems(OnExit(RunTurnState::AwaitingInput), (hide_tooltip,));
    }
}
```

# 添加一个新 feature

在 src/Cargo.toml 的 features 下新增一个 dev_not_editor,代码如下：

```rust
[features]
default = ['dev']
dev = ["bevy/dynamic_linking", "bevy_editor_pls"]
dev_not_editor = []
```

运行代码 cargo run --features dev_not_editor --no-default-features，会出现下图界面。

![运行界面](./images/first.png)

# 清除 tooptipui

在 src/ui/tooltip.rs 添加一个系统 clear_tooltip_ui，清除生成的 tooptip 组件,代码如下:

```rust
fn clear_tooltip_ui(mut commands: Commands, q_tooltip_box: Query<Entity, With<ToolTipBox>>) {
    for entity in q_tooltip_box.iter() {
        commands.entity(entity).despawn_recursive();
    }
}

```

在 TooltipsPlugin 中添加调度，代码如下:

```rust
impl Plugin for TooltipsPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(OnEnter(GameState::Playing), (spawn_tooltip_ui,));
        app.add_systems(
            Update,
            (update_tooltip,).run_if(in_state(RunTurnState::AwaitingInput)),
        );

        app.add_systems(OnExit(RunTurnState::AwaitingInput), (hide_tooltip,));
        app.add_systems(OnExit(GameState::Playing), (clear_tooltip_ui,));
    }
}
```

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路
- [BevyRoguelike](https://github.com/thephet/BevyRoguelike) bevy 实现版本
