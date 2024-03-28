+++
title = "roguelike_chapter4 添加视野"
date = 2024-01-24

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 重构 map

将上期提到的 apply_horizontal_tunnel，apply_vertical_tunnel，new_map_rooms_and_corridors 放入 map 的 impl 中，方便后续的使用，代码如下:

```rust
fn apply_horizontal_tunnel(&mut self, x1: i32, x2: i32, y: i32) {
    for x in x1.min(x2)..=x1.max(x2) {
        let idx = self.xy_idx(x, y);
        if idx > 0 && idx < self.width * self.heigth {
            self.tiles[idx] = TileType::Floor;
        }
    }
}

fn apply_room_to_map(&mut self, room: &Rect) {
    for y in room.y1 + 1..=room.y2 {
        for x in room.x1 + 1..=room.x2 {
            let idx = self.xy_idx(x, y);
            self.tiles[idx] = TileType::Floor;
        }
    }
}

fn apply_vertical_tunnel(&mut self, y1: i32, y2: i32, x: i32) {
    for y in y1.min(y2)..=y1.max(y2) {
        let idx = self.xy_idx(x, y);
        if idx > 0 && idx < self.width * self.heigth {
            self.tiles[idx] = TileType::Floor;
        }
    }
}

pub fn new_map_rooms_and_corridors() -> (Map, Vec<Rect>) {
    let mut map = Map::default();

    let mut rooms: Vec<Rect> = Vec::new();
    const MAX_ROOMS: i32 = 30;
    const MIN_SIZE: i32 = 6;
    const MAX_SIZE: i32 = 10;

    let mut rng = RandomNumberGenerator::new();

    for _ in 0..MAX_ROOMS {
        let w = rng.range(MIN_SIZE, MAX_SIZE);
        let h = rng.range(MIN_SIZE, MAX_SIZE);
        let x = rng.roll_dice(1, 80 - w - 1) - 1;
        let y = rng.roll_dice(1, 50 - h - 1) - 1;
        let new_room = Rect::new(x, y, w, h);
        let mut ok = true;
        for other_room in rooms.iter() {
            if new_room.intersect(other_room) {
                ok = false
            }
        }
        //没有覆盖的room才能更新地图
        if ok {
            map.apply_room_to_map(&new_room);

            if !rooms.is_empty() {
                let (new_x, new_y) = new_room.center();
                let (prev_x, prev_y) = rooms[rooms.len() - 1].center();
                if rng.range(0, 2) == 1 {
                    map.apply_horizontal_tunnel(prev_x, new_x, prev_y);
                    map.apply_vertical_tunnel(prev_y, new_y, new_x);
                } else {
                    map.apply_vertical_tunnel(prev_y, new_y, prev_x);
                    map.apply_horizontal_tunnel(prev_x, new_x, new_y);
                }
            }

            rooms.push(new_room);
        }
    }

    (map, rooms)
}
```

# 添加有限的视野

尽管地图上的元素早已生成，但是玩家不应该直接知晓地图上的所有信息，从未知到已知，玩家才有探索感。
添加一个新的依赖:

```rust
bracket-pathfinding = { version = "0.8" }

```

在 src/map.rs 中添加一个新的组件 Viewshed,代码如下:

```rust
///视野
#[derive(Component)]
pub struct Viewshed {
    pub visible_tiles: Vec<Position>,
    pub range: i32,
}
```

在 src/logic.rs 中的 setup_game 为 player 添加视野组件，代码如下:

```rust
commands.spawn_empty().insert((
        Position {
            x: first_room_centerr.0,
            y: first_room_centerr.1,
        },
        Renderable {
            glyph: '@',
            fg: Color::YELLOW,
            bg: Color::BLACK,
        },
        Player {},
        Viewshed {
            visible_tiles: vec![],
            range: 8,
        },
    ));
```

# 为 Map 实现 Algorithm2D 和 BaseMap

Algorithm2D 和 BaseMap 为 bracket-pathfinding crate 中的 trait。需要先行添加,代码如下：

```rust
impl Algorithm2D for Map {
    fn dimensions(&self) -> Point {
        Point::new(self.width, self.height)
    }
}

impl BaseMap for Map {
    fn is_opaque(&self, idx: usize) -> bool {
        self.tiles[idx as usize] == TileType::Wall
    }
}
```

