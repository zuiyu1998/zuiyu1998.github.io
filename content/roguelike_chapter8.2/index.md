+++
title = "roguelike_chapter8.2 重构ui"
date = 2024-04-28

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)
参考仓库: [BevyRoguelike](https://github.com/thephet/BevyRoguelike.git)

<!-- more -->

# 添加 bevy_egui 依赖

在 src/Cargo.toml 添加 bevy_egui 的依赖，代码如下:

```rust
bevy_egui = { version = "0.27" }

```

修改调试的工具，由 bevy_editor_pls 改为 bevy_inspector_egui。代码如下:

```rust
[features]
default = ['dev']
dev = ["bevy/dynamic_linking", "bevy-inspector-egui"]

[dependencies]
bevy-inspector-egui = { version = "0.24", optional = true }

```

# 新增 CoreUiPlugin

在 src/core/mod.rs 中新增 InternalCorePlugin，用来存储基础的 plugin。代码如下:

```rust
pub struct InternalCorePlugin;

impl Plugin for InternalCorePlugin {
    fn build(&self, app: &mut App) {}
}
```

在 src/core/ui.rs 中新增 CoreUiPlugin，这个 plugin 用来引入 bevy_egui 和 ui 相关的功能。代码如下:

```rust
use bevy::prelude::*;
use bevy_egui::{egui, EguiContexts};

pub struct CoreUiPlugin;

impl Plugin for CoreUiPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(Startup, configure_visuals_system);

        #[cfg(not(feature = "dev"))]
        {
            use bevy_egui::EguiPlugin;
            app.add_plugins((EguiPlugin));
        }
    }
}

fn configure_visuals_system(mut contexts: EguiContexts) {
    contexts.ctx_mut().set_visuals(egui::Visuals {
        window_rounding: 0.0.into(),
        window_shadow: egui::epaint::Shadow::NONE,
        ..Default::default()
    });
}
```

在 src/ui/mod.rs 的 InternalCorePlugin 中添加 CoreUiPlugin。代码如下:

```rust
impl Plugin for InternalCorePlugin {
    fn build(&self, app: &mut App) {
        app.add_plugins(CoreUiPlugin);
    }
}

```

记得将 InternalCorePlugin 放入 GamePlugin 中。

# 重构菜单页面

在 src/menu.rs 中删除其他代码只保留 MenuPlugin，新增一个 MenuUiState 结构，代码如下：

```rust
pub struct MenuUiState {
    item_list: Vec<MenuItem>,
}
#[derive(Debug, Clone, Event)]
pub struct MenuItem {
    item_type: MenuItemType,
}

#[derive(Debug, Clone)]
pub enum MenuItemType {
    Playing,
}
```

MenuItem 为自定义的事件，用于菜单 Click 触发。为 MenuItem 添加 playing 构造函数。代码如下:

```rust
impl MenuItem {
    pub fn playing() -> Self {
        MenuItem {
            item_type: MenuItemType::Playing,
        }
    }
}
```

为 MenuItemType 实现 ToString，方便将 MenuItemType 转化为 String。代码如下：

```rust
impl ToString for MenuItemType {
    fn to_string(&self) -> String {
        match *self {
            MenuItemType::Playing => format!("Playing"),
        }
    }
}
```

为 MenuUiState 新增 Default trait 的实现。代码如下:

```rust
impl Default for MenuUiState {
    fn default() -> Self {
        MenuUiState {
            item_list: vec![MenuItem::playing()],
        }
    }
}
```

新增一个系统 show_menu，用来渲染菜单列表。代码如下:

```rust
fn show_menu(mut contexts: EguiContexts, mut menu_item_ew: EventWriter<MenuItem>) {
    let ui_state = MenuUiState::default();

    egui::Window::new("Menu")
        .title_bar(false)
        .anchor(Align2::CENTER_CENTER, [0.0, 0.0])
        .show(contexts.ctx_mut(), |ui| {
            for item in ui_state.item_list.iter() {
                let button = ui.add(egui::Button::new(item.item_type.to_string()));

                if button.clicked() {
                    menu_item_ew.send(item.clone());
                }
            }
        });
}
```

这里使用 egui::Window 渲染了一个窗口，通过 anchor 设置在了屏幕中间。在窗口中间渲染了整个菜单列表。当按钮按下会触发对应事件。

新增一个系统 handle_menu_item 处理按下的事件，代码如下:

```rust
fn handle_menu_item(
    mut menu_item_er: EventReader<MenuItem>,
    mut app_state_manager: AppStateManager,
) {
    for e in menu_item_er.read() {
        match e.item_type {
            MenuItemType::Playing => {
                app_state_manager.start_game();
            }
        }
    }
}
```

当 Playing 事件触发，项目进入游戏状态。

将 show_menu ，handle_menu_item 放入 MenuPlugin 中，并注册 MenuItem 事件，代码如下:

```rust
impl Plugin for MenuPlugin {
    fn build(&self, app: &mut App) {
        app.add_event::<MenuItem>();

        app.add_systems(
            Update,
            (show_menu, handle_menu_item).run_if(in_state(AppState::Menu)),
        );
    }
}
```

> 注意这里没有对样式进行任何优化

# 重构 Hud 页面

新增 show_hud 用于显示 hud 数据。代码如下:

```rust
fn show_hud(
    q_stats: Query<&CombatStats, With<Player>>,
    game_log: Res<GameLog>,
    mut contexts: EguiContexts,
) {
    let stats = q_stats.single();

    let length = game_log.entries.len();

    let mut logs = vec![];

    if length <= 4 {
        for i in 0..length {
            logs.push(game_log.entries[i].clone());
        }
    } else {
        for i in length - 4..length {
            logs.push(game_log.entries[i].clone());
        }
    }

    egui::TopBottomPanel::bottom("my_bottom")
        .min_height(100.0)
        .show(contexts.ctx_mut(), |ui| {
            ui.columns(2, |columns| {
                egui::ScrollArea::vertical().show(&mut columns[0], |ui| {
                    for log in logs.iter() {
                        ui.label(log);
                    }
                });

                egui::Frame::none().show(&mut columns[1], |ui| {
                    ui.horizontal(|ui| {
                        let progress = stats.hp as f32 / stats.max_hp as f32;

                        ui.label(format!("{}/{}", stats.hp, stats.max_hp));

                        ui.add(egui::ProgressBar::new(progress));
                    });
                });
            });
        });
}

```

# 重构 Tooltip

新增 ToolTipEntity 用来保存选中的实体。代码定义如下:

```rust
#[derive(Resource, Default)]
pub struct ToolTipEntity(pub Option<Entity>);

```

更改 src/ui/tooltip.rs 中 update_tooltip 系统，代码如下:

```rust
fn update_tooltip(
    // need to get window dimensions
    wnds: Query<&Window, With<PrimaryWindow>>,
    // to get the mouse clicks
    buttons: Res<ButtonInput<MouseButton>>,
    // query to get camera transform
    q_camera: Query<(&Camera, &GlobalTransform), With<MainCamera>>,
    // query to get all the entities with Name component
    q_names: Query<(&Position, Entity), With<Enemy>>,
    // query to get the player field of view
    player_fov_q: Query<&Viewshed, With<Player>>,
    q_map: Query<&GlobalTransform, With<MapInstance>>,
    mut app_state_manager: AppStateManager,
    mut tooltip_entity: ResMut<ToolTipEntity>,
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
            let grid_x =
                (point_wld.x - map_wld.x + SPRITE_SIZE[0] as f32 / 2.0) / SPRITE_SIZE[0] as f32;
            let grid_y =
                (point_wld.y - map_wld.y + SPRITE_SIZE[0] as f32 / 2.0) / SPRITE_SIZE[1] as f32;

            let grid_position = Position {
                x: grid_x as i32,
                y: grid_y as i32,
            };

            // now we go through all the entities with name to see which one is the nearest
            // some variables placeholders to save the entity name and its health
            let mut good_click = false;
            let mut tooltip_entity_tmp: Option<Entity> = None;

            // obtain also player fov
            let player_fov = player_fov_q.single();

            q_names
                .iter()
                .filter(|(pos, _)| {
                    **pos == grid_position
                        && player_fov
                            .visible_tiles
                            .contains(&(Point::new(grid_position.x, grid_position.y)))
                })
                .for_each(|(_, entity)| {
                    good_click = true;
                    tooltip_entity_tmp = Some(entity);
                });

            if good_click {
                app_state_manager.start_tootip();

                tooltip_entity.0 = tooltip_entity_tmp;
            }
        }
    }
}

```

update_tootip 现在更新 ToolTipEntity 的实体。同时在选中成功后更改游戏状态。

新增 show_tooltip 系统 用来生成 ui，代码如下:

```rust
fn show_tooltip(
    tooltip_entity: ResMut<ToolTipEntity>,
    q_enemy: Query<(&CombatStats, &Name)>,
    mut contexts: EguiContexts,
) {
    let entity = tooltip_entity.0.clone().unwrap();

    if let Ok((stats, name)) = q_enemy.get(entity) {
        egui::show_tooltip(contexts.ctx_mut(), egui::Id::new("my_tooltip"), |ui| {
            ui.label(format!("{} {}:{}", name, stats.hp, stats.max_hp));
        });
    }
}

```

当 tooltip_entity 存在实体时生成敌人的部分信息。删除未被使用的系统和组件。现在 TooltipPlugin 组件如下:

```rust
impl Plugin for TooltipsPlugin {
    fn build(&self, app: &mut App) {
        app.init_resource::<ToolTipEntity>();

        app.add_systems(
            Update,
            (update_tooltip,).run_if(in_state(GameState::Playing)),
        );

        app.add_systems(
            Update,
            (change_to_playing, show_tooltip).run_if(in_state(GameState::ToolTip)),
        );
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

```

```
