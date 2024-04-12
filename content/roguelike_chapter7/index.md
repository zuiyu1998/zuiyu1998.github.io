+++
title = "roguelike_chapter7 ui"
date = 2024-04-11

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 添加一个 uiTerminal

在 src/consts.rs 中新增三个常量，代码如下：

```rust
//视图大小
pub const VIEWPORT_SIZE: [usize; 2] = [80, 40];
//ui大小
pub const UI_SIZE: [usize; 2] = [VIEWPORT_SIZE[0], 8];
//地图大小
pub const GAME_SIZE: [usize; 2] = [VIEWPORT_SIZE[0], VIEWPORT_SIZE[1] - UI_SIZE[1]];

```

重构 map 的 default trait 实现，使用 GAME_SIZE 声明的长宽，代码如下：

```rust
impl Default for Map {
    fn default() -> Self {
        let size = GAME_SIZE[0] * GAME_SIZE[1];
        let width = GAME_SIZE[0];
        let height = GAME_SIZE[1];

        let map = Map {
            tiles: vec![TileType::Wall; size],
            width,
            height,
            revealed_tiles: vec![false; size],
            visible_tiles: vec![false; size],
            blocked: vec![false; size],
            tile_content: vec![Vec::new(); size],
        };

        map
    }
}
```

在 src/render.rs 中新增 GameTerminal 组件，这个组件用来标识 game terminal,修改 render 系统中 q_render_terminal 参数，添加 With<GameTerminal>过滤。代码如下:

```rust
mut q_render_terminal: Query<&mut Terminal, With<GameTerminal>>,

```

修改 src/logic.rs 中的 setup_game 系统，在生成 TerminalBundle 的组件的同时添加 GameTerminal，修改后的代码如下:

```rust
commands.spawn((
    // Spawn the terminal bundle from our terminal
    TerminalBundle::from(terminal),
    Name::new("GameTerminal"),
    GameTerminal,
));
```

注意，同时需要修改 clear_game 系统。
新建 src/ui.rs 文件，新增 UiTerminal 组件和 InternalUIPlugin。代码如下:

```rust
#[derive(Component)]
pub struct UiTerminal;

pub struct InternalUIPlugin;

impl Plugin for InternalUIPlugin {
    fn build(&self, app: &mut App) {}
}

```

在 src/ui.rs 中添加 setup_ui_terminal 系统，代码如下:

```rust
pub fn setup_ui_terminal(mut commands: Commands) {
    let terminal = Terminal::new(UI_SIZE).with_border(Border::single_line());
    let term_y = -(VIEWPORT_SIZE[1] as i32 / 2) + UI_SIZE[1] as i32 / 2 - 1;

    let mut terminal_bundle = TerminalBundle::from(terminal);

    terminal_bundle = terminal_bundle.with_position([0, term_y]);

    commands.spawn((terminal_bundle, Name::new("UiTerminal"), UiTerminal));
}

```

with_position 为 terminal 的相对位置，更改 src/logic.rs 中生成 terminal 的代码添加相对位置,代码如下:

```rust
let terminal = Terminal::new([GAME_SIZE[0], GAME_SIZE[1]]).with_border(Border::single_line());

let term_y = VIEWPORT_SIZE[1] as i32 / 2 - GAME_SIZE[1] as i32 / 2;

let mut terminal_bundle = TerminalBundle::from(terminal);

terminal_bundle = terminal_bundle.with_position([0, term_y]);

commands.spawn((terminal_bundle, Name::new("GameTerminal"), GameTerminal));

```

# 添加一个血量栏

在 src/ui.rs 添加一个系统 ui_render 系统，用来绘制玩家的状态。代码如下:

```rust
pub fn ui_render(
    q_player: Query<&CombatStats, With<Player>>,
    mut q_render_terminal: Query<&mut Terminal, With<UiTerminal>>,
) {
    let mut term = match q_render_terminal.get_single_mut() {
        Ok(term) => term,
        Err(_) => return,
    };
    term.clear();

    if let Ok(stats) = q_player.get_single() {
        let hp_string = format!(
            "HP: {} / {}",
            stats.hp.to_string(),
            stats.max_hp.to_string()
        );
        let y = term.side_index(Side::Top) as i32;
        let bar_width = term.width() as i32 - 20;
        let bar_x = term.width() as i32 - bar_width - 1;
        let hp_x = bar_x - hp_string.len() as i32 - 1;

        let fg_color = Color::YELLOW;
        term.put_string([hp_x, y], hp_string.as_str().fg(fg_color));

        term.put_color([bar_x, y], Color::RED.bg());

        progress(&mut term, bar_x, y, stats.max_hp, stats.hp);
    }
}

fn progress(terminal: &mut Terminal, start: i32, y: i32, max: i32, cul: i32) {
    for x in start..start + max {
        terminal.put_color([x, y], Color::rgb(127.0, 33.0, 33.0).bg());
    }

    for x in start..start + cul {
        terminal.put_color([x, y], Color::RED.bg());
    }
}

```