> ps 注意 height 的拼写错误

Algorithm2D 和 BaseMap 是 bracket-pathfinding 中的 trait，可以帮忙生成 Viewshed 的可见 tile。

# 添加一个系统用来更新 Viewshed

在 src/map.rs 中添加一个 update_viewshed 系统，用来实时更新 viewshed,代码如下:

```rust
pub fn update_view(
    mut q_player_view: Query<(&Position, &mut Viewshed), With<Player>>,
    map: Res<Map>,
) {
    for (pos, mut viewshed) in q_player_view.iter_mut() {
        viewshed.visible_tiles.clear();
        viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
        viewshed
            .visible_tiles
            .retain(|p| p.x >= 0 && p.x < map.width as i32 && p.y >= 0 && p.y < map.height as i32);
    }
}
```

将 update_view 添加到 MapPlugin 中,代码如下:

```rust
app.add_systems(PreUpdate, (update_view,));

```

# 更改 render 函数

修改 src/render.rs 中的 render 函数，不渲染整个地图数据，而是渲染可见的数据。代码更改如下:

```rust
pub fn render(
    q_position_and_renderable: Query<(&Position, &Renderable)>,
    q_viewshed: Query<&Viewshed>,
    mut q_render_terminal: Query<&mut Terminal>,
    map: Res<Map>,
) {
    let mut term = match q_render_terminal.get_single_mut() {
        Ok(term) => term,
        Err(_) => return,
    };
    term.clear();

    for view in q_viewshed.iter() {
        render_view(view, &map, &mut *term);
    }

    q_position_and_renderable
        .iter()
        .for_each(|(position, renderable)| {
            let tile: FormattedTile = renderable.clone().into();

            term.put_char(position.clone(), tile);
        });
}

```

这里引入了一个新的函数，用来渲染 view,代码如下:

```rust
fn render_view(view: &Viewshed, map: &Map, terminal: &mut Terminal) {
    for p in view.visible_tiles.iter() {
        let idx = map.xy_idx(p.x, p.y);
        let tile: FormattedTile = map.tiles[idx].get_renderable().into();

        terminal.put_char([p.x, p.y], tile);
    }
}
```

# 缓存 view 已经探明的地图数据

在 map 中添加新的字段 revealed_tiles,当为 true 是，表示 tile 已被探明。revealed_tiles 为地图大小，默认为 false。
Map 的 default 实现如下:

```rust
impl Default for Map {
    fn default() -> Self {
        let map = Map {
            tiles: vec![TileType::Wall; 80 * 50],
            width: 80,
            height: 50,
            revealed_tiles: vec![false; 80 * 50],
        };

        map
    }
}
```

更改 update_view 系统，在获取可见 tile 的同时缓存数据，代码如下:

```rust
pub fn update_view(mut q_player_view: Query<(&Position, &mut Viewshed)>, mut map: ResMut<Map>) {
    for (pos, mut viewshed) in q_player_view.iter_mut() {
        viewshed.visible_tiles.clear();
        viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
        viewshed
            .visible_tiles
            .retain(|p| p.x >= 0 && p.x < map.width as i32 && p.y >= 0 && p.y < map.height as i32);

        for tile in viewshed.visible_tiles.iter() {
            let idx = map.xy_idx(tile.x, tile.y);

            map.revealed_tiles[idx] = true;
        }
    }
}
```

将地图中缓存的数据显示，这需要更改 render 函数，代码如下:

```rust
pub fn render(
    q_position_and_renderable: Query<(&Position, &Renderable)>,
    q_viewshed: Query<&Viewshed>,
    mut q_render_terminal: Query<&mut Terminal>,
    map: Res<Map>,
) {
    let mut term = match q_render_terminal.get_single_mut() {
        Ok(term) => term,
        Err(_) => return,
    };
    term.clear();

    //渲染地图的缓存信息
    for x in 0..map.width {
        for y in 0..map.height {
            let idx = map.xy_idx(x as i32, y as i32);

            if map.revealed_tiles[idx] {
                let tile: FormattedTile = map.tiles[idx].get_renderable().into();

                term.put_char([x, y], tile);
            }
        }
    }

    for view in q_viewshed.iter() {
        render_view(view, &map, &mut *term);
    }

    q_position_and_renderable
        .iter()
        .for_each(|(position, renderable)| {
            let tile: FormattedTile = renderable.clone().into();

            term.put_char(position.clone(), tile);
        });
}

```

