---
Status: 🌲
tags:
  - bevy
  - rust
Links:
  - "[Bevy MOC](Bevy%20MOC.md)"
categories:
  - 游戏开发
  - 编程语言
Series:
  - Bevy制作拼图游戏
title: Bevy制作拼图游戏 Day 2
date: 2024-10-30T10:48:32
Author:
  - ZoOL
share: true
Slug: create a jigsaw puzzle game by bevy Day 2
---

# Bevy初始化
本文撰写时，Bevy正处于0.15发布周期，所以下面的代码以0.15版本为准
初始化项目
```toml
bevy = { version = "0.15.0-rc.2", features = ["wayland"] }  
jigsaw_puzzle_generator = { path = "jigsaw_puzzle_generator" }  
rand = "0.8.5"
```
加载Plugin模块
```rust
pub struct PuzzlePlugin;  
  
impl Plugin for PuzzlePlugin {  
    fn build(&self, app: &mut App) {  
        app.add_plugins(DefaultPlugins.set(WindowPlugin {  
            primary_window: Some(Window {  
                title: "Jigsaw Puzzle Game".to_string(),  
                canvas: Some("#bevy".to_string()),  
                fit_canvas_to_parent: true,  
                prevent_default_event_handling: true,  
                ..Default::default()  
            }),  
            ..default()  
        }));  

		app.add_plugins((ui::plugin, generator::plugin));
    }  
}

fn main() {  
    App::new().add_plugins(DefaultPlugins).run();  
}

/// ui plugin

pub(super) fn plugin(app: &mut App) {  
    app.add_systems(Startup, setup);  
}

/// 生成2D相机 
fn setup(mut commands: Commands) {  
    commands.spawn(Camera2d);  
}

```

# Schedule调度
Bevy 是ECS(Entity-Component-System)架构。
大部分的业务逻辑都是在System里实现
```rust
app.add_systems(Startup, setup_generator);  
```
上面的代码设置Bevy进入`Startup`调度时，执行`setup` System.
Bevy的主要Schedule
```rust
impl Default for MainScheduleOrder {  
    fn default() -> Self {  
        Self {  
            labels: vec![  
                First.intern(),  
                PreUpdate.intern(),  
                RunFixedMainLoop.intern(),  
                Update.intern(),  
                SpawnScene.intern(),  
                PostUpdate.intern(),  
                Last.intern(),  
            ],  
            startup_labels: vec![PreStartup.intern(), Startup.intern(), PostStartup.intern()],  
        }  
    }  
}
```
可以看到Bevy启动时按顺序调用`PreStartup`、`Startup`、`PostStartup`三个Schedule一次， 然后每个Tick按顺序调用 `First`、`PreUpdate`、 `RunFixedMainLoop`、`Update`、`SpawnScene`、`PostUpdate`、`Last`.   
我们的业务逻辑一般写在`Update`里。

# 加载图片

```rust
fn setup_generator(mut commands: Commands, asset_server: Res<AssetServer>) {  
    let image_path = "raw.jpg";  
    let generator = JigsawGenerator::from_path(image_path, 9, 6).expect("Failed to load image");  
  
    // load image from dynamic image  
    let image = asset_server.add(Image::from_dynamic(  
        generator.origin_image().clone(),  
        true,  
        RenderAssetUsages::RENDER_WORLD,  
    ));  
  
    commands.spawn((Sprite::from_image(image), BoardBackgroundImage));  
    commands.insert_resource(JigsawPuzzleGenerator(generator));  
}  
  
#[derive(Debug, Resource, Deref, DerefMut)]  
pub struct JigsawPuzzleGenerator(pub JigsawGenerator);

/// 背景图片标记
#[derive(Component)]  
pub struct BoardImageMarker;  
```

