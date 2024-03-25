+++
title = "roguelike_chapter2 地图"
date = 2024-01-24

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 重构 对已有代码进行重新组织

在 src 下新增 logic 文件，将 src/render 中的代码迁移过来，包括 left_movement，user_input，spawn_character，spawn_terminal。将这些系统放入 LogicPlugin 中，并在 GamePlugin 中添加此 plugin。

# 添加地图

在 src 下新增 map.rs,新增一个 enum 用来表示 map 中的 tile 类型，代码如下:

```rust
#[derive(PartialEq, Copy, Clone)]
pub enum TileType {
    Wall,
    Floor,
}

```

> PartialEq trait 可以更方便的使用 =

为 TileType 添加一个函数，获取对应的 Renderable,以便在 Terminal 中显示。代码如下：

```rust
impl TileType {
    pub fn get_renderable(&self) -> Renderable {
        match self {
            TileType::Floor => Renderable {
                fg: Color::rgb(0.5, 0.5, 0.5),
                bg: Color::rgb(0., 0., 0.),
                glyph: '.',
            },
            TileType::Wall => Renderable {
                fg: Color::rgb(0.0, 1.0, 0.0),
                bg: Color::rgb(0., 0., 0.),
                glyph: '#',
            },
        }
    }
}
```

新增一个地图的定义,代码如下:

```rust
#[derive(Resource)]
pub struct Map {
    pub width: usize,
    pub heigth: usize,
    pub tiles: Vec<TileType>,
}
```

为 Map 定义一个函数，将二维的坐标转换为一维的索引，代码如下:

```rust
 pub fn xy_idx(&self, x: i32, y: i32) -> usize {
    (y as usize * self.width) + x as usize
}
```

然后新增一个函数 new_map 来得到一个地图。代码如下:

```rust
pub fn new_map() -> Map {
    let mut map = Map {
        tiles: vec![TileType::Floor; 80 * 50],
        width: 80,
        heigth: 50,
    };

    for x in 0..80 {
        let start_idx = map.xy_idx(x, 0);
        let end_idx = map.xy_idx(x, 49);

        map.tiles[start_idx] = TileType::Wall;
        map.tiles[end_idx] = TileType::Wall;
    }
    for y in 0..50 {
        let start_idx = map.xy_idx(0, y);
        let end_idx = map.xy_idx(79, y);

        map.tiles[start_idx] = TileType::Wall;
        map.tiles[end_idx] = TileType::Wall;
    }

    let mut rng = RandomNumberGenerator::new();

    for _i in 0..400 {
        let x = rng.roll_dice(1, 79);
        let y = rng.roll_dice(1, 49);
        let idx = map.xy_idx(x, y);
        if idx != map.xy_idx(40, 25) {
            map.tiles[idx] = TileType::Wall;
        }
    }

    map
}
```

其中 RandomNumberGenerator 为[bracket-random](https://github.com/amethyst/bracket-lib)中定义的随机数生成器。这里需要引入 bracket-random 的依赖。

添加一个 MapPlugin，将 map 注册到 app 中，代码如下:

```rust
pub struct MapPlugin;

impl Plugin for MapPlugin {
    fn build(&self, app: &mut App) {
        app.init_resource::<Map>();
    }
}
```

在 src/logic.rs 中重命名 spawn_terminal 为 setup_game，在原有的代码基础上添加一个地图资源。代码如下:

```rust
pub fn setup_game(mut commands: Commands, mut map: ResMut<Map>) {
    let terminal = Terminal::new([80, 50]).with_border(Border::single_line());

    commands.spawn((
        // Spawn the terminal bundle from our terminal
        TerminalBundle::from(terminal),
        Name::new("Terminal"),
    ));

    *map = new_map();
}
```

最后在 src/render.rs 中，在渲染 renderable 之前，先渲染 map,代码如下:

```rust
pub fn render(
    q_q_position_and_renderable: Query<(&Position, &Renderable)>,
    mut q_render_terminal: Query<&mut Terminal>,
    map: Res<Map>,
) {
    let mut term = match q_render_terminal.get_single_mut() {
        Ok(term) => term,
        Err(_) => return,
    };
    term.clear();

    for x in 0..map.width as i32 {
        for y in 0..map.heigth as i32 {
            let idx = map.xy_idx(x, y);

            let tile: FormattedTile = map.tiles[idx].get_renderable().into();

            term.put_char([x, y], tile);
        }
    }

    q_q_position_and_renderable
        .iter()
        .for_each(|(position, renderable)| {
            let tile: FormattedTile = renderable.clone().into();

            term.put_char(position.clone(), tile);
        });
}

```

运行代码，右键左上角的按钮，点击 playing，右键左上角的按钮，会出现下图界面。

![运行界面](./images/first.png)

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
