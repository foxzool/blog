---
Status: 🌲
tags:
  - bevy
  - rust
Links:
  - "[Bevy MOC](Bevy%20MOC.md)"
title: Bevy制作拼图游戏 Day 1
Date: 2024-10-22T11:08:19
share: true
categories:
  - 游戏开发
  - 编程语言
Author:
  - ZoOL
Series:
  - Bevy制作拼图游戏
Slug: create a jigsaw puzzle game by bevy Day 1
---
# 目标
使用bevy制作拼图游戏(jigsaw puzzle)
- [x] 将图片按照参数分割成小图片 
- [ ] 打乱后,点击拖动放置
- [ ] 判断是否成功拼完或者重新开始
- [ ] 支持WASM
# 切割图片

## 思路分析
网上搜索一番后，找到了一个[网页版拼图生成器](https://draradech.github.io/jigsaw/jigsaw.html)
![Pasted image 20241026185259.png](/posts/attachments/Pasted%20image%2020241026185259.png)
和rust对应的[SVG拼图生成器](https://gitlab.switch.ch/ub-unibas/puzzle-app/puzzle-paths)
阅读代码后，可以看到拼图每个块(Piece)， 由上、右、下、左四条边(Edge)组成，每条边分为直边(StraightEdge)和锯齿边(IdentedEdge)两种， 锯齿边由三段二次贝塞尔曲线拼成， 突起方向和大小由TabSize和Jitter控制。

## 添加基础库
``` toml
bezier-rs = "0.4.0"  // 贝塞尔曲线
image = "0.25.4"     // 图像处理
imageproc = "0.25.0" // 图像高级处理
log = "0.4.22"       // 日志库
glam = "=0.24.2"     // 数学计算
```

## 预处理图片
准备构造器
```rust
pub struct JigsawGenerator {  
    /// 原始图像 
    origin_image: DynamicImage,  
    /// 每列块数
    pieces_in_column: usize,  
    /// 每行块数
    pieces_in_row: usize,  
    /// 凸起系数
    tab_size: Option<f32>,  
    /// “抖动”系数。数字越大，拼图就越不对称  
    jitter: Option<f32>,  
    /// 生成拼图时随机性的可选种子值。
    seed: Option<usize>,  
}
```
从外部参数读取图片地址, 载入后,准备按照9 x 6的方式生成拼图
``` rust
let image_path = env::args().nth(1).unwrap_or("images/raw.jpeg".to_string());  
info!("Start to load {}", image_path);  
let img = image::open(image_path).expect("Failed to open image");  
info!("load image successfully!");  
let template = JigsawGenerator::new(img, 9, 6).generate();
```
为了防止图片太大, 影响后续处理速度, 先按固定的尺寸缩放一下
``` rust
/// 超过固定尺寸的进行缩放
fn scale_image(image: &DynamicImage) -> RgbaImage {  
    let (width, height) = image.dimensions();  
    let scale = if width > MAX_WIDTH || height > MAX_HEIGHT {  
        let scale_x = MAX_WIDTH as f32 / width as f32;  
        let scale_y = MAX_HEIGHT as f32 / height as f32;  
        scale_x.min(scale_y)  
    } else {  
        1.0  
    };  
    if scale < 1.0 {  
        image  
            .resize(  
                (width as f32 * scale) as u32,  
                (height as f32 * scale) as u32,  
                image::imageops::FilterType::Lanczos3,  
            )  
            .to_rgba8()  
    } else {  
        image.to_rgba8()  
    }  
}
```
## 计算轮廓
按照算法计算轮廓线.
注意坐标系原点是图片的左上角,x朝右, y朝下
```rust
let (starting_points_x, piece_width) = divide_axis(image_width, pieces_in_column);  
let (starting_points_y, piece_height) = divide_axis(image_height, pieces_in_row);  
let mut contour_gen = EdgeContourGenerator::new(  
    piece_width,  
    piece_height,  
    self.tab_size,  
    self.jitter,  
    self.seed,  
);
let mut vertical_edges = vec![];  
let mut horizontal_edges = vec![];
/// 将每条边的数据存下来
for index_y in 0..starting_points_y.len() {  
    let mut left_border = true;  
    for index_x in 0..starting_points_x.len() {  
        horizontal_edges.push(if top_border {  
            Edge::StraightEdge(StraightEdge {  
                starting_point: (starting_points_x[index_x], 0.0),  
                end_point: (end_point_pos(index_x, &starting_points_x, image_width), 0.0),  
            })  
        } else {  
            Edge::IndentedEdge(IndentedEdge::new(  
                (starting_points_x[index_x], starting_points_y[index_y]),  
                (  
                    end_point_pos(index_x, &starting_points_x, image_width),  
                    starting_points_y[index_y],  
                ),  
                &mut contour_gen,  
            ))  
        });
```
处理后,我们得到了水平边和垂直边的起始点、控制点、终点等坐标点。
我们按照上、右、下、左的顺序，将每条边先转为贝塞尔曲线，然后将4条边的曲线合起来组成了一个闭合路径(subpath)
``` rust
let top_beziers = top_edge.to_beziers(false);  
let right_beziers = right_edge.to_beziers(false);  
let bottom_beziers = bottom_edge.to_beziers(true);  
let left_beziers = left_edge.to_beziers(true);  
let beziers: Vec<_> = vec![top_beziers, right_beziers, bottom_beziers, left_beziers]  
    .into_iter()  
    .flatten()  
    .collect();  
let sub_path: Subpath<PuzzleId> = Subpath::from_beziers(&beziers, true);
```
## 裁剪拼图
我们使用`imageproc`的绘画工具把线条画一下
![Pasted image 20241029105308.png](/posts/attachments/Pasted%20image%2020241029105308.png)
感觉不错、接下来将每个小块裁出来，注意原本每条边的尺寸是按9x6的块数计算后获得的，但由于每个块可能有几条边会凸起，所以裁剪的位置要更大。幸好bezier-rs提供了bounding-box的计算
```rust
let [box_min, box_max] = piece  
    .sub_path  
    .bounding_box()  
    .expect("Failed to get bounding box");
```
有了包围盒的坐标，就可以计算出原图片上对应的起始坐标，进行图片裁剪
```rust
let mut piece_image = self  
    .origin_image  
    .view(  
        top_left_x as u32,  
        top_left_y as u32,  
        width as u32,  
        height as u32,  
    )  
    .to_image();
```
将不在封闭路径里的点改为透明, 这里使用了image的并行运算加速处理
```rust
piece_image  
    .par_enumerate_pixels_mut()  
    .for_each(|(x, y, pixel)| {  
        let point = DVec2::new(top_left_x + x as f64, top_left_y + y as f64);  
        if !piece.contains(point) {  
            *pixel = Rgba([0, 0, 0, 0])  
        }  
    });
```
再描一个白边, 完成。
![200](/posts/attachments/Pasted%20image%2020241029125651.png)
将代码移入workspace,准备开始用bevy搭建游戏
```toml
[workspace]  
resolver = "2"  
members = [  
    "jigsaw_puzzle_generator",  
]
```