# 添加一个日志记录器

新增一个资源 GameLog 来存储所有的日志信息。为 GameLog 添加默认的日志信息，在 setup_ui_terminal 中添加相应的资源，代码如下:

```rust
commands.insert_resource(GameLog {
    entries: vec!["Welcome to Rusty Roguelike".to_string()],
});
```

在 ui_render 系统中渲染最新的 5 条 log。代码如下:

```rust
pub fn ui_render(
    q_player: Query<&CombatStats, With<Player>>,
    mut q_render_terminal: Query<&mut Terminal, With<UiTerminal>>,
    game_log: Res<GameLog>,
) {
    let mut term = match q_render_terminal.get_single_mut() {
        Ok(term) => term,
        Err(_) => return,
    };
    term.clear();

    let mut y = 0;

    for log in game_log.entries.iter().rev() {
        if y < 5 {
            term.put_string([4, y], log.fg(Color::WHITE));
        }
        y += 1;
    }

    if let Ok(stats) = q_player.get_single() {
        let hp_string = format!(
            "HP: {} / {}",
            stats.hp.to_string(),
            stats.max_hp.to_string()
        );
        let y = term.side_index(Side::Top) as i32;
        let bar_width = term.width() as i32 - 20;
        let bar_x = term.width() as i32 - bar_width - 1;
        let hp_x = bar_x - hp_string.len() as i32 - 1;

        let fg_color = Color::YELLOW;
        term.put_string([hp_x, y], hp_string.as_str().fg(fg_color));

        term.put_color([bar_x, y], Color::RED.bg());

        progress(&mut term, bar_x, y, stats.max_hp, stats.hp);
    }
}

```

# 清除生成的资源

添加一个系统 clear_ui_terminal，用于 setup_ui_terminal 系统实体和资源的回收，代码如下：

```rust
pub fn clear_ui_terminal(mut commands: Commands, q_terminal: Query<Entity, With<UiTerminal>>) {
    for entity in q_terminal.iter() {
        commands.entity(entity).despawn_recursive();
    }

    commands.remove_resource::<GameLog>();
}

```

# 记录攻击日志

修改 src/logic 下的 melee_combat 系统，在计算伤害后，向日志中写入数据。代码如下:

```rust
pub fn melee_combat(
    mut commands: Commands,
    q_wants_to_melee: Query<(&WantsToMelee, &Parent, Entity)>,
    mut q_combat_stats: Query<(&CombatStats, &Name, Option<&mut SufferDamage>)>,
    mut game_log: ResMut<GameLog>,
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
            game_log.entries.push(format!(
                "{} is unable to hurt {}",
                active_name, unactive_name
            ));
        } else {
            game_log.entries.push(format!(
                "{} hits {}, for {} hp.",
                active_name, unactive_name, damage
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

# 记录敌人的死亡

修改 src/logic.rs 中的 delete_the_dead 系统，代码如下:

```rust
pub fn delete_the_dead(
    mut commands: Commands,
    q_combat_stats: Query<(&CombatStats, Entity, &Name)>,
    player_entity: Res<PlayerEntity>,
    mut next_state: ResMut<NextState<GameState>>,
    mut game_log: ResMut<GameLog>,
) {
    for (combat_stats, entity, name) in q_combat_stats.iter() {
        if combat_stats.hp <= 0 {
            if entity == **player_entity {
                next_state.set(GameState::Menu);
            } else {
                game_log.entries.push(format!("{} is dead", name));
                commands.entity(entity).despawn_recursive();
            }
        }
    }
}
```

# 修复 RunTurnState 未及时更新的 bug

在角色死亡时，需要将 RunTurnState 更新为 PreRun，否则其他系统会继续运行，造成 bug。修改 delete_the_dead 系统，代码如下:

```rust
pub fn delete_the_dead(
    mut commands: Commands,
    q_combat_stats: Query<(&CombatStats, Entity, &Name)>,
    player_entity: Res<PlayerEntity>,
    mut next_state: ResMut<NextState<GameState>>,
    mut next_run_state: ResMut<NextState<RunTurnState>>,
    mut game_log: ResMut<GameLog>,
) {
    for (combat_stats, entity, name) in q_combat_stats.iter() {
        if combat_stats.hp <= 0 {
            if entity == **player_entity {
                next_state.set(GameState::Menu);
                next_run_state.set(RunTurnState::PreRun);
            } else {
                game_log.entries.push(format!("{} is dead", name));
                commands.entity(entity).despawn_recursive();
            }
        }
    }
}
```

运行代码，右键左上角的按钮，点击 playing，右键左上角的按钮，会出现下图界面。

![运行界面](./images/first.png)

# 添加 Tooltips

ToolTips 显示最重要的参数因素有三个，一个是显示的位置,一个是整个地图的大小，一个是自身的长度。新建一个系统 render_tooltips 处理这些逻辑，代码如下:

```

```

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路

```

```
