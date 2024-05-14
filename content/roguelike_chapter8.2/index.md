+++
title = "roguelike_chapter8.2 道具和背包"
date = 2024-05-14

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
                app.add_systems(OnExit(AppState::Loading), configure_visuals_system);


        #[cfg(not(feature = "dev"))]
        {
            use bevy_egui::EguiPlugin;
            app.add_plugins((EguiPlugin));
        }
    }
}

fn set_fonts(context: &mut egui::Context) {
    let mut fonts = FontDefinitions::default();

    fonts.font_data.insert(
        "VonwaonBitmap".to_owned(),
        FontData::from_static(include_bytes!(
            "../../assets/fonts/VonwaonBitmap-16pxLite.ttf"
        )),
    );

    fonts
        .families
        .get_mut(&egui::FontFamily::Proportional)
        .unwrap()
        .insert(0, "VonwaonBitmap".to_owned());

    context.set_fonts(fonts);
}

//同步bevy的image到egui
fn set_textures(contexts: &mut EguiContexts, texture_assets: &TextureAssets) {
    contexts.add_image(texture_assets.i.clone());
}

fn configure_visuals_system(mut contexts: EguiContexts, texture_assets: Res<TextureAssets>) {
    contexts.ctx_mut().set_visuals(egui::Visuals {
        window_rounding: 0.0.into(),
        window_shadow: egui::epaint::Shadow::NONE,
        ..Default::default()
    });

    set_fonts(contexts.ctx_mut());

    set_textures(&mut contexts, &texture_assets);
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

新增一个 trait UiSystem,该 trait 生成一些胶水代码结合 bevy 和 egui。代码如下:

```rust
pub trait UiSystem: SystemParam + 'static {
    type UiState: UiContainer<Self>;

    fn extra_ui_state(item: &<Self as SystemParam>::Item<'_, '_>) -> Self::UiState;

    fn ui(
        ui_state: Self::UiState,
        ui_context: EguiUiContext,
        bevy_context: BevyBuildContext<<Self as SystemParam>::Item<'_, '_>>,
    ) {
        ui_state.container(ui_context, bevy_context);
    }

    fn show_ui(world: &mut World) {
        let mut state = SystemState::<EguiContexts>::new(world);
        let mut contexts = state.get_mut(world);
        let context = contexts.ctx_mut().clone();

        let textures = world.get_resource::<EguiUserTextures>().unwrap().clone();

        let mut state = SystemState::<Self>::new(world);
        let item = state.get_mut(world);

        let ui_state = Self::extra_ui_state(&item);

        let ui_context: EguiUiContext = EguiUiContext { context, textures };

        let build_context = BevyBuildContext { item };

        Self::ui(ui_state, ui_context, build_context);
    }
}

```

UiState，为 egui 的容器数据，它必须实现 UiContainer trait。UiContainer 为 ui 布局的实现。extra_ui_state 函数用于从 bevy ecs 中获取数据。show_ui 函数为胶水代码。ui 函数中使用 UiContainer 的具体实现。这里有两个重要的定义。
EguiUiContext 为 egui 所需要的上下文。代码如下:

```rust
pub struct EguiUiContext {
    context: egui::Context,
    textures: EguiUserTextures,
}

```

context，为 egui 的上下文，textures 为 egui 从 bevy 中获取的图像索引。
BevyBuildContext 为从 bevy ecs 中获取的上下文，代码如下:

```rust
pub struct BevyBuildContext<T> {
    pub item: T,
}
```

这两个上下文都是在编写 ui 时需要的。具体的实现为 UiContainer。代码如下:

```rust
pub trait UiContainer<T>
where
    T: SystemParam + 'static,
{
    fn container(
        &self,
        ui_context: EguiUiContext,
        bevy_context: BevyBuildContext<<T as SystemParam>::Item<'_, '_>>,
    );
}
```

这里的泛型 T 就是从 bevy ecs 提取的系统参数。实现使用的则是 bevy_context。

新增一个 UiWidght 定义小部件，以便复用 ui 代码，定义如下:

```rust

pub trait UiWidght<T> {
    fn widght<'a>(&self, context: EguiWidghtBuildContext<'a, T>, ui: &'a mut Ui);
}
```

ui 为 egui 布局。context 为 egui context 的引用和实际使用 bevy context。定义如下；

```rust
pub struct EguiWidghtBuildContext<'a, T> {
    pub item: T,
    pub ui_context: &'a EguiUiContext,
}

```

新增一个 trait BuildUiWidght，用来描述 Ui Container 中使用的 bevy context 到小部件所使用的 bevy context 的转化关系。定义如下:

```rust
pub trait BuildUiWidght<T, P>
where
    P: SystemParam + 'static,
    Self: Sized,
{
    fn build_widght(
        &self,
        ui_context: &EguiUiContext,
        bevy_context: &mut BevyBuildContext<<P as SystemParam>::Item<'_, '_>>,
        ui: &mut Ui,
    );
}
```

# 完善道具系统

新增 ItemInBackpacks 资源。该资源用来保存游戏中所有在背包中的数据。代码如下:

```rust
#[derive(Debug, Resource, Deref, DerefMut, Default)]
pub struct ItemInBackpacks(HashMap<Entity, ItemInBackpack>);

#[derive(Debug, Resource, Deref, DerefMut, Default)]
pub struct ItemInBackpack(HashMap<ItemType, ItemData>);

#[derive(Debug, Clone, Default)]
pub struct ItemData {
    pub count: i32,
    pub data: Vec<Entity>,
}


```

ItemData 保存了某一类的数量和道具实体。ItemInBackpack 则是道具所有者的集合。
新增 ItemAddedEvent/ItemRemoveEvent/ItemApplyEvent 三个事件。代码如下:

```rust
#[derive(Debug, Event)]
pub struct ItemAddedEvent {
    owner: Entity,
    item: Entity,
}

#[derive(Debug, Event)]
pub struct ItemRemoveEvent {
    item: Entity,
    owner: Entity,
}

#[derive(Debug, Event)]
pub struct ItemApplyEvent {
    pub item: Entity,
    pub item_type: ItemType,
    pub owner: Entity,
}
```

ItemAddedEvent 表示为物品新增事件，ItemRemoveEvent 表示为物品剔除事件，ItemApplyEvent 表示物品使用事件。

添加一个系统 handle_item_update_event 处理 ItemAddedEvent 和 ItemRemoveEvent。代码如下:

```rust
pub fn handle_item_update_event(
    mut item_remove_er: EventReader<ItemRemoveEvent>,
    mut item_added_er: EventReader<ItemAddedEvent>,
    mut item_in_backs: ResMut<ItemInBackpacks>,
    mut q_item: Query<&ItemType>,
) {
    for event in item_remove_er.read() {
        let mut item_in_back = match item_in_backs.remove(&event.owner) {
            None => {
                continue;
            }
            Some(item_in_back) => item_in_back,
        };

        let mut need_insert = true;

        if let Ok(item_type) = q_item.get_mut(event.item) {
            let mut item_data = item_in_back.remove(item_type).unwrap_or_default();
            item_data.count -= 1;
            item_data.data.pop();

            if item_data.count > 0 {
                item_in_back.insert(*item_type, item_data);
            }

            if item_in_back.is_empty() {
                need_insert = false;
            }
        }

        if need_insert {
            item_in_backs.insert(event.owner, item_in_back);
        }
    }

    for event in item_added_er.read() {
        let mut item_in_back = item_in_backs.remove(&event.owner).unwrap_or_default();

        if let Ok(item_type) = q_item.get_mut(event.item) {
            let mut item_data = item_in_back.remove(item_type).unwrap_or_default();
            item_data.count += 1;
            item_data.data.push(event.item);

            item_in_back.insert(*item_type, item_data);
        }

        item_in_backs.insert(event.owner, item_in_back);
    }
}
```

handle_item_update_event 系统的主要功能是更新 ItemInBackpacks 资源。修改 item_collect 系统，将 ItemEvent 改为 ItemAddedEvent，代码如下:

```rust
pub fn item_collect(
    mut commands: Commands,
    q_wants_to_pickup_item: Query<(&Parent, Entity, &WantsToPickupItem)>,
    q_items: Query<Entity, (With<Item>, Without<InBackpack>)>,
    mut item_ew: EventWriter<ItemAddedEvent>,
) {
    for (parent, wants_to_pickup_item_entity, wants_to_pickup_item) in q_wants_to_pickup_item.iter()
    {
        if let Ok(_) = q_items.get(wants_to_pickup_item.item) {
            commands
                .entity(wants_to_pickup_item.item)
                .insert(InBackpack {
                    owner: parent.get(),
                })
                .remove::<SpriteSheetBundle>()
                .remove::<Position>()
                .set_parent(wants_to_pickup_item.collected_by);

            item_ew.send(ItemAddedEvent {
                owner: parent.get(),
                item: wants_to_pickup_item.item,
            });

            commands
                .entity(wants_to_pickup_item_entity)
                .despawn_recursive();
        }
    }
}

```

将 src/loading.rsz 中 TextureAssets 移入 src/core/ui.rs 中。
为 ItemType 新增一个工具函数用来获取道具图片。代码如下:

```rust
impl ItemType {
    pub fn get_image_handle(&self, texture_assets: &TextureAssets) -> Handle<Image> {
        match self {
            ItemType::HealthPotion => texture_assets.i.clone(),
        }
    }
}
```

新增一个 trait ItemComponent 用于抽象道具。代码如下:

```rust
pub trait ItemComponent: Component {
    type Event: Event;

    fn from_item_apply_event(event: &ItemApplyEvent) -> Option<Self::Event>;

    fn once() -> bool {
        true
    }
}
```

once 函数表示是否为一次性道具。from_item_apply_event 将物品使用事件转化为道具系统自身可以使用的事件。

在 src/item/modd.rs 中新增 ItemTypePlugin<Item>，用于复用道具组件的逻辑，代码如下:

```rust
impl<Item> Plugin for ItemTypePlugin<Item>
where
    Item: ItemComponent,
{
    fn build(&self, app: &mut App) {
        app.add_event::<Item::Event>();

        app.add_systems(
            Update,
            (handle_item_apply_event::<Item>).run_if(in_state(AppState::InGame)),
        );
    }
}
```

ItemTypePlugin 会注册道具系统使用的事件和一个默认的 handle_item_apply_event 系统。handle_item_apply_event 系统的实现如下:

```rust
pub fn handle_item_apply_event<Item>(
    mut item_apply_er: EventReader<ItemApplyEvent>,
    mut item_ew: EventWriter<Item::Event>,
    mut item_reomve_ew: EventWriter<ItemRemoveEvent>,
) where
    Item: ItemComponent,
{
    let mut events = vec![];

    for e in item_apply_er.read() {
        if let Some(event) = Item::from_item_apply_event(e) {
            events.push((
                ItemRemoveEvent {
                    owner: e.owner,
                    item: e.item,
                },
                event,
            ));
        }
    }

    for (remove_event, item_event) in events.into_iter() {
        item_ew.send(item_event);

        if Item::once() {
            item_reomve_ew.send(remove_event);
        }
    }
}

```

该系统将 ItemApplyEvent 转换为道具系统自己使用的事件。

在 src/item/potion.rs 中，将 Potion 的定义从 rc/item/mod.ItemComponent,代码如下:

```rust
impl ItemComponent for Potion {
    type Event = PotionEvent;

    fn from_item_apply_event(event: &ItemApplyEvent) -> Option<Self::Event> {
        if matches!(event.item_type, ItemType::HealthPotion) {
            Some(PotionEvent { item: event.item })
        } else {
            None
        }
    }
}

//生命药水
#[derive(Component, Debug)]
pub struct Potion {
    pub heal_amount: i32,
}

#[derive(Debug, Event)]
pub struct PotionEvent {
    item: Entity,
}
```

新增一个系统 potion_apply 用来处理 PotionEvent。代码如下:

```rust
fn potion_apply(
    mut item_er: EventReader<PotionEvent>,
    q_items: Query<(&Parent, &Potion), With<Item>>,
    mut q_stats: Query<&mut CombatStats>,
    mut commands: Commands,
) {
    for e in item_er.read() {
        if let Ok((parent, potion)) = q_items.get(e.item) {
            if let Ok(mut stats) = q_stats.get_mut(parent.get()) {
                let tmp_hp = stats.hp + potion.heal_amount;

                stats.hp = tmp_hp.min(stats.max_hp);

                commands.entity(e.item).despawn_recursive();
            }
        }
    }
}
```

potion_apply 会提升人物的血量。
新增一个 PotionPlugin，定义如下:

```rust
pub struct PotionPlugin;

impl Plugin for PotionPlugin {
    fn build(&self, app: &mut App) {
        app.add_plugins(ItemTypePlugin::<Potion>::default());

        app.add_systems(Update, (potion_apply,).run_if(in_state(AppState::InGame)));
    }
}
```

将 PotionPlugin 放入 ItemPlugin。

# 背包 ui

在 src/ui/backpack.rs 中新增 ItemUiData 和 ItemUiDataInternal。ItemUiData 这个背包中格子的抽象，因此它有可能为空。ItemUiDataInternal 为格子中数据的抽象。代码如下:

```rust
#[derive(Debug, Clone, Default)]
pub struct ItemUiData(Option<ItemUiDataInternal>);

#[derive(Debug, Clone)]
pub struct ItemUiDataInternal {
    pub item_data: ItemData,
    pub item_image: Handle<Image>,
    pub item_type: ItemType,
    pub owner: Entity,
}
```

item_data 定义在 src/item/mod.rs 中，它保存了同类物品的数量和实际的实体。它的实现如下:

```rust
#[derive(Debug, Clone, Default)]
pub struct ItemData {
    pub count: i32,
    pub data: Vec<Entity>,
}
```

item_image 为道具图片，item_type 道具类型，ower 为所有者。
为 ItemUiData 添加一些工具函数，代码如下:

```rust
impl ItemUiData {
    pub fn new(
        item_data: ItemData,
        item_image: Handle<Image>,
        item_type: ItemType,
        owner: Entity,
    ) -> Self {
        ItemUiData(Some(ItemUiDataInternal {
            item_data,
            item_image,
            item_type,
            owner,
        }))
    }

    pub fn get_item_data(&self) -> Option<&ItemData> {
        self.0
            .as_ref()
            .and_then(|internal| Some(&internal.item_data))
    }

    pub fn get_item_image(&self) -> Option<&Handle<Image>> {
        self.0
            .as_ref()
            .and_then(|internal| Some(&internal.item_image))
    }

    pub fn get_item(&self) -> Option<&ItemUiDataInternal> {
        self.0.as_ref()
    }
}
```

新增 BackPackUiStateItem，这是要从 bevy 中获取的上下文。定义如下:

```rust
pub struct BackPackUiStateItem<'b, 'w> {
    pub item_ew: &'b mut EventWriter<'w, ItemApplyEvent>,
}

```

为 ItemUiData 实现 UiWidght，代码如下:

```rust
impl<'b, 'w> UiWidght<BackPackUiStateItem<'b, 'w>> for ItemUiData {
    fn widght<'a>(
        &self,
        context: EguiWidghtBuildContext<'a, BackPackUiStateItem<'b, 'w>>,
        ui: &'a mut egui::Ui,
    ) {
        ui.set_height(60.0);
        ui.set_width(60.0);

        let frame = egui::Frame::none();

        frame.show(ui, |ui| {
            if self.0.is_some() {
                let item_image_id = context.image_id(self.get_item_image().unwrap()).unwrap();

                let rect = ui.available_rect_before_wrap();

                let mut child_ui = ui.child_ui(
                    rect,
                    egui::Layout::centered_and_justified(egui::Direction::RightToLeft),
                );

                child_ui.image(egui::load::SizedTexture::new(
                    item_image_id,
                    egui::vec2(32., 32.),
                ));

                let rect = ui.available_rect_before_wrap();

                let mut child_ui = ui.child_ui(rect, egui::Layout::bottom_up(egui::Align::Max));

                child_ui.label(format!("{}", self.get_item_data().unwrap().count));
            }
        });

        let size = ui.available_size();

        let (_, res) = ui.allocate_exact_size(size, egui::Sense::click());

        if res.clicked() {
            if let Some(item) = self.get_item() {
                context.item.item_ew.send(ItemApplyEvent {
                    item: *item.item_data.data.last().unwrap(),
                    item_type: item.item_type,
                    owner: item.owner,
                });
            }
        }
    }
}
```

更改 UiWidght 的实现可以实现漂亮的界面。

新增 BackPackUiState，这是整个背包的实现，代码如下:

```rust
pub struct BackPackUiState {
    pub data: Vec<ItemUiData>,
    pub row_count: usize,
}
```

data 为所有格子的数据，row_count 表示每行的格子数。
为 BackPackUiState 实现 UiWidght，代码如下:

```rust
impl<'b, 'w> UiWidght<BackPackUiStateItem<'b, 'w>> for BackPackUiState {
    fn widght<'a>(
        &self,
        context: EguiWidghtBuildContext<'a, BackPackUiStateItem<'a, 'w>>,
        ui: &'a mut egui::Ui,
    ) {
        let len = self.data.len();
        let step = self.row_count;

        let EguiWidghtBuildContext {
            mut item,
            ui_context,
        } = context;

        for (index, _) in (0..len).into_iter().step_by(step).enumerate() {
            ui.columns(step, |columns: &mut [egui::Ui]| {
                for column_index in 0..step {
                    let data_index = index * 9 + column_index;

                    let ui_state_item = BackPackUiStateItem {
                        item_ew: &mut item.item_ew,
                    };

                    let widght_build_context =
                        EguiWidghtBuildContext::new(ui_state_item, ui_context);

                    self.data[data_index].widght(widght_build_context, &mut columns[column_index]);
                }
            });
        }
    }
}
```

# 玩家状态 ui

在 src/ui/player.rs 中新增 PlayerUIParams，这是从 bevy 中需要获取的全部上下文。代码如下:

```rust
#[derive(SystemParam)]
pub struct PlayerUIParams<'w> {
    q_items: Res<'w, ItemInBackpacks>,
    player_entity: Res<'w, PlayerEntity>,
    texture_assets: Res<'w, TextureAssets>,
    item_ew: EventWriter<'w, ItemApplyEvent>,
}
```

新增 PlayerUiState，这是玩家 ui 状态的抽象，代码如下:

```rust
#[derive(Default)]
pub struct PlayerUiState {
    backpack: BackPackUiState,
}
```

为 PlayerUIParams 实现 UiSystem，代码如下:

```rust
impl<'w: 'static> UiSystem for PlayerUIParams<'w> {
    type UiState = PlayerUiState;

    fn extra_ui_state(item: &<Self as SystemParam>::Item<'_, '_>) -> Self::UiState {
        let player_entity = item.player_entity.0;

        let mut state = PlayerUiState::default();

        if let Some(item_in_back) = item.q_items.get(&player_entity) {
            if item_in_back.len() > state.backpack.data.len() {
                let mut tmp = vec![];
                for (item_type, item_data) in item_in_back.iter() {
                    tmp.push(ItemUiData::new(
                        item_data.clone(),
                        item_type.get_image_handle(&item.texture_assets),
                        *item_type,
                        player_entity,
                    ));
                }

                state.backpack.data = tmp;
            } else {
                for (index, (item_type, item_data)) in item_in_back.iter().enumerate() {
                    state.backpack.data[index] = ItemUiData::new(
                        item_data.clone(),
                        item_type.get_image_handle(&item.texture_assets),
                        *item_type,
                        player_entity,
                    );
                }
            }
        }

        state
    }
}
```

extra_ui_state 函数为提取玩家的背包数据，这个的数据来源于 ItemInBackpacks。
为 PlayerUIParams 实现 UiContainer,代码如下:

```rust
impl<'w: 'static> UiContainer<PlayerUIParams<'w>> for PlayerUiState {
    fn container(
        &self,
        ui_context: crate::core::EguiUiContext,
        mut bevy_context: BevyBuildContext<<PlayerUIParams<'w> as SystemParam>::Item<'_, '_>>,
    ) {
        egui::Window::new("背包").show(ui_context.get(), |ui| {
            self.build_widght(&ui_context, &mut bevy_context, ui);
        });
    }
}
```

这里可以使用 build_widght 函数是因为 BuildUiWidght trait。代码如下:

```rust
impl<'b, 'w> BuildUiWidght<BackPackUiStateItem<'b, 'w>, PlayerUIParams<'w>> for PlayerUiState
where
    'w: 'static,
{
    fn build_widght(
        &self,
        ui_context: &EguiUiContext,
        bevy_context: &mut BevyBuildContext<<PlayerUIParams<'w> as SystemParam>::Item<'_, '_>>,
        ui: &mut egui::Ui,
    ) {
        //todo 简化
        let ui_state_item = BackPackUiStateItem {
            item_ew: &mut bevy_context.item.item_ew,
        };
        let widght_build_context = EguiWidghtBuildContext::new(ui_state_item, ui_context);
        self.backpack.widght(widght_build_context, ui);
    }
}
```

BuildUiWidght 描述 UiSystem 中的上下文怎么变为 UiWidght 所需的上下文。

新增一个 PlayerUIPlugin，并注册到 ItemPlugin，代码如下:

```rust
pub struct PlayerUIPlugin;

impl Plugin for PlayerUIPlugin {
    fn build(&self, app: &mut App) {
        app.add_systems(
            Update,
            PlayerUIParams::show_ui.run_if(in_state(GameState::Tab)),
        );
    }
}
```

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

新增一个 MenuUiParams ,代码如下:

```rust
#[derive(SystemParam)]
pub struct MenuUiParams<'w> {
    pub menu_item_er: EventWriter<'w, MenuItem>,
}
```

为 MenuUiParams 实现 UiSystem，代码如下:

```rust
impl<'w> UiSystem for MenuUiParams<'w>
where
    'w: 'static,
{
    type UiState = MenuUiState;

    fn extra_ui_state(
        _item: &<Self as bevy::ecs::system::SystemParam>::Item<'_, '_>,
    ) -> Self::UiState {
        MenuUiState::default()
    }
}
```

为 MenuUiState 实现 UiContainer,代码如下:

```rust
impl<'w> UiContainer<MenuUiParams<'w>> for MenuUiState
where
    'w: 'static,
{
    fn container(
        &self,
        ui_context: EguiUiContext,
        mut bevy_context: BevyBuildContext<
            <MenuUiParams<'w> as bevy::ecs::system::SystemParam>::Item<'_, '_>,
        >,
    ) {
        egui::Window::new("menu")
            .title_bar(false)
            .anchor(Align2::CENTER_CENTER, [0.0, 0.0])
            .show(ui_context.get(), |ui| {
                for item in self.item_list.iter() {
                    let button = ui.add(egui::Button::new(item.item_type.to_string()));

                    if button.clicked() {
                        bevy_context.item.menu_item_er.send(item.clone());
                    }
                }
            });
    }
}
```

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

# 重构 Hud 页面

新增 HudUiState 用于显示 hud 数据。代码如下:

```rust
pub struct HudUiState {
    logs: Vec<String>,
    hp: i32,
    max_hp: i32,
}

```

新增 HudParams 获取 bevy 上下文,代码如下:

```rust
#[derive(SystemParam)]
pub struct HudParams<'w, 's> {
    q_stats: Query<'w, 's, &'static CombatStats, With<Player>>,
    game_log: Res<'w, GameLog>,
}
```

为 HudParams 实现 UiSystem，代码如下:

```rust
impl<'w, 's> UiSystem for HudParams<'w, 's>
where
    'w: 'static,
    's: 'static,
{
    type UiState = HudUiState;

    fn extra_ui_state(item: &<Self as SystemParam>::Item<'_, '_>) -> Self::UiState {
        let stats = item.q_stats.single();

        let length = item.game_log.entries.len();

        let mut logs = vec![];

        if length <= 4 {
            for i in 0..length {
                logs.push(item.game_log.entries[i].clone());
            }
        } else {
            for i in length - 4..length {
                logs.push(item.game_log.entries[i].clone());
            }
        }

        HudUiState {
            logs,
            hp: stats.hp,
            max_hp: stats.max_hp,
        }
    }
}
```

为 HudUiState 实现 UiContainer，代码如下:

```rust
pub struct HudUiState {
    logs: Vec<String>,
    hp: i32,
    max_hp: i32,
}

impl<'w, 's> UiContainer<HudParams<'w, 's>> for HudUiState
where
    'w: 'static,
    's: 'static,
{
    fn container(
        &self,
        ui_context: EguiUiContext,
        _bevy_context: BevyBuildContext<<HudParams<'w, 's> as SystemParam>::Item<'_, '_>>,
    ) {
        egui::TopBottomPanel::bottom("my_bottom")
            .min_height(100.0)
            .show(ui_context.get(), |ui| {
                ui.columns(2, |columns| {
                    egui::ScrollArea::vertical().show(&mut columns[0], |ui| {
                        for log in self.logs.iter() {
                            ui.label(log);
                        }
                    });

                    egui::Frame::none().show(&mut columns[1], |ui| {
                        ui.horizontal(|ui| {
                            let progress = self.hp as f32 / self.max_hp as f32;

                            ui.label(format!("{}/{}", self.hp, self.max_hp));

                            ui.add(egui::ProgressBar::new(progress));
                        });
                    });
                });
            });
    }
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
