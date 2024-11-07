---
Status: 🌲
tags:
  - blog
Links:
  - "[Bevy MOC](Bevy%20MOC.md)"
categories:
  - 编程语言
  - 游戏开发
Series:
  - Bevy制作拼图游戏
title: Bevy制作拼图游戏 Day 3
date: 2024-11-01T11:36:20
Author:
  - ZoOL
share: true
externalLink: https://github.com/foxzool/jigsaw_puzzle
---

# 选中拼图
Bevy 0.15开始提供点击插件，图片点击需要打开`bevy_sprite_picking_backend`的feature
```
bevy = { version = "0.15.0-rc.2", features = ["wayland", "bevy_sprite_picking_backend"] }
```
我们准备实现的移动方式是点击选中抬起图片， 图片随着鼠标移动， 再次点击时放下。

首先在原来的拼图实体上挂一个`Observe`
``` rust
let entity = commands  
    .spawn((  
        Piece(piece.clone()),  
        Transform::from_xyz(calc_position.x, calc_position.y, piece.index as f32),  
    ))  
    .observe(on_click_piece)  
    .id();

#[derive(Component)]  
struct MoveStart {  
    image_position: Transform,  
    click_position: Vec2,  
}
  
fn on_click_piece(  
    trigger: Trigger<Pointer<Click>>,  
    mut image: Query<(&mut Transform, Option<&MoveStart>)>,  
    camera: Single<(&Camera, &GlobalTransform), With<Camera2d>>,  
    mut commands: Commands,  
) {  
    if let Ok((mut transform, opt_moveable)) = image.get_mut(trigger.entity()) {  
        let click_position = trigger.event().pointer_location.position;  
        let (camera, camera_global_transform) = camera.into_inner();  
  
        let point = camera  
            .viewport_to_world_2d(camera_global_transform, click_position)  
            .unwrap();  
  
        // move end  
        if opt_moveable.is_some() {  
            transform.translation.z = 0.0;  
            commands.entity(trigger.entity()).remove::<MoveStart>();  
            commands.trigger_targets(MoveEnd, vec![trigger.entity()]);  
        } else {  
            transform.translation.z = 1.0;  
            commands.entity(trigger.entity()).insert(MoveStart {  
                image_position: *transform,  
                click_position: point,  
            });  
        }  
    }  
}
```
在点击拼图时, 给实体挂上/取消 `MoveStart` 组件，为后面移动拼图做准备。
注意在上面我们再次点击取消选中拼图时， 给同一个拼图的Entity发送了一个`MoveEnd`的事件，此事件用来触发拼图放下的事件（磁吸拼图，判断拼图完成等）

# 移动拼图
在上一步点击拼图时，我们记录了点击的起始位置，这样我们用`move_piece`来计算移动距离
``` rust
fn move_piece(  
    window: Single<&Window>,  
    camera_query: Single<(&Camera, &GlobalTransform)>,  
    moveable: Single<(&mut Transform, &MoveStart)>
) {  
    let (camera, camera_transform) = *camera_query;  
    let Some(cursor_position) = window.cursor_position() else {  
        return;  
    };  
    let Ok(point) = camera.viewport_to_world_2d(camera_transform, cursor_position) else {  
        return;  
    };  
  
    let (mut transform, move_start) = moveable.into_inner();  
    let cursor_move = point - move_start.click_position;  
    let move_end = move_start.image_position.translation + cursor_move.extend(0.0);    
    transform.translation = move_end;  
}
```
通过计算鼠标移动的相对距离，将此距离应用到拼图的坐标上， 移动拼图完成。

# 磁吸拼图
当拼图放下后，判断一下当前拼图和周围拼图的位置和排列关系，如果靠近到一定范围，且两者排列是顺序关系，将当前拼图位置移动到合适位置，将两个拼图拼起来。

同时我们在Piece实体上再增加一个`MoveTogether`组件， 当一个组件移动时，相连的组件一起移动。
``` rust
let mut iter = query.iter_combinations_mut();  
let end_entity = trigger.entity();  
  
let mut all_entities = HashSet::default();  
while let Some([(e1, p1, transform1, together1), (e2, p2, transform2, together2)]) =  
    iter.fetch_next()  
{
	// 当拼图在左侧时
	if target.is_on_the_left_side(compare, target_loc, compare_loc) {  
	    debug!("{} on the left side {}", target.index, compare.index);  
	    target_transform.translation.x = compare_transform.translation.x - target.width;  
	    target_transform.translation.y = compare_transform.translation.y;  
	    has_snapped = true;  
	}
	// 将这两个拼图的实体ID保持起来
	if has_snapped {  
	    let mut merged_set: HashSet<_> = together1.union(&together2).cloned().collect();  
	    merged_set.insert(e1);  
	    merged_set.insert(e2);  
	  
	    all_entities.extend(merged_set);  
	}
}

if all_entities.len() == generator.pieces_count() {  
    debug!("所有拼图已经完成");  
}

// 执行触发器，将所有相关的实体ID发送出去
commands.trigger(CombineTogether(all_entities))
```

触发器更新`MoveTogether`
``` rust
#[derive(Event)]  
struct CombineTogether(HashSet<Entity>);  
  
fn combine_together(trigger: Trigger<CombineTogether>, mut query: Query<&mut MoveTogether>) {  
    let entities: Vec<Entity> = trigger.event().0.iter().cloned().collect();  
    let mut together_iter = query.iter_many_mut(&entities);  
    while let Some(mut move_together) = together_iter.fetch_next() {  
        move_together.0 = trigger.event().0.clone();  
    }  
}
```

修改一下`move_piece`系统, 将相对位移应用到每块相关拼图上
```rust
  
fn move_piece(  
    window: Single<&Window>,  
    camera_query: Single<(&Camera, &GlobalTransform)>,  
    moveable: Single<(&mut Transform, &MoveStart, &MoveTogether)>,  
    mut other_piece: Query<&mut Transform, Without<MoveStart>>,  
) {  
    let (camera, camera_transform) = *camera_query;  
    let Some(cursor_position) = window.cursor_position() else {  
        return;  
    };  
    let Ok(point) = camera.viewport_to_world_2d(camera_transform, cursor_position) else {  
        return;  
    };  
  
    let (mut transform, move_start, move_together) = moveable.into_inner();  
    let cursor_move = point - move_start.click_position;  
    let move_end = move_start.image_position.translation + cursor_move.extend(0.0);  
    let offset = move_end - transform.translation;  
    transform.translation = move_end;  
  
    for other in move_together.iter() {  
        if let Ok(mut other_transform) = other_piece.get_mut(*other) {  
            other_transform.translation += offset;  
        }  
    }  
}
```
实现效果如下：
![2024-11-04_20-07.png](/posts/attachments/2024-11-04_20-07.png)