程序启动后加载图片并将我们的拼图生成器通过`Commands`存入`Resource`(`Resource`是Bevy标记存放全局唯一数据的地方).
将原始图片载入精灵图(Sprite)并放入世界,显示结果如下：
![Pasted image 20241030141014.png](/posts/attachments/Pasted%20image%2020241030141014.png)
可以看到由于窗口比图片小，所以只显示了一部分图片。
因此我们调整一下相机， 看全图片。
# 调整相机投影
```rust
app.add_systems(Update, adjust_camera_on_added_sprite);

fn adjust_camera_on_added_sprite(  
    sprite: Single<Entity, Added<BoardBackgroundImage>>,  
    mut camera_2d: Single<&mut OrthographicProjection, With<Camera2d>>,  
    window: Single<&Window>,  
    generator: Res<JigsawPuzzleGenerator>,  
    mut commands: Commands,  
) {  
    let window_width = window.resolution.width();  
    let image_width = generator.origin_image().width() as f32;  
    let scale = image_width / window_width;  
    let target_scale = scale / 0.8;  
    camera_2d.scale = target_scale;  
    commands.entity(*sprite).insert(Visibility::Hidden);  
}
```
这个`System`虽然挂在`Update`下面, 但只会在图片挂载的时候运行一次。原因是这条Query
`sprite: Single<Entity, Added<BoardBackgroundImage>>`
当`BoardBackgroundImage`这个component刚刚添加到world时, 此query才会成功查询到一条数据。此时根据图片和窗口的宽度，换算出一个缩放比例应用到相机上。
在Bevy里2D相机是正交投影，所以用single query查询出`OrthographicProjection`,并修改投影的scale值。此时效果如下：
![Pasted image 20241030165245.png](/posts/attachments/Pasted%20image%2020241030165245.png)
最后在图片的entity下插入`Visibility::Hidden`这个Component，将图片隐藏。准备进行后续的图片分割。
# 生成拼图块
```rust
app.add_systems(  
    PostUpdate,  
    spawn_piece.run_if(resource_changed::<JigsawPuzzleGenerator>),  
)
```
这里是一种新的system运行控制方法，它的条件是当`JigsawPuzzleGenerator`这个Resource的值变动时，才运行`spawn_piece`
按照底图的方式生成块的Sprite
```
fn spawn_piece(  
    mut commands: Commands,  
    generator: Res<JigsawPuzzleGenerator>,  
    asset_server: Res<AssetServer>,  
) {  
    if let Ok(template) = generator.generate() {  
        for piece in template.pieces.iter() {  
            let image = asset_server.add(Image::from_dynamic(  
                template.crop(piece),  
                true,  
                RenderAssetUsages::RENDER_WORLD,  
            ));  
  
            commands.spawn(Sprite::from_image(image));  
        }  
    };  
}
```
运行后，卡了很久才出现窗口
![400](/posts/attachments/Pasted%20image%2020241030190809.png)
原因是切一张图要耗时0.2～0.3秒， 9x6=54张图按队列生成就要卡很久，我们将切图这一步放到异步任务里去做。
## 异步生成拼图图片
```rust
app.add_systems(  
    Update,  
    (  
        spawn_piece.run_if(resource_changed::<JigsawPuzzleGenerator>),  
        handle_tasks,  
    ),  
);

fn spawn_piece(mut commands: Commands, generator: Res<JigsawPuzzleGenerator>) {  
    if let Ok(template) = generator.generate(false) {  
        let thread_pool = AsyncComputeTaskPool::get();  
        for piece in template.pieces.iter() {  
            let template_clone = template.clone();  
            let piece_clone = piece.clone();  
            let entity = commands.spawn(JigsawTile { index: piece.index }).id();  
  
            let task = thread_pool.spawn(async move {  
                let cropped_image = template_clone.crop(&piece_clone);  
                let mut command_queue = CommandQueue::default();  
  
                command_queue.push(move |world: &mut World| {  
                    let asset_server = world.resource::<AssetServer>();  
                    let image = asset_server.add(Image::from_dynamic(  
                        cropped_image,  
                        true,  
                        RenderAssetUsages::RENDER_WORLD,  
                    ));  
                    world  
                        .entity_mut(entity)  
                        .insert(Sprite::from_image(image))  
                        .remove::<CropTask>();  
                });  
  
                command_queue  
            });  
  
            commands.entity(entity).insert(CropTask(task));  
        }  
    };  
}  
  
fn handle_tasks(mut commands: Commands, mut crop_tasks: Query<&mut CropTask>) {  
    for mut task in &mut crop_tasks {  
        if let Some(mut commands_queue) = block_on(future::poll_once(&mut task.0)) {  
            commands.append(&mut commands_queue);  
        }  
    }  
}
```
上面的代码使用了Bevy的异步线程池, 将切图->创建Sprite等步骤放到`CommandQueue`里， 当handle_tasks的里获取到返回值后，将queue重放执行。
## 移动拼图
先解释以下Bevy的坐标系
* 2D
  X轴从左往右(右为正方向)
  Y轴从下到上(上为正方向)
  Z轴从从远到近(朝你的眼睛/屏幕方向是正方向)
  默认原点(x=0.0, y=0.0)是窗口的中心点
* 3D
  记住一点，Bevy使用右手坐标系， Y轴朝上，Z轴朝屏幕
  ![Pasted image 20241031101658.png](/posts/attachments/Pasted%20image%2020241031101658.png)```

首先我们修改一下Sprite的生成方式，将锚点改为左上角
``` rust
let sprite = Sprite {  
    image,  
    anchor: Anchor::TopLeft,  
    ..default()  
};
```
随机生成拼图块的左上角的坐标，保持在窗口里
``` rust
let resolution = &window.resolution;  
let calc_position = random_position(&piece, resolution.size(), camera.scale);  
let entity = commands  
    .spawn((  
        JigsawTile { index: piece.index },  
        Transform::from_xyz(calc_position.x, calc_position.y, 1.0),  
    ))  
    .id();

fn random_position(piece: &JigsawPiece, window_size: Vec2, scale: f32) -> Vec2 {  
    let window_width = window_size.x / 2.0 * scale;  
    let window_height = window_size.y / 2.0 * scale;  
    let min_x = -window_width + piece.crop_width as f32;  
    let min_y = -window_height + piece.crop_height as f32;  
    let max_x = window_width - piece.crop_width as f32;  
    let max_y = window_height - piece.crop_height as f32;  
  
    let mut rng = rand::thread_rng();  
    let x = rng.gen_range(min_x..max_x);  
    let y = rng.gen_range(min_y..max_y);  
    Vec2::new(x, y)  
}
```
注意，我们在之前缩放了窗口， 所以计算坐标时需要乘上缩放比例。随机效果如下:
![Pasted image 20241031182030.png](/posts/attachments/Pasted%20image%2020241031182030.png)