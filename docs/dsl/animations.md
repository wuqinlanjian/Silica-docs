# 动画与过渡

Silica 提供两层时间驱动能力：

| 机制 | 适合 | 作用范围 |
| --- | --- | --- |
| `transition` | 值变化时平滑插值 | 颜色、透明度 |
| `@keyframes` | 循环或定次播放的动效 | 透明度、渲染缩放 |

两者都由宿主每帧调用 `TickTree` 推进，使用真实帧间隔，因此动画速度与帧率无关。帧间隔会被限制在 0.25 秒以内，卡顿后不会瞬移。

## 过渡 `transition`

在控件或样式里声明要平滑的属性：

```sil
Style MenuButton {
    background: #33363A;
    color: #E6E6E6;
    radius: 4;
    transition: background .15s ease-out, color .15s ease-out;
}
```

语法：

```text
transition: 属性 时长 [缓动][, 属性 时长 [缓动]]...
```

- 时长写成 `0.3s`、`300ms` 或裸数字（单位秒）。
- 缓动可省略，默认 `linear`。
- `all` 表示所有受支持的属性。
- 属性名可写 `background`/`background-color`/`bg`，`color`/`text-color`/`foreground`；内部会归一。

### 支持插值的属性

| 属性 | 说明 |
| --- | --- |
| `background` | 纯色填充（渐变不插值） |
| `color` | 文字颜色 |
| `border-color` | 描边颜色 |
| `opacity` | 整体不透明度 |

`width`、`height`、`radius`、`padding` 等几何属性不会插值；改变它们会立即生效。需要尺寸动效时，用 `@keyframes` 的 `transform: scale`。

### 什么变化会走过渡

过渡作用于“运行期值发生变化”的场合：

```sil
Var hot: false;

Panel {
    width: 160;
    height: 40;
    transition: background .25s ease-out;
    background: @hot ? "#8A3B3B" : "#33363A";  // 状态切换时平滑过渡
}
```

- `@state ? a : b` 形式会注册为每帧求值的动态规则，切换时走过渡。
- 普通绑定（`background: @color`）在 `DataStore` 更新时也走过渡。
- 首次构建时直接落到目标值，不会从默认值播放一次动画。

!!! warning "悬停状态色不参与过渡"
    `hover-background`、`pressed-background`、`disabled-background` 直接替换填充色，切换是瞬时的。想要平滑的悬停高亮，用 `when` 条件或绑定驱动普通 `background`，再配合 `transition`。

## 关键帧 `@keyframes`

在文档顶层定义：

```sil
@keyframes pulse {
    0%   { opacity: 1; }
    50%  { opacity: .55; }
    100% { opacity: 1; }
}

@keyframes pop {
    0%   { transform: scale(.8); opacity: 0; }
    60%  { transform: scale(1.06); opacity: 1; }
    100% { transform: scale(1); opacity: 1; }
}
```

在控件上播放：

```sil
Label {
    text: "加载中";
    animation: pulse 1.5s infinite;
}
```

语法：

```text
animation: 名称 时长 [次数|infinite]
```

- 名称必须与 `@keyframes` 后的名字完全一致（区分大小写）。
- 省略时长时为 1 秒；省略次数时播放 1 次。
- `infinite` 表示无限循环。
- `animation: none` 停止并清除当前动画。

### 关键帧支持的属性

| 写法 | 含义 |
| --- | --- |
| `opacity: 0..1` | 与控件自身透明度相乘 |
| `transform: scale(0.9)` | 以控件中心缩放 |
| `scale: 0.9` | 与上一行等价 |

其它属性写在关键帧里会被忽略。需要颜色循环时，用脚本或 `DataStore` 定时改值，再依靠 `transition` 插值。

### 百分比与插值

- 步骤写成 `0%`、`50%`、`100%` 等，编译器会按偏移排序。
- 未覆盖到的属性在整段动画中保持 1（透明度和缩放都是）。
- 两个步骤之间线性插值；缓动不能写在关键帧步骤里。

### 缩放是纯渲染变换

`transform: scale` 只影响绘制，不改变布局与命中区域：

```sil
Button {
    text: "确认";
    animation: pop .3s;
}
```

- 缩放围绕控件中心进行。
- 父容器绘制子节点时临时放大其矩形，绘制完立即还原。
- 点击、悬停和布局始终按原始尺寸计算。

因此缩放不会把兄弟控件挤开，也不会让点击区域“跟着视觉变大”。

### 透明度叠加

关键帧透明度与控件基础透明度相乘：

```sil
Label {
    opacity: .5;
    animation: pulse 2s infinite;   // 关键帧最高 1 时，最终为 .5
}
```

若关键帧最低为 `.55`，最终最低为 `.275`。

## 缓动

| 名称 | 曲线 |
| --- | --- |
| `linear` | 匀速 |
| `ease-out` / `easeout` / `out` | 先快后慢（二次） |
| `ease-in-out` / `ease` / `cubic` | 两端慢、中间快（三次） |
| `ease-in-out-sine` / `sine` | 正弦缓动 |

`transition` 与 ScrollView 的滚动动画都使用这些名称；`@keyframes` 使用线性插值。

## 重新触发动画

```sil
Button {
    text: "播放";
    animation: pop .4s;
    onClick: { @plays = @plays + 1; }
}
```

- 重新应用同名 `animation` 属性会重置时间并从头播放。
- 目前没有“由状态变化自动重播”的声明语法；需要重播时，在脚本或 C++ 里重新设置 `animation` 属性。
- 修改 `@keyframes` 内容后需要重载文档，已构建的控件不会自动拾取新步骤。

## 组合示例

```sil
Var muted: false;

Style ToggleRow {
    height: 44;
    padding: 8;
    background: #2C2F33;
    radius: 6;
    transition: background .2s ease-out, color .2s ease-out;
}

@keyframes breathe {
    0%   { opacity: .45; }
    50%  { opacity: 1; }
    100% { opacity: .45; }
}

Container {
    class: ToggleRow;
    layout: horizontal;

    Label {
        text: "静音提示";
        width: 160;
        height: 28;
        color: @muted ? "#777" : "#E8E8E8";
    }
    Label {
        text: "●";
        width: 24;
        height: 28;
        color: @muted ? "#666" : "#7AD97A";
        animation: breathe 1.2s infinite;
    }
    Button {
        text: "切换";
        width: 72;
        height: 28;
        onClick: { @muted = !@muted; }
    }
}
```

## 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 过渡不生效 | 改的是几何属性，或悬停状态色 | 只对颜色/透明度使用过渡 |
| 动画不播放 | 名称拼写或大小写不符 | 让 `animation` 与 `@keyframes` 名称完全一致 |
| 动画只播一次 | 省略了次数 | 写 `infinite` 或指定次数 |
| 控件看起来变大但点不到 | 缩放在命中测试后不参与 | 这是预期行为；命中按原始矩形 |
| 深度嵌套时动画开销大 | 每帧对整棵子树采样 | 只给需要动效的节点加动画 |

下一步：[列表与循环](lists.md) · [图片](images.md)
