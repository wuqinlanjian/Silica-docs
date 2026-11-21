# 布局

Silica 的布局由容器计算：子控件不需要知道自己的绝对坐标，容器在每帧的 `Update` 中把位置与尺寸写入子控件。理解三个概念就能摆好绝大多数界面：**布局方向**、**尺寸规则**、**边距与内边距**。

## 容器与方向

只有 `Container`（以及继承它的 `ForEach`、`ScrollView`）能给子控件排版。`Panel` 是纯色块，只能挂 `Part` 装饰，不能放子控件。

`Container` 的 `layout` 属性接受：

| 值 | 含义 |
| --- | --- |
| `vertical`、`column`、`v` | 从上到下排列，默认拉伸子控件宽度 |
| `horizontal`、`row`、`h` | 从左到右排列，默认拉伸子控件高度 |
| `absolute`、`none` | 不排版，子控件使用自己的 `x` / `y` |

!!! warning "默认是绝对布局"
    `Container` 的默认 `layout` 是 `absolute`。如果忘记写 `layout: vertical;`，子控件会全部堆在左上角。`ForEach` 是例外，它默认 `vertical`。

```sil
Container {
    layout: vertical;
    width: 280;
    height: 160;
    padding: 10;

    Label { text: "第一行"; height: 16; }
    Label { text: "第二行"; height: 16; }
}
```

## 尺寸规则

### 沿布局方向

- 子控件的**主轴尺寸**由自己决定：竖排看 `height`，横排看 `width`。
- 容器自身的**主轴尺寸**若为 `0`，会在排版结束后按内容总长度自动计算（含内边距与尾随边距）。

### 垂直于布局方向

- 子控件的**交叉轴尺寸**若为 `0`，会被拉伸到容器的内容宽度/高度。
- 容器自身的交叉轴尺寸不会自动计算，需要显式设置，或由它的父容器拉伸。

因此常见写法是：竖排容器写 `width: 0;` 让父容器决定宽度，写固定或 `0` 的 `height` 决定是否随内容增高。

```sil
// 占满父容器宽度，高度随内容
Container {
    layout: vertical;
    width: 0;
    height: 0;
    padding: 8;
}
```

### 控件自己的默认尺寸

- `Label` 的高度为 `0` 时：普通模式高度为 `14 * font-size`；开启 `wrap` 后默认高度为 3 行。
- `Button`、`Checkbox`、`Slider`、`TextBox`、`Dropdown` 在未指定高度时有各自的默认值。
- `Image` 没有内在尺寸；应显式给出 `width` / `height`，否则不可见。

## 内边距

`padding` 是容器在四周留出的内缩。内容区域的起点为 `(x + padding, y + padding)`，交叉轴拉伸也以内容区域为准：

```sil
Container {
    layout: vertical;
    width: 200;
    height: 100;
    padding: 12;
    background: #171B1F;

    Label { text: "距边框 12"; height: 16; }
}
```

`padding` 对所有子控件生效，且只在容器内计算，不会影响容器自身的 `width` / `height` 含义：`width: 200` 包含左右各 12 的内边距，内容宽为 176。

## 外边距

`margin` 系列控制子控件之间的间距：

| 属性 | 作用 |
| --- | --- |
| `margin` | 尾随方向的默认间距：竖排为下、横排为右 |
| `margin-top` | 竖排时放在子控件**之前**的间距 |
| `margin-left` | 横排时放在子控件**之前**的间距 |
| `margin-bottom` | 竖排时放在子控件**之后**的间距；未设置时使用 `margin` |
| `margin-right` | 横排时放在子控件**之后**的间距；未设置时使用 `margin` |

!!! note "margin 只作用于尾随方向"
    `margin: 8` 在竖排中只等价于 `margin-bottom: 8`，不会在上方增加间距。要让第一个子控件离容器顶部更远，用 `padding`，或给该子控件写 `margin-top: 8`。

多个相邻子控件的间距会相加：前一个的 `margin-bottom` 与后一个的 `margin-top` 都会占用空间，不做折叠。

```sil
Container {
    layout: vertical;
    width: 240;
    padding: 8;

    Label { text: "标题"; height: 16; margin-bottom: 6; }
    Label { text: "说明"; height: 12; font-size: 0.7; }
    Button { text: "确定"; width: 80; height: 22; margin-top: 10; }
}
```

## 隐藏与禁用

- `visible: false` 的控件不绘制、不参与布局、不占空间，也不能被点击。
- `disabled: true` 的控件仍参与布局与绘制，但忽略指针、滚轮与键盘输入；容器被禁用时整个子树都不可交互。禁用状态的默认变暗程度由 `disabled-opacity` 控制。
- 两者都可以绑定：`visible: @show_hint;`、`disabled: @locked;`。

## 层叠与点击顺序

`z-order`（别名 `z`、`layer`、`z-index`）决定同级控件的绘制顺序与点击优先级：

- 数值越大越后绘制，也越先接到点击。
- 数值相同时按插入顺序，后写的节点在上层。
- `Part` 的 `layer` 属性是另一种含义（装饰层在控件之上或之下），不要与 `z-order` 混用。

```sil
Container {
    layout: absolute;
    width: 200;
    height: 120;

    Panel { x: 10; y: 10; width: 120; height: 80; background: #2A3036; }

    Panel {
        x: 60; y: 40; width: 120; height: 80;
        background: #3C8446;
        z-order: 10; // 覆盖在前一个面板之上
    }
}
```

## 绝对布局

`layout: absolute` 下容器不移动子控件，子控件使用自己的 `x` / `y`。适合做悬浮按钮、角标、叠层卡片。

- `x` / `y` 是相对容器内容区域左上角的坐标（容器有 `padding` 时从内容区域算起）。
- 祖先使用竖排/横排时，子控件的 `x` / `y` 会被覆盖，因此绝对定位必须放在 `absolute` 容器里。
- 绝对布局的容器不会根据子控件计算自身尺寸，需要显式设置宽高。

## 不裁剪与裁剪

普通容器**不裁剪**子控件：超出容器的部分仍会绘制，也可能被点击。需要裁剪时使用 `ScrollView`，它只绘制并命中视口内的内容。

如果确实需要把内容限制在一个盒子里，优先重新设计尺寸；不要依赖普通容器的边界。

## 布局与动画

关键帧里的 `transform: scale(...)` 是纯渲染变换，不会改变布局与点击区域。它围绕控件中心缩放，适合做脉冲、强调等视觉效果；不要让需要点击的目标依赖缩放后的视觉边界。

平滑滑动条（`smooth`）与 `ScrollView` 的滚动同样只改变绘制内容，布局尺寸保持稳定。

## 排错清单

| 现象 | 常见原因 |
| --- | --- |
| 子控件全叠在左上角 | 容器没有设置 `layout`，默认是 `absolute` |
| 子控件宽度为 0 看不见 | 交叉轴没有父容器可拉伸；给子控件显式宽度 |
| 容器高度为 0 | 竖排容器 `height: 0` 本应自动增高；检查子控件是否都 `visible: false` |
| 第一个控件贴着顶边 | `margin` 不作用于前导方向；改用 `padding` 或 `margin-top` |
| 点击位置与视觉错位 | 宿主传入的不是 GUI 坐标，或没有按界面缩放换算 |
| 超出的内容仍能点到 | 普通容器不裁剪；改用 `ScrollView` |

---

[上一章：基础语法](basics.md) · [下一章：常用控件](widgets.md)