# 减少 view 的更新频率

在玩家位置不改变的时候，view 并不需要更新。在 Viewshed 添加一个字段保存这个状态。代码如下:

```rust
#[derive(Component)]
pub struct Viewshed {
    pub visible_tiles: Vec<Point>,
    pub range: i32,
    pub dirty: bool,
}
```

在用户输入的时候让 dirty 为 true，只有为 true 的时候才更新 Viewshed。user_input 的更改如下:

```rust
pub fn user_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut q_player: Query<(&mut Position, &mut Viewshed), With<Player>>,
    map: Res<Map>,
) {
    let mut x = 0;
    let mut y = 0;

    if keyboard_input.just_pressed(KeyCode::KeyA) {
        x -= 1;
    }

    if keyboard_input.just_pressed(KeyCode::KeyD) {
        x += 1;
    }

    if keyboard_input.just_pressed(KeyCode::KeyW) {
        y += 1;
    }

    if keyboard_input.just_pressed(KeyCode::KeyS) {
        y -= 1;
    }

    for (mut position, mut viewshed) in q_player.iter_mut() {
        position.movement(x, y, &map, &mut viewshed);
    }
}
```

movement 的更改如下:

```rust
    pub fn movement(&mut self, delta_x: i32, delta_y: i32, map: &Map, view: &mut Viewshed) {
        let next_x = 79.min(0.max(self.x + delta_x));
        let next_y = 79.min(0.max(self.y + delta_y));

        let idx = map.xy_idx(next_x, next_y);

        if map.tiles[idx] == TileType::Floor {
            self.x = next_x;
            self.y = next_y;

            view.dirty = true;
        }
    }
```

# 更改已探明但是未看见 tile 的颜色

在 Map 中新增一个 visible_tiles 字段，保存正在探明的 tile 的状态。代码如下:

```rust
#[derive(Resource)]
pub struct Map {
    pub width: usize,
    pub height: usize,
    pub tiles: Vec<TileType>,
    pub revealed_tiles: Vec<bool>,
    //新增
    pub visible_tiles: Vec<bool>,
}
```

同样要修改 default 的实现，这里和 revealed_tiles 的类似。同时需要修改 update_view 的系统，将 view 的 visible_tiles 同步到 map 中,代码如下:

```rust
pub fn update_view(mut q_player_view: Query<(&Position, &mut Viewshed)>, mut map: ResMut<Map>) {
    for (pos, mut viewshed) in q_player_view.iter_mut() {
        if viewshed.dirty {
            viewshed.dirty = false;

            viewshed.visible_tiles.clear();
            viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
            viewshed.visible_tiles.retain(|p| {
                p.x >= 0 && p.x < map.width as i32 && p.y >= 0 && p.y < map.height as i32
            });

            for t in map.visible_tiles.iter_mut() {
                *t = false
            }

            for tile in viewshed.visible_tiles.iter() {
                let idx = map.xy_idx(tile.x, tile.y);

                map.revealed_tiles[idx] = true;

                map.visible_tiles[idx] = true;
            }
        }
    }
}
```

最后在 render 系统 中区分 renderable 的渲染颜色。代码如下:

```rust
pub fn render(
    q_position_and_renderable: Query<(&Position, &Renderable)>,
    mut q_render_terminal: Query<&mut Terminal>,
    map: Res<Map>,
) {
    let mut term = match q_render_terminal.get_single_mut() {
        Ok(term) => term,
        Err(_) => return,
    };
    term.clear();

    for x in 0..map.width {
        for y in 0..map.height {
            let idx = map.xy_idx(x as i32, y as i32);

            if map.revealed_tiles[idx] {
                let mut tile: FormattedTile = map.tiles[idx].get_renderable().into();

                if !map.visible_tiles[idx] {
                    tile = tile.fg(Color::GRAY);
                }

                term.put_char([x, y], tile);
            }
        }
    }

    q_position_and_renderable
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
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路
