+++
title = "roguelike_chapter5 添加敌人"
date = 2024-01-24

[taxonomies]
tags = ["roguelike", "bevy"]
+++

[bracketproductions](https://bfnightly.bracketproductions.com)的 bevy 实现。
代码仓库: [RoguelikeTutorial](https://github.com/zuiyu1998/RoguelikeTutorial.git)

<!-- more -->

# 在每个房间内添加一个敌人

使用组合而不是继承。
敌人和玩家有一些共同的组件，比如 Position 和 Renderable。
先使用这些共同的组件生成一个可见的敌人。
在 src/logic/setup_game 的系统内，利用生成的 rooms 为每个房间添加一个敌人,代码如下:

```rust
///新加的代码
rooms.iter().skip(1).for_each(|room| {
        let center = room.center();

        commands.spawn_empty().insert((
            Position {
                x: center.0,
                y: center.1,
            },
            Renderable {
                glyph: 'g',
                fg: Color::RED,
                bg: Color::BLACK,
            },
            Viewshed {
                visible_tiles: vec![],
                range: 8,
                dirty: false,
            },
        ));
    });
```

同时需要修改 src/map.rs 中 update_view 系统的参数,因为现在的是实现是更新了所有具有 Position 和 Viewshed 的实体。上期只有 player 的实体存在，所以是可以实现的。但是现在引入多个敌人实体，它们同样具有 Position 和 Viewshed,所以需要一个约束。代码如下:

```rust
pub fn update_view(
    mut q_player_view: Query<(&Position, &mut Viewshed), With<Player>>,
    mut map: ResMut<Map>,
)
```

With<Player>就是添加的约束。
运行代码，右键左上角的按钮，点击 playing，右键左上角的按钮，会出现下图界面。
![运行界面](./images/first.png)

可以看到，每个敌人的位置都被透视了。在 src/render.rs 的 render 系统中修复这个问题。这个问题的原因是在渲染 renderable 数据的时候没有判断可见性。代码如下:

```rust
q_position_and_renderable
        .iter()
        .for_each(|(position, renderable)| {
            let idx = map.xy_idx(position.x, position.y);

            //添加这个可见性判定
            if map.visible_tiles[idx] {
                let tile: FormattedTile = renderable.clone().into();

                term.put_char(position.clone(), tile);
            }
        });
```

# 添加多个类型的敌人

不同类型的敌人应该有很多不同，但是现在先实现一个简易的版本，就是显示的不同，利用随机数生成器生成一个数据，然后根据这个数据生成不同的字符，利用这个字符构建 Renderable。修改 src/logic.rs 中 setup_game 系统，代码如下:

```rust
    let mut rng = RandomNumberGenerator::new();

    rooms.iter().skip(1).for_each(|room| {
        let center = room.center();

        let roll = rng.roll_dice(1, 2);

        let glyph;

        match roll {
            1 => glyph = 'g',
            _ => glyph = 'o',
        }

        commands.spawn_empty().insert((
            Position {
                x: center.0,
                y: center.1,
            },
            Renderable {
                glyph,
                fg: Color::RED,
                bg: Color::BLACK,
            },
            Viewshed {
                visible_tiles: vec![],
                range: 8,
                dirty: false,
            },
        ));
    });
```

运行代码，右键左上角的按钮，点击 playing，右键左上角的按钮，会出现下图界面。

![运行界面](./images/first.png)

# 制作敌人的 ai 系统

首先定义一个 Monster 组件标记敌人。在 src/logic 下新增如下代码:

```rust
#[derive(Component, Debug)]
pub struct Monster {}
```

在 src/logic.rs 中新增一个 monster_ai 系统。代码如下:

```rust
pub fn monster_ai(mut q_monster: Query<(&mut Position, &mut Viewshed), With<Monster>>) {
    for _monster in q_monster.iter_mut() {
        //占位
        info!("Monster considers their own existence")
    }
}

```

同时修改 src/logic.rs 中 setup_game 系统，为敌人实体添加 Monster 组件，代码如下:

```rust
pub fn monster_ai(mut q_monster: Query<(&mut Position, &mut Viewshed, &Monster)>) {
    for _monster in q_monster.iter_mut() {
        //占位
        info!("Monster considers their own existence")
    }
}
```

最后修改 src/map.rs 中的 updat_view 系统，原先这里实现的玩家实体 Viewshed 的更新，现在敌人的 Viewshed 同样要更新。代码更改如下：

```rust
pub fn update_view(
    mut q_view: Query<(&Position, &mut Viewshed)>,
    mut map: ResMut<Map>,
    q_player: Query<Entity, With<Player>>,
) {
    for (pos, mut viewshed) in q_view.iter_mut() {
        if viewshed.dirty {
            viewshed.dirty = false;

            viewshed.visible_tiles.clear();
            viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
            viewshed.visible_tiles.retain(|p| {
                p.x >= 0 && p.x < map.width as i32 && p.y >= 0 && p.y < map.height as i32
            });
        }
    }

    for player_entity in q_player.iter() {
        if let Ok((_, viewshed)) = q_view.get(player_entity) {
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

# 让玩家和敌人拥有自己名字

bevy 提供了一个 Name 的组件用来标识一个实体的名称，使用这个为玩家和敌人提供一个名称。修改 src/logic.rs 下的 setup_game 系统。代码如下:

```rust
 rooms.iter().skip(1).enumerate().for_each(|(i, room)| {
        let center = room.center();

        let roll = rng.roll_dice(1, 2);

        let glyph;
        let name;

        match roll {
            1 => {
                glyph = 'g';
                name = "Goblin".to_string();
            }
            _ => {
                glyph = 'o';
                name = "Orc".to_string();
            }
        }

        commands.spawn_empty().insert((
            Position {
                x: center.0,
                y: center.1,
            },
            Renderable {
                glyph,
                fg: Color::RED,
                bg: Color::BLACK,
            },
            Viewshed {
                visible_tiles: vec![],
                range: 8,
                dirty: false,
            },
            Monster {},
            Name::new(format!("{} #{}", name, i)),
        ));
    });
```

# 让敌人发现玩家

现在让我们一步一步实现敌人的 ai，首先让敌人发现玩家。代码如下:

```rust
pub fn monster_ai(
    mut set: ParamSet<(
        Query<(&mut Position, &mut Viewshed, &Monster, &Name)>,
        Query<&Position, With<Player>>,
    )>,
) {
    let player = set.p1();
    let player_pos = player.single().clone();

    for (_pos, viewshed, _, name) in set.p0().iter_mut() {
        //占位

        if viewshed
            .visible_tiles
            .contains(&Point::new(player_pos.x, player_pos.y))
        {
            info!("{} shouts insults", name);
        }
    }
}
```

正常情况下，运行代码，应该可以看到日志中输出了 xx shouts insults,但是在看到敌人时没有出现，这是因为敌人实体的 Viewshed 组件的 dirty 默认为 false。修改这个字段，整个系统就可以正常工作了。

> ParamSet 是一个特殊的参数，它可以同时拥有同个组件数据的多个引用。

# 致谢

- [bevy](https://github.com/bevyengine/bevy),游戏引擎
- [bevy_game_template](https://github.com/NiklasEi/bevy_game_template.git),游戏模板
- [bevy_ascii_terminal](https://github.com/sarkahn/bevy_ascii_terminal),字符显示
- [bevy_editor_pls](https://github.com/jakobhellermann/bevy_editor_pls),可视化编辑器
- [bracket-random](https://github.com/amethyst/bracket-lib)，随机数生成器
- [bracket-pathfinding](https://github.com/amethyst/bracket-lib) 寻路

```

```
