# 控件参考

本页按控件列出职责、默认值、完整属性表和状态/事件。用法教程见 [DSL 控件](dsl/widgets.md)。

## 控件一览

| 控件 | 职责 | 默认尺寸 | 可容纳 |
| --- | --- | --- | --- |
| `Container` | 布局容器，可画背景 | 0×0（按需自增） | 任意控件 |
| `Panel` | 纯装饰盒子 | 0×0 | 仅 `Part` |
| `Label` | 单行/多行文本 | 宽 0，高 14×字号 | 仅 `Part` |
| `Button` | 可点击按钮 | 120×20 | 仅 `Part` |
| `Image` | 贴图 | 16×16 | 仅 `Part` |
| `Checkbox` | 复选/开关 | 高 18 | 仅 `Part` |
| `Slider` | 数值滑块 | 高 20 | 仅 `Part` |
| `TextBox` | 单行输入 | 高 22 | 仅 `Part` |
| `Dropdown` | 下拉选择 | 160×24 | 仅 `Part` |
| `ScrollView` | 裁剪滚动的竖向视口 | 0×0 | 任意控件（进入内部内容容器） |
| `ForEach` | 数组重复器 | 0×0（高自适应） | 任意控件（作为行模板） |
| `Part` | 挂在控件上的装饰层 | 跟随锚点 | 无 |

!!! warning "只有容器能装任意子控件"
    `Panel`、`Button`、`Label`、`Image`、`Checkbox`、`Slider`、`TextBox`、`Dropdown` 只接受 `Part` 作为子节点。把其它控件放进去会报错 `'X' cannot contain 'Y'`。

## 通用属性

