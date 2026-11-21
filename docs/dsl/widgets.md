# 常用控件

本篇按“拿来就用”的方式介绍内置控件：什么时候用、最小写法是什么、有哪些容易踩的行为。完整属性表见[控件参考](../widgets.md)。

## 一览

| 控件 | 用途 | 交互 |
| --- | --- | --- |
| `Container` | 排版容器 | 本身不响应点击（可挂 `Part`） |
| `Panel` | 纯色块 / 边框 / 分隔线 | 无子控件 |
| `Label` | 单行或自动换行文本 | 无 |
| `Button` | 按钮 | 点击、回车/空格激活 |
| `Checkbox` | 勾选或开关 | 点击、空格切换 |
| `Slider` | 连续数值 | 拖动、方向键 |
| `TextBox` | 单行文本输入 | 点击聚焦、键入、回车提交 |
| `Dropdown` | 下拉选择 | 点击展开、选择、Esc 收起 |
| `ScrollView` | 裁剪滚动区域 | 滚轮、拖动滚动条 |
| `Image` | 纹理或图片文件 | 无 |
| `Part` | 装饰层 | 不参与布局、不响应点击 |
| `ForEach` | 按数组生成行 | 行内控件各自交互 |

下面的示例默认放在一个竖排 `Container` 中，省略外层容器。

## Label

显示一行文本。默认不换行，超宽会被截断并加省略号。

```sil
Label {
    text: "生命值：{{@health}} / 100";
    height: 16;
    font-size: 0.85;
    color: #C8CFD5;
}
```

需要多行时开启 `wrap`，并给出高度；高度决定最多显示几行，超出部分不绘制：

```sil
Label {
    text: "这是一段较长的说明，开启 wrap 后按控件宽度折行。";
    width: 260;
    height: 40;
    font-size: 0.6;
    wrap: true;
    color: #87919A;
}
```

- 高度为 `0` 时：普通 Label 取一行高度；`wrap: true` 取 3 行高度。
- `text-align` 支持 `left` / `center` / `right`。
- 颜色、字体、透明度等通用属性对文字生效。

## Button

```sil
Button {
    text: "应用设置";
    width: 112;
    height: 24;
    background: #2F6B37;
    border: 1 solid #5CD65C;
    color: #D9F2D9;
    hover-background: #3C8446;
    pressed-background: #275B2E;
    onClick: "apply_settings";
}
```

- 点击判定：在按钮内按下并在按钮内释放。
- 事件可以写回调名（字符串）或脚本（`{ ... }`）。脚本优先；脚本存在时不会再调用同名 C++ 回调。
- 悬停与按下的默认高亮会被 `hover-background` / `pressed-background` 覆盖。
- 通过焦点环选中后，回车或空格会触发点击。

## Checkbox

既可用于勾选，也可把方块隐藏后当作开关：

```sil
Checkbox {
    text: "启用绿色主题";
    width: 150;
    height: 18;
    checked: @theme_green;
    accent: #5CD65C;
    onChange: { @theme_color = event.checked ? "#5CD65C" : "#E05252"; };
}
```

