---
Status: 🌲
tags:
  - bevy
  - rust
Links:
  - "[Bevy MOC](Bevy%20MOC.md)"
date: 2024-11-05T15:32:44
categories:
  - 游戏开发
  - 编程语言
Series:
  - Bevy制作拼图游戏
title: Bevy制作拼图游戏 Day 4
Author:
  - ZoOL
share: true
---
# 拖拽拼图
由于我们在上一步已经实现了`move_piece`的功能，所以我们同样只要在拖拽开始和结束时加入`MoveStart`组件
``` rust
fn on_drag_start(  
    trigger: Trigger<Pointer<DragStart>>,  
    mut image: Query<&mut Transform, With<Piece>>,  
    camera: Single<(&Camera, &GlobalTransform), With<Camera2d>>,  
    mut commands: Commands,  
) {  
    if let Ok(mut transform) = image.get_mut(trigger.entity()) {  
        let click_position = trigger.event().pointer_location.position;  
        let (camera, camera_global_transform) = camera.into_inner();  
        let point = camera  
            .viewport_to_world_2d(camera_global_transform, click_position)  
            .unwrap();  
        transform.translation.z = 200.0;  
        commands.entity(trigger.entity()).insert(MoveStart {  
            image_position: *transform,  
            click_position: point,  
        });  
    }  
}  
  
fn on_drag_end(  
    trigger: Trigger<Pointer<DragEnd>>,  
    mut image: Query<&mut Transform, With<MoveStart>>,  
    mut commands: Commands,  
) {  
    if let Ok(mut transform) = image.get_mut(trigger.entity()) {  
        transform.translation.z = 0.0;  
        commands.entity(trigger.entity()).remove::<MoveStart>();  
        commands.trigger_targets(MoveEnd, vec![trigger.entity()]);
    }  
}
```
# ESC 取消选择拼图
由于目前Bevy还在rc阶段，Sprite Picking还有些问题，重叠时可能引起两次点击，所以加了一个键盘判断: 按下ESC时， 将所有的选择的拼图放下。
``` rust
fn cancel_all_move(  
    key: Res<ButtonInput<KeyCode>>,  
    query: Query<Entity, With<MoveStart>>,  
    mut commands: Commands,  
) {  
    if key.just_pressed(KeyCode::Escape) {  
        for entity in query.iter() {  
            commands.entity(entity).remove::<MoveStart>();  
        }  
    }  
}
```
# 正确排序拼图高度
bevy 2D坐标系里，z越高就越靠前. 
我们初始生成拼图时，使用拼图序号作为Z轴高度。此时拼图各自的高度关系是正确的。
当我们点击/拖动时，将Z改为200或者更大的值，保证移动的图块都在最上面。
当我们放下时， 计算一下周围拼图的最大z-index, 这样就可以让放下的拼图有Z轴最大值。
``` rust
// 没有用包围盒计算，这里有点误差没关系
if target_transform  
    .translation  
    .xy()  
    .distance(compare_transform.translation.xy())  
    < (target.crop_width.max(target.crop_height) as f32)  
{  
    max_z = max_z.max(compare_transform.translation.z);  
}
```

# 拼图选中效果
![200](/posts/attachments/Pasted%20image%2020241105203947.png)

为了这个选中效果， 在切图时，多切了一张同样大小，但颜色是白色的图片， 放在彩色图片下面，两种图片分别加个`ColorImage`和`WhiteImage`的组件用来区分。

添加MoveStart触发器，当选中拼图时，将当前拼图和相连拼图都加上一个`Selected`组件。
``` rust
fn on_add_move_start(  
    trigger: Trigger<OnInsert, MoveStart>,  
    query: Query<&MoveTogether>,  
    mut commands: Commands,  
) {  
    let move_together = query.get(trigger.entity()).unwrap();  
    commands.entity(trigger.entity()).insert(Selected);  
    for entity in move_together.iter() {  
        if entity == &trigger.entity() {  
            continue;  
        }  
        commands.entity(*entity).insert(Selected);  
    }  
}
```

添加Selected触发器，将彩色图片向左上移动几个像素，将白色图片改一个颜色
``` rust
fn on_selected(  
    trigger: Trigger<OnInsert, Selected>,  
    query: Query<&Children>,  
    mut q_image: Query<&mut Transform, (With<ColorImage>, Without<WhiteImage>)>,  
    mut w_image: Query<&mut Sprite, (With<WhiteImage>, Without<ColorImage>)>,  
) {  
    let children = query.get(trigger.entity()).unwrap();  
  
    for child in children.iter() {  
        if let Ok(mut transform) = q_image.get_mut(*child) {  
            transform.translation.x -= 4.0;  
            transform.translation.y += 4.0;  
        }  
        if let Ok(mut image) = w_image.get_mut(*child) {  
            image.color = Color::Srgba(YELLOW);  
        }  
    }  
}
```
取消选择拼图时， 就是反向移动和改回白色。