以下属性所有控件都接受：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `name` / `id` | 文本 | 节点名，供 `FindByName` 查找 |
| `x` / `y` | 数字 | 绝对定位坐标（布局容器会覆盖） |
| `width` / `height` | 数字 | 尺寸；0 表示交给布局决定 |
| `margin` | 数字 | 尾部间距（竖排底部、横排右侧） |
| `margin-top` / `margin-right` / `margin-bottom` / `margin-left` | 数字 | 分方向间距；前导方向用 `margin-top`/`margin-left` |
| `visible` | 布尔 | 不参与布局、不渲染、不响应 |
| `disabled` / `enabled` | 布尔 | 禁用态；子控件继承禁用状态 |
| `z-order` / `z` / `layer` / `z-index` | 数字 | 绘制与命中顺序，大者在上 |
| `opacity` | 0–1 | 整体透明度，参与过渡 |
| `background` / `bg` | 颜色/渐变 | 填充 |
| `color` / `text-color` | 颜色 | 文字色，也是图片的着色 |
| `radius` / `border-radius` | 数字 | 圆角 |
| `border-width` | 数字 | 描边宽度 |
| `border-color` | 颜色 | 描边颜色 |
| `border` | `宽度 [颜色]` | 同时设置描边 |
| `shadow` | `...` | 阴影，见[图片](dsl/images.md#shadow) |
| `gradient` / `bg-gradient` | 渐变 | 背景渐变 |
| `border-image` / `nine-slice` | 贴图 切片... | 九宫格边框 |
| `font-family` | 文本 | 字体名或 `default`/`smooth`/`unicode`/`rune`/`ui` |
| `text-align` / `align` | `left`/`center`/`right` | 文本对齐 |
| `transition` | 列表 | 颜色/透明度过渡 |
| `animation` | `名称 时长 [次数]` | 关键帧动画 |
| `class` / `style` | 样式名列表 | 应用全局样式 |
| `on<事件>` / `<事件>:` | 脚本或处理器名 | 见[事件](#events) |

`padding` 只对容器的布局有意义（`Container`、`ScrollView`、`ForEach`）；叶控件请用各自的文本内边距属性。

## Container

布局容器。默认 `absolute`，可画背景、描边、圆角。

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `layout` / `direction` | `vertical`/`column`/`v`、`horizontal`/`row`/`h`、`absolute`/`none` | 排列方式，默认 `absolute` |
| `padding` | 数字 | 四边内边距 |

```sil
Container {
    layout: vertical;
    padding: 8;
    width: 200;

    Label { text: "标题"; height: 20; }
    Label { text: "说明"; height: 16; }
}
```

- 竖向布局中，子控件宽度为 0 时拉伸至内容宽度。
- 横向布局中，子控件高度为 0 时拉伸至内容高度。
- 主轴上容器尺寸为 0 时，按子控件累计尺寸自动计算。
- 隐藏子控件不占空间。

## Panel

纯盒子，画背景、描边、圆角、阴影和九宫格边框，不能布局子控件。

| 属性 | 说明 |
| --- | --- |
| 通用外观属性 | 全部可用 |
| 子节点 | 仅 `Part` |

```sil
Panel {
    x: 10; y: 10;
    width: 120; height: 60;
    background: #2C2F33;
    radius: 6;
}
```

## Label

文本控件。默认单行，超出宽度截断；开启 `wrap` 后自动换行。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `text` / `value` | 文本/绑定 | 空 | 显示内容 |
| `font-size` / `font` | 数字 | 1 | 字号倍数 |
| `wrap` / `word-wrap` | 布尔 | `false` | 多行换行 |
| `color` | 颜色 | 近白 | 文字颜色 |
| `text-align` | 枚举 | `left` | 对齐 |

- 高度为 0：单行模式取 `14 × 字号`；换行模式取 3 行高度（`30 × 字号`）。
- 换行时优先在空格处断行，没有空格（中文、长单词）则按字符断开。
- 文本 `{{ }}` 模板见[响应式绑定](dsl/bindings.md#templates)。

## Button

按钮，默认使用 `UIStyle::PrimaryButton()`。点击（鼠标释放于内部，或聚焦时按激活键）触发 `click`。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `text` / `value` | 文本 | 空 | 按钮文字 |
| `font-size` | 数字 | 1 | 字号 |
| `onClick` / `on-click` / `action` | 处理器名 | 无 | C++ 命名处理器 |
| `onClick: { ... }` | 脚本 | 无 | 脚本优先于命名处理器 |
| 状态外观 | | | `hover-background`/`pressed-background`/`disabled-*` |

未设置状态色时，悬停叠加 10% 白、按下叠加 18% 黑。

## Image

见[图片](dsl/images.md)。主要属性：

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `src` / `source` / `texture` | 文本 | 空 | 游戏贴图路径或图片文件路径 |
| `width` / `height` | 数字 | 16 | 拉伸到矩形；0 回落为 16 |
| `color` | 颜色 | 白 | 乘法着色 |

## Checkbox

复选框。点击、空格或回车切换，`change` 报告新状态。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `text` / `label` | 文本 | 空 | 标题 |
| `checked` / `value` | 布尔 | `false` | 选中状态（单一标识符绑定可回写） |
| `accent` / `check-color` | 颜色 | 绿色 | 选中标记颜色 |
| `box-size` | 数字 | 0（随字号） | 方框边长；0 时取 `max(14, 14×字号)` |
| `text-gap` / `gap` | 数字 | 8 | 方框与文字间距 |
| `check-inset` | 数字 | -1 | 勾选标记内缩；负值按比例 |
| `show-box` | 布尔 | `true` | 隐藏内置方框，交给 Part 绘制 |
| `font-size` | 数字 | .9 | 字号 |
| `onChange` | 处理器名 | 无 | 收到 `"true"`/`"false"` |
| `onChange: { ... }` | 脚本 | 无 | 优先于处理器 |

状态：`checked`/`on`/`selected`、`unchecked`/`off`，以及通用的 `hovered`、`pressed`、`focused`、`disabled`。

## Slider

数值滑块，点击跳转、拖动连续更新，可量化到 `step`。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `min` | 数字 | 0 | 下限 |
| `max` | 数字 | 100 | 上限 |
| `value` | 数字 | 50 | 当前值（单一标识符绑定可回写） |
| `step` | 数字 | 0 | 0 表示连续 |
| `knob-size` | 数字 | 12 | 滑块直径 |
| `accent` / `fill-color` | 颜色 | 绿色 | 已填充轨道颜色 |
| `track-height` | 数字 | 4 | 轨道高度 |
| `track-color` | 颜色 | 深灰 | 轨道底色 |
| `knob-color` | 颜色 | 浅灰 | 滑块颜色 |
| `knob-active-color` | 颜色 | 近白 | 拖动时的滑块颜色 |
| `smooth` / `animate` | 布尔 | `false` | 滑块缓动到目标值（拖动也缓动） |
| `smoothing` / `smooth-speed` | 数字 | 20 | 缓动速率（每秒逼近系数） |
| `onChange` | 处理器名 | 无 | 收到格式化后的数值 |
| `onChange: { ... }` | 脚本 | 无 | 优先于处理器 |

## TextBox

单行输入框。点击聚焦，可输入、退格；`Esc` 失焦，`Enter` 提交。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `text` / `value` | 文本 | 空 | 内容（单一标识符绑定可回写） |
| `placeholder` / `hint` | 文本 | 空 | 占位提示 |
| `max-length` | 数字 | 32 | 最大字符数 |
| `font-size` | 数字 | .9 | 字号 |
| `text-padding` / `padding-x` | 数字 | 5 | 文本水平内边距 |
| `caret-color` | 颜色 | 白 | 光标颜色 |
| `caret-width` | 数字 | 1.5 | 光标宽度 |
| `placeholder-color` | 颜色 | 灰 | 占位文字颜色 |
| `onChange` | 处理器名 | 无 | 每次文本变化 |
| `onSubmit` / `onEnter` | 处理器名 | 无 | 回车提交 |
| `onChange: { ... }` / `onSubmit: { ... }` | 脚本 | 无 | 优先于处理器 |

状态：`empty`/`blank`、`filled`/`has-text`、`editing`，以及通用状态。

## Dropdown

下拉选择框。点击展开，选择后收起并经 `change` 报告索引。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `options` / `items` / `choices` | 逗号分隔文本 | 空 | 选项；选项内不能含逗号 |
| `selected` / `index` / `value` | 数字 | 0 | 当前索引（单一标识符绑定可回写） |
| `item-height` | 数字 | 20 | 列表行高 |
| `item-padding` | 数字 | 8 | 行内水平内边距 |
| `font-size` | 数字 | .9 | 字号 |
| `accent` | 颜色 | 绿色 | 同时设置选中背景 |
| `selection-color` | 颜色 | 半透明绿 | 选中行背景 |
| `arrow` | 一个或两个字形 | `v` | 收起状态、展开状态 |
| `arrow-closed` / `arrow-open` | 文本 | `v` | 分别设置箭头 |
| `arrow-color` | 颜色 | 浅灰 | 箭头颜色 |
| `list-background` | 颜色 | 深色 | 展开列表背景 |
| `open` / `expanded` | 布尔 | `false` | 初始展开状态 |
| `onChange` | 处理器名 | 无 | 收到索引 |
| `onChange: { ... }` | 脚本 | 无 | 优先于处理器 |

- 默认 `z-order: 100`，保证展开列表盖住后面的兄弟节点。
- 展开时命中区域包含列表区域。
- 点击别处或失焦会收起。
- 状态：`expanded`/`open`、`collapsed`/`closed`。

## ScrollView

竖向裁剪滚动视口。子控件进入内部内容容器（竖向排列），视口外的内容既不绘制也不可点击。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `scroll-step` / `wheel-step` | 数字 | 24 | 每格滚轮距离 |
| `smooth-scroll` | 布尔 | `true` | 是否缓动滚动 |
| `scroll-smoothing` / `scroll-easing` | 数字 | 22 | 缓动速率 |
| `show-scrollbar` / `scrollbar` | 布尔 | `true` | 显示滚动条 |
| `scrollbar-width` | 数字 | 3 | 滚动条宽度 |
| `scrollbar-inset` | 数字 | 3 | 距右边缘距离 |
| `scrollbar-radius` | 数字 | 0 | 滚动条圆角 |
| `scrollbar-color` | 颜色 | 10% 白 | 轨道颜色 |
| `scrollbar-thumb-color` | 颜色 | 38% 白 | 滑块颜色 |
| `offset` / `scroll-y` | 数字 | 0 | 初始滚动偏移 |
| `padding` | 数字 | 0 | 视口内边距 |

- `layout` 属性会被接受但忽略：内容始终竖向排列。
- `scrollOffsetY` 缓动逼近 `scrollTargetY`；滚轮和滚动条只改目标值。
- 滚动条可抓取拖动。
- 内容高度由子控件累计，不需要手动设置。

```sil
ScrollView {
    width: 240;
    height: 180;
    scroll-step: 32;

    ForEach (data: @items, item: @item) {
        Label { text: @item.name; height: 24; }
    }
}
```

## ForEach

数组重复器，详见[列表与循环](dsl/lists.md)。

| 参数 | 必需 | 说明 |
| --- | --- | --- |
| `data` | 是 | 产出数组的表达式 |
| `item` | 否 | 行元素变量名，默认 `item` |
| `index` | 否 | 行索引变量名，默认 `index` |

| 属性 | 默认 | 说明 |
| --- | --- | --- |
| `layout` | `vertical` | 行的排列方式 |
| `height` | 0 | 0 表示按行累计；显式值优先 |
| 其它容器属性 | | 作用于重复器本身 |

- 单次最多 512 行，超出部分丢弃并记录日志。
- 数据变化后在下一次 `Update` 重建全部行。

## Part

挂在任意控件上的装饰层，用结构而不是图片来实现悬停、选中、开关等皮肤。

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `anchor` / `attach` | 见下表 | `fill` | 相对宿主的位置 |
| `inset` | 1–4 个数字 | 0 | 内缩，四边/上下左右 |
| `inset-x` / `inset-y` | 数字 | 0 | 水平/垂直内缩 |
| `inset-top` / `inset-right` / `inset-bottom` / `inset-left` | 数字 | 0 | 单边内缩 |
| `offset-x` / `offset-y` | 数字 | 0 | 位置微调 |
| `layer` | `under`/`over` | `over` | 画在宿主之下或之上 |
| `when` / `if` | 条件串 | 空 | 空格分隔，`!` 取反 |
| `src` | 文本 | 空 | 贴图 |
| `tint` | 颜色 | 白 | 贴图着色 |
| `text` | 文本 | 空 | 居中文字 |
| `font-size` | 数字 | 1 | 文字字号 |
| `width` / `height` | 数字 | 跟随锚点 | 非 `fill` 时为实际尺寸 |
| `visible` / `show` / `enabled` | 布尔 | `true` | 静态启用/禁用 |
| 通用外观属性 | | | `background`、`radius`、`border-*` 等 |

锚点值：

| 值 | 位置 |
| --- | --- |
| `fill` / `stretch` | 铺满宿主内框，忽略宽高 |
| `top` | 顶部，默认铺满宽度 |
| `bottom` | 底部，默认铺满宽度 |
| `left` | 左侧，垂直居中 |
| `right` | 右侧，垂直居中 |
| `center` | 居中 |
| `top-left` / `top-right` / `bottom-left` / `bottom-right` | 四个角 |

`when` 可使用的宿主状态：

| 宿主 | 状态 |
| --- | --- |
| 全部 | `hovered`/`hover`、`pressed`/`active`、`focused`/`focus`、`visible`、`disabled` |
| `Checkbox` | `checked`/`on`/`selected`、`unchecked`/`off` |
| `Dropdown` | `expanded`/`open`、`collapsed`/`closed` |
| `TextBox` | `empty`/`blank`、`filled`/`has-text`、`editing` |

```sil
Checkbox {
    text: "夜间模式";
    box-size: 40;
    show-box: false;

    Part { anchor: left; width: 40; height: 20; radius: 10; background: #33363A; when: "!checked"; }
    Part { anchor: left; width: 40; height: 20; radius: 10; background: #498FC3; when: "checked"; }
}
```

Part 不接收鼠标事件，悬停/按下状态跟随宿主。

## 事件 {#events}

| 事件键 | 触发 | 载荷 |
| --- | --- | --- |
| `onClick:` | 按钮点击 | `event.name = "click"` |
| `onChange:` | 值变化 | `event.name/value/text/checked/index` 按控件填充 |
| `onSubmit:` | 输入框回车 | `event.name = "submit"`，`event.text/value` 为文本 |

事件键必须带 `on` 前缀（`action` 是 `onClick` 的别名），大小写与连字符差异会被忽略：`onChange` = `on-change` = `ONCHANGE`。值为 `{ ... }` 时作为脚本运行；否则作为 C++ 命名处理器名注册。脚本存在时，C++ 处理器不再调用。

脚本语法、载荷字段和回写规则见 [事件脚本](dsl/scripts.md)。

## 下一步

- [响应式绑定](dsl/bindings.md)
- [事件脚本](dsl/scripts.md)
- [C++ 集成](cpp-guide.md)
- [示例](examples.md)