- `checked` 是布尔绑定；用户切换时会写回绑定的数据键（两向绑定）。
- 事件脚本内读 `event.checked`（布尔）或 `event.value`。
- 使用回调名时，值以文本 `true` / `false` 传入。
- `box-size`、`text-gap`、`check-inset`、`show-box: false`、`accent` 控制外观。
- 隐藏方块 + `Part` 可以做成拨动开关，示例见[示例集锦](../examples.md#switch)。

## Slider

```sil
Slider {
    width: 220;
    height: 20;
    min: 0;
    max: 100;
    value: @health;
    step: 1;
    accent: #5CD65C;
    smooth: true;
    onChange: { @health = event.value; };
}
```

- 拖动时连续更新；`step` 会吸附到最近的刻度；`smooth` 让滑块位置平滑追随数值。
- 事件脚本内读 `event.value`（数字）。
- 使用回调名时，值以格式化文本传入（如 `42.5`）。
- 获得焦点后可用左右方向键调整。

## TextBox

```sil
TextBox {
    width: 180;
    height: 22;
    text: @player_name;
    placeholder: "输入名称";
    max-length: 32;
    onChange: { @player_name = event.text; };
    onSubmit: "rename_player";
}
```

- 点击获得焦点后开始输入；`Backspace` 删除，`Esc` 放弃焦点。
- `onChange` 在每次文本变化时触发；失去焦点时若文本有变化，会再通知一次。
- `onSubmit` 在回车时触发，脚本读 `event.text`。
- `max-length` 按字节截断，中文等多字节字符会按完整字符边界处理。
- 获得焦点时方向键、字符键由输入框消费，不会触发宿主快捷键。

!!! note "输入法与光标"
    Silica 自己绘制光标，不接管系统输入法候选框。需要中文等复杂输入时，宿主应提供可用的字符事件或输入框接管；框架只接收逐字符输入。

## Dropdown

```sil
Dropdown {
    width: 160;
    height: 24;
    options: "低, 中, 高";
    selected: @quality;
    onChange: { @quality = event.index; };
}
```

- `options` 是逗号分隔的纯文本列表，首尾空白会被去掉；选项内不能包含逗号。
- `selected`（别名 `index` / `value`）是索引，从 `0` 开始；绑定到数据键时用户选择会写回。
- 展开时点击列表项选择；点击外部或按 `Esc` 收起；失去焦点自动收起。
- 展开后的列表可以超出控件原高度，点击判定会同步扩展。
- 事件脚本读 `event.index` / `event.value`（数字）与 `event.text`（选项文字）；回调名收到的是索引文本。

## ScrollView

```sil
ScrollView {
    name: scroll_area;
    width: 0;
    height: 240;
    padding: 6;
    scroll-step: 28;
    scrollbar-width: 3;
    scrollbar-inset: 3;
    scrollbar-radius: 0;

    // 子节点会进入垂直内容区，依次向下排列
    Label { text: "第一行"; height: 16; margin-bottom: 4; }
    Label { text: "第二行"; height: 16; margin-bottom: 4; }
    // ...
}
```

- 子控件放入内部内容容器，始终垂直排列；在 `ScrollView` 上写 `layout` 会被忽略。
- 视口外的内容不绘制、不可点击。
- 滚轮滚动与滚动条拖动都先改变目标偏移，再由每帧缓动逼近；`smooth-scroll: false` 可立即跳转。
- `show-scrollbar: false` 可隐藏滚动条，滚轮仍然可用。
- 内容高度由子控件自动计算；不要给内部内容设置固定高度。

## Image

```sil
Image {
    src: "textures/items/diamond";
    width: 32;
    height: 32;
}

Image {
    src: "file:ui/logo.png";
    width: 24;
    height: 24;
}
```

- `src` 可以是游戏内部纹理路径，也可以是 `file:` 开头的图片文件（也接受绝对路径与 `.png` / `.tga` 相对路径）。
- 图片会被拉伸到控件矩形，不保持比例；需要保持比例时自行计算宽高。
- 文件图片由 Silica 复制到生成的资源包中，路径必须位于模组根目录内。
- 加载失败时绘制暗色占位块，并在日志中记录一次。

详见[图片](images.md)。

## Part

`Part` 是挂在任意控件上的装饰层：它不参与布局，每帧按锚点贴合宿主，并可用宿主状态控制显示。适合做高亮条、角标、开关滑块、贴图皮肤。

```sil
Button {
    name: fav_button;
    text: "收藏";
    width: 96;
    height: 24;

    Part {
        anchor: right;
        width: 20;
        height: 20;
        inset-right: 2;
        src: "textures/ui/icon_star";
        tint: #FFD75A;
        when: "hovered";
    }
}
```

常用锚点：`fill`、`top`、`bottom`、`left`、`right`、`center` 与四个角。`inset` 控制内缩（1–4 个数字，按上/右/下/左的 CSS 规则），`dx` / `dy` 做额外偏移。

`when` 用空格分隔多个条件，全部满足才显示；`!` 表示取反：

```sil
Part { anchor: left; width: 2; background: #5CD65C; when: "checked"; layer: under; }
Part { anchor: fill; background: #FFFFFF07; when: "hovered !pressed"; layer: under; }
```

- 可用状态由宿主控件决定：所有控件都有 `hovered`、`pressed`、`focused`、`visible`、`disabled`；`Checkbox` 额外有 `checked`，`Dropdown` 有 `open` / `collapsed`，`TextBox` 有 `empty` / `filled` / `editing`。
- `layer: under` 画在宿主自身内容之下，默认画在其上。
- `Part` 自身不接收点击；它只是视觉层。

## 控件与数据的分工

控件只负责显示与用户输入，不保存业务规则。建议：

- 界面状态（当前页、是否展开）可以放在 `DataStore`，由脚本修改。
- 业务状态（背包内容、配置项）由 C++ 写入 `DataStore`，界面只读绑定。
- 需要跨帧保持的输入值通过双向绑定写回，而不是在 C++ 里轮询控件。

下一章介绍绑定的完整规则。

---

[上一章：布局](layout.md) · [下一章：响应式绑定](bindings.md)
