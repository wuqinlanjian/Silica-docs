# 图片

`Image` 用来绘制贴图：物品图标、成就图标、界面边框、外部美术资源都可以。`Image` 默认尺寸是 16×16，宽度或高度写 0 时回落到默认值，不会像容器那样自适应拉伸。

```sil
Image { src: "textures/items/diamond"; width: 32; height: 32; }
```

贴图会被拉伸到控件的整个矩形，不做等比裁剪。需要保持比例时，自己按宽高比设置尺寸。

## `src` 的三种形式

| 写法 | 含义 | 例子 |
| --- | --- | --- |
| 游戏贴图路径 | 直接交给游戏渲染器查找 | `textures/items/diamond` |
| `file:` 前缀 | 指向磁盘上的图片文件 | `file:ui/logo.png` |
| 任意 `.png` / `.tga` 路径 | 同样按磁盘文件处理 | `ui/logo.png`、`D:/art/cat.png` |

判定规则：

1. 以 `file:` 开头 → 磁盘文件。
2. 扩展名是 `.png` 或 `.tga`（不区分大小写）→ 磁盘文件。
3. 其它拼写原样传给游戏，按内置资源路径查找。

因此 `textures/ui/icon` 走游戏资源，而 `textures/ui/icon.png` 会被当成模组目录下的文件。

## 使用磁盘图片

Silica 不直接读取文件交给渲染器，而是把图片复制进一个自动生成的资源包：

```sil
Image { src: "file:ui/banner.png"; width: 240; height: 64; }
```

工作方式：

- 目标资源包位于 `<模组目录>/resource_packs/silica_images`。
- 文件被复制为 `textures/silica/<hash>.<ext>`，再以该贴图名绘制。
- LeviLamina 会自动把每个模组的 `resource_packs` 目录注册为资源包来源，不需要改全局包列表。
- 相对路径以 `Silica::SetRootPath` 设置的目录为基准；通常就是模组目录。
- 同名内容按大小和修改时间缓存；文件变化后下一次解析会重新导入。
- 只接受 `.png` 和 `.tga`。
- 绝对路径只要位于模组目录内也可用；指向模组目录之外的路径会被拒绝并记录一次日志：

```text
silica: refusing image outside the mod directory: D:/...
```

!!! tip "推荐用相对路径"
    把美术资源放进模组目录，用 `ui/xxx.png` 这样的相对路径引用，部署到别的机器时无需修改。

### 首次加载与重试

编译文档后会调用 `imagepack::Warmup` 预导入图片。若此时客户端资源包系统尚未就绪，解析会失败，控件在后续帧自动重试；图片出现前会绘制一个暗色占位框。

### 缺失资源

- 文件不存在或贴图名拼错：绘制占位框，并记录一次

  ```text
  silica: image texture not available: ui/typo.png
  ```

- 游戏内置贴图名不存在时同样回落到占位框。

占位框是半透明深色矩形加描边，方便一眼看出位置和尺寸。

## 着色

图片使用控件的 `color` 作为乘法着色，默认白色表示原色：

```sil
Image { src: "textures/ui/icon"; width: 20; height: 20; color: "#FF8080"; }
```

`color` 的 alpha 与控件透明度相乘，因此也可以用 `opacity` 整体淡出。

## 九宫格边框

需要可拉伸的面板边框时，用 `border-image`（别名 `nine-slice`）代替纯色描边：

```sil
Panel {
    width: 240;
    height: 120;
    border-image: ui/panel.png 8;
}
```

语法：

```text
border-image: 贴图 切片...
border-image-slice: 切片...
```

- 贴图可写游戏路径或磁盘图片路径，**不要加引号**。
- 切片单位是**纹理像素**，不是百分比。
- 切片数量与含义：

| 数量 | 含义 |
| --- | --- |
| 1 | 四边相同 |
| 2 | 上下 / 左右 |
| 3 | 上 / 左右 / 下 |
| 4 | 上 / 右 / 下 / 左 |

- 四角保持原尺寸，边沿单向拉伸，中间双向拉伸。
- 盒子小于两侧切片总和时，切片按比例缩小。
- `border-image: none` 清除九宫格边框。
- 九宫格边框存在时，普通 `border-width` 描边不再绘制。

!!! note "边框图片同样走资源包"
    磁盘边框图会像 `Image` 一样被复制进 `resource_packs/silica_images`，并受模组目录边界限制。

## 渐变

`background` 可以直接给渐变：

```sil
Panel {
    width: 200;
    height: 60;
    background: linear-gradient(to right, #23262A, #3A3F45);
}
```

支持的方向与写法：

| 方向 | 写法 |
| --- | --- |
| 向右 | `to right` / `right` / `row` |
| 向左 | `to left` / `left` |
| 向下 | `to bottom` / `bottom` / `down` / `vertical` |
| 向上 | `to top` / `top` / `up` |
| 角度 | `45deg`、`180deg`（0° 朝上，顺时针） |

- 每个颜色后可跟位置（`0`–`1` 或百分比）：`#000 0%, #FFF 100%`。
- 角度会被归到最接近的轴向（水平或垂直），不支持任意斜向渐变。
- 省略位置时自动均分。
- 也支持旧写法：`background: #23262A #3A3F45`（两色以上）或用 `horizontal`/`vertical` 指定方向。
- 透明色可用 `transparent`。
- 渐变色不参与 `transition` 插值；悬停状态色会覆盖渐变。

## 描边与圆角

```sil
Panel {
    width: 180;
    height: 80;
    background: #26292D;
    radius: 8;
    border-width: 1;
    border-color: #454A50;
}
```

- `border: 1 #454A50` 也可同时写入宽度和颜色。
- 圆角同时作用于填充和描边。

## 阴影 {#shadow}

```sil
Panel {
    width: 180;
    height: 80;
    background: #26292D;
    shadow: 0 4 12 #00000080;
}
```

`shadow` 接受 1–4 个数字加可选颜色：

| 数字个数 | 含义 |
| --- | --- |
| 1 | 模糊半径 |
| 2 | 偏移 x / y |
| 3 | 偏移 x / y + 模糊 |
| 4 | 偏移 x / y + 模糊 + 扩散 |

`shadow: none` 清除阴影。阴影按圆角轮廓绘制，位于填充之下。

## 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 显示黑白格或暗色框 | 贴图路径不存在 | 检查 `src` 拼写与资源包 |
| 文件图片不出现 | 路径在模组目录之外，或扩展名不是 png/tga | 放进模组目录并使用相对路径 |
| 图片被拉伸变形 | `Image` 会把整张贴图铺满矩形 | 按原始比例设置宽高 |
| 九宫格边框比例不对 | 切片按纹理像素解释 | 用图片编辑工具确认边缘像素宽度 |
| 边框图片不显示 | 路径加了引号 | 去掉引号 |

下一步：[控件参考](../widgets.md) · [C++ 集成](../cpp-guide.md)
