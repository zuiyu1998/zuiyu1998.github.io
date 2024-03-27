+++
title = "roguelike_chapter3 更有意思的地图"
date = 2024-01-24

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 去除无用的代码

在现在的项目里，还保留着一个 left_movement 系统和相关的定义，但是现在已经不需要它了。可以在项目中将相关的代码删除。

# 添加多个房间

上期添加了 new_map 函数来新建一个 map。这期新建一个更复杂的地图。
在 src/map.rs 中新增一个 Rect，并为它添加基本的方法,代码如下：

```rust
pub struct Rect {
    pub x1: i32,
    pub x2: i32,
    pub y1: i32,
    pub y2: i32,
}

impl Rect {
    pub fn new(x: i32, y: i32, w: i32, h: i32) -> Rect {
        Rect {
            x1: x,
            y1: y,
            x2: x + w,
            y2: y + h,
        }
    }

    ///在和其他rect重叠的情况下返回true
    pub fn intersect(&self, other: &Rect) -> bool {
        self.x1 <= other.x2 && self.x2 >= other.x1 && self.y1 <= other.y2 && self.y2 >= other.y1
    }

    pub fn center(&self) -> (i32, i32) {
        ((self.x1 + self.x2) / 2, (self.y1 + self.y2) / 2)
    }
}
```

新增一个方法，将 rect 的地图数据生成在地图上，代码如下：

```rust
fn apply_room_to_map(room: &Rect, map: &mut Map) {
    for y in room.y1 + 1..=room.y2 {
        for x in room.x1 + 1..=room.x2 {
            let idx = map.xy_idx(x, y);
            map.tiles[idx] = TileType::Floor;
        }
    }
}
```

新增一个 new_map_rooms_and_corridors 函数生成一个复杂的地图,代码如下:

```rust
pub fn new_map_rooms_and_corridors() -> Map {
    let mut map = Map::default();

    let room1 = Rect::new(20, 15, 10, 15);
    let room2 = Rect::new(35, 15, 10, 15);

    apply_room_to_map(&room1, &mut map);
    apply_room_to_map(&room2, &mut map);

    map
}
```

这里使用了 Map 的 defalult 方法，但是现在这是默认实现的，不满足需要。手动为 Map 实现 default trait，代码如下:

```rust
impl Default for Map {
    fn default() -> Self {
        let map = Map {
            tiles: vec![TileType::Wall; 80 * 50],
            width: 80,
            heigth: 50,
        };

        map
    }
}
```

更改 src/logic 下生成地图的函数，将 new_map 改为 new_map_rooms_and_corridors。
运行代码，右键左上角的按钮，点击 playing，应该可以看到两个不互通的房间。
按住 wasd，可以发现，玩家还可以穿墙。修改 user_input 函数，在移动之前，判断下一个 tile 是否可以移动。代码如下：

```rust
impl Position {
    pub fn movement(&mut self, delta_x: i32, delta_y: i32, map: &Map) {
        let next_x = 79.min(0.max(self.x + delta_x));
        let next_y = 79.min(0.max(self.y + delta_y));

        let idx = map.xy_idx(next_x, next_y);

        if map.tiles[idx] == TileType::Floor {
            self.x = next_x;
            self.y = next_y;
        }
    }
}
```

user_input 更改之后的结果如下:

```rust
pub fn user_input(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut q_player: Query<&mut Position, With<Player>>,
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

    for mut position in q_player.iter_mut() {
        position.movement(x, y, &map);
    }
}

```

现在玩家在屏幕上的移动便正常了。

# 添加通道

现在两个房间还无法连通，新建两个函数用来联通房间。代码如下：

```rust
fn apply_horizontal_tunnel(map: &mut Map, x1: i32, x2: i32, y: i32) {
    for x in x1.min(x2)..=x1.max(x2) {
        let idx = map.xy_idx(x, y);
        if idx > 0 && idx < map.width * map.heigth {
            map.tiles[idx] = TileType::Floor;
        }
    }
}

fn apply_vertical_tunnel(map: &mut Map, y1: i32, y2: i32, x: i32) {
    for y in y1.min(y2)..=y1.max(y2) {
        let idx = map.xy_idx(x, y);
        if idx > 0 && idx < map.width * map.heigth {
            map.tiles[idx] = TileType::Floor;
        }
    }
}
```

# 添加一个简易地牢

修改 new_map_rooms_and_corridors 的实现，代码如下:

```rust
pub fn new_map_rooms_and_corridors() -> Map {
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
            apply_room_to_map(&new_room, &mut map);

            if !rooms.is_empty() {
                let (new_x, new_y) = new_room.center();
                let (prev_x, prev_y) = rooms[rooms.len() - 1].center();
                if rng.range(0, 2) == 1 {
                    apply_horizontal_tunnel(&mut map, prev_x, new_x, prev_y);
                    apply_vertical_tunnel(&mut map, prev_y, new_y, new_x);
                } else {
                    apply_vertical_tunnel(&mut map, prev_y, new_y, prev_x);
                    apply_horizontal_tunnel(&mut map, prev_x, new_x, new_y);
                }
            }

            rooms.push(new_room);
        }
    }

    map
}
```

创建一个地图之后，原来初始化的玩家位置可能 floor,导致不能移动，因此玩家需要在地图之后创立。new_map_rooms_and_corridors 现在需要返回二个值，一个是地图，一个是所有的房间。房间用于玩家位置的创建。代码如下：

```rust
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
            apply_room_to_map(&new_room, &mut map);

            if !rooms.is_empty() {
                let (new_x, new_y) = new_room.center();
                let (prev_x, prev_y) = rooms[rooms.len() - 1].center();
                if rng.range(0, 2) == 1 {
                    apply_horizontal_tunnel(&mut map, prev_x, new_x, prev_y);
                    apply_vertical_tunnel(&mut map, prev_y, new_y, new_x);
                } else {
                    apply_vertical_tunnel(&mut map, prev_y, new_y, prev_x);
                    apply_horizontal_tunnel(&mut map, prev_x, new_x, new_y);
                }
            }

            rooms.push(new_room);
        }
    }

    (map, rooms)
}
```

在 src/logic.rs 中，去除 spawn_character 函数，并更改 setup_game 函数，代码如下:

```rust
pub fn setup_game(mut commands: Commands, mut map: ResMut<Map>) {
    let terminal = Terminal::new([80, 50]).with_border(Border::single_line());

    commands.spawn((
        // Spawn the terminal bundle from our terminal
        TerminalBundle::from(terminal),
        Name::new("Terminal"),
    ));

    let (map_instance, rooms) = new_map_rooms_and_corridors();

    *map = map_instance;

    //获取第一个房间的中心位置
    let first_room_centerr = rooms[0].center();

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
    ));
}

```

# 清除无用代码

现在 new_map 函数已经没有用到，在项目文件删除它。

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
