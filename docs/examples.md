# 示例集锦

本页给出可直接改造的完整片段，覆盖常见界面形态：设置页、搜索列表、滑动开关、图片卡片、标签页与确认弹窗。每个示例都只用文档中列出的属性与语法；把片段放进 `.sil` 文件，由宿主设置对应的数据键即可显示。

!!! note "示例里的数据键"
    示例假定宿主已经在 `DataStore` 里设置了所需键，并在每帧驱动控件树。加载、渲染与输入接入见 [C++ 集成](cpp-guide.md)；只替换 `.sil` 文件时，用模组提供的重载入口（SilicaTest 为 `F5`）即可看到修改。

<!-- 此处插入加载 .sil 与挂载到渲染帧的接入代码片段 -->

初始化示例数据：

```cpp
silica::DataStore::Set("volume", silica::Variant{80});
silica::DataStore::Set("muted", silica::Variant{false});
silica::DataStore::Set("ui_scale_index", silica::Variant{1});
```

## 1. 设置面板

用 `Style` 统一卡片外观，用 `Component` 复用“标题 + 说明 + 控件”的行结构。复选框、滑块与下拉框都把用户操作写回数据键。

```sil
Var surface: #1B1E22;
Var text_main: #D5DBE1;
Var text_dim: #7C848C;

Style Card {
    background: $surface;
    radius: 6;
    border: 1 solid #34383D;
    shadow: 0 6 18 #00000066;
}

Style RowTitle {
    font-size: 0.72;
    color: $text_main;
}

Style RowHint {
    font-size: 0.62;
    color: $text_dim;
}

Component SettingRow(title, hint: "") {
    Container {
        layout: vertical;
        margin-top: 14;

        Label { text: @title; height: 16; class: RowTitle; }
        Label { text: @hint; height: 8; class: RowHint; }

        Container {
            layout: vertical;
            margin-top: 6;
            Slot
        }
    }
}

Container {
    name: settings_panel;
    layout: vertical;
    width: 380;
    padding: 16;
    class: Card;

    Label {
        text: "音频与界面";
        height: 24;
        font-size: 0.95;
        color: $text_main;
    }

    SettingRow(title: "主音量", hint: "拖动即时生效") {
        Slider {
            height: 18;
            min: 0;
            max: 100;
            step: 1;
            smooth: true;
            value: @volume;
            onChange: { @volume = event.value; };
        }
    }

    SettingRow(title: "静音", hint: "保留界面提示音") {
        Checkbox {
            text: "关闭所有游戏声音";
            height: 18;
            checked: @muted;
            onChange: { @muted = event.checked; };
        }
    }

    SettingRow(title: "界面缩放", hint: "修改后重新打开界面") {
        Dropdown {
            width: 140;
            height: 22;
            options: "小, 中, 大";
            selected: @ui_scale_index;
            onChange: { @ui_scale_index = event.index; };
        }
    }
}
```

| 数据键 | 类型 | 说明 |
| --- | --- | --- |
| `volume` | 数字 | 0–100，滑块拖动时由脚本写回 |
| `muted` | 布尔 | 复选框状态 |
| `ui_scale_index` | 数字 | 下拉框索引，从 0 开始 |

要点：

- `SettingRow` 的根容器是竖排，`width` 留 0 会被父容器拉满；不需要手写绝对坐标。
- 下拉框的 `options:` 是逗号分隔文本，选项自身不能带逗号。
- 三个控件的绑定都是单个标识符，因此用户操作能回写数据键；C++ 只需监听这些键。

## 2. 搜索框与结果列表

文本框负责输入，`ForEach` 负责行，`ScrollView` 负责裁剪与滚动。结果为空时显示提示。

```sil
Component ResultRow(name, meta) {
    Container {
        layout: horizontal;
        height: 34;
        padding: 6;
        margin-top: 4;
        radius: 4;
        background: #FFFFFF08;

        Label {
            text: @name;
            width: 200;
            height: 16;
            color: #D5DBE1;
        }
        Label {
            text: @meta;
            width: 72;
            height: 16;
            font-size: 0.7;
            color: #7C848C;
            text-align: right;
        }
    }
}

Container {
    layout: vertical;
    width: 320;
    padding: 12;
    background: #171B1F;
    border: 1 solid #30363C;

    TextBox {
        height: 24;
        text: @query;
        placeholder: "搜索物品…";
        onChange: { @query = event.text; };
        onSubmit: { @search_submitted = @search_submitted + 1; };
    }

    Label {
        text: "没有匹配结果";
        height: 18;
        margin-top: 10;
        font-size: 0.75;
        color: #6E767D;
        visible: @result_count == 0;
    }

    ScrollView {
        height: 220;
        margin-top: 8;
        scroll-step: 30;
        scrollbar-width: 3;
        scrollbar-inset: 3;
        scrollbar-thumb-color: #4A5158;

        ForEach (data: @results, item: @result) {
            ResultRow(name: @result.name, meta: @result.meta)
        }
    }
}
```

| 数据键 | 类型 | 说明 |
| --- | --- | --- |
| `query` | 字符串 | 输入内容，`onChange` 写回 |
| `search_submitted` | 数字 | 回车次数，供 C++ 触发查询 |
| `results` | 数组 | 每项含 `name`、`meta` 字段 |
| `result_count` | 数字 | 结果条数，用于空状态提示 |

要点：

- `ForEach` 的数据不是数组时按空列表处理，不需要在 `.sil` 里判断类型。
- `result_count` 是独立键：`ForEach` 本身不暴露行数，空状态由 C++ 或脚本另外维护。
- 行内 `Label` 在横排里必须显式写宽度；主轴不会因为 `width: 0` 自动拉伸。

## 3. 滑动开关 {#switch}

`Checkbox` 关闭内置方框，用 `Part` 画出轨道和滑块，并用 `when` 按 `checked` 状态切换。

```sil
Checkbox {
    text: "夜间模式";
    width: 220;
    height: 28;
    checked: @night;
    show-box: false;
    text-gap: 56;
    font-size: 0.8;
    onChange: { @night = event.checked; };

    Part {
        anchor: right;
        width: 42;
        height: 22;
        radius: 11;
        background: #33363A;
        when: "!checked";
        layer: under;
    }
    Part {
        anchor: right;
        width: 42;
        height: 22;
        radius: 11;
        background: #498FC3;
        when: "checked";
        layer: under;
    }
    Part {
        anchor: right;
        inset-right: 22;
        width: 18;
        height: 18;
        radius: 9;
        background: #87919A;
        when: "!checked";
        layer: under;
    }
    Part {
        anchor: right;
        inset-right: 2;
        width: 18;
        height: 18;
        radius: 9;
        background: #FFFFFF;
        when: "checked";
        layer: under;
    }
}
```

要点：

- `anchor: right` 的部件在宿主控件内垂直居中，`inset-right` 控制离右边的距离。
- `when` 条件按空格分隔多个状态，前缀 `!` 表示取反；可用状态包括 `checked`、`hovered`、`pressed`、`disabled`。
- `Part` 不接收鼠标事件，点击仍然由 `Checkbox` 处理。
- 滑块切换是瞬时的；需要滑动动画时可以再加两个 `Part` 并配合关键帧缩放，但不要依赖部件位置参与布局。

## 4. 图片卡片

用九宫格边框做自适应外框，内部横排图标与文字。边框图与图标都可以来自模组目录。

```sil
Container {
    width: 260;
    height: 86;
    padding: 12;
    layout: horizontal;
    border-image: ui/card.png 12;
    shadow: 0 6 16 #00000066;

    Image {
        src: "textures/items/diamond";
        width: 48;
        height: 48;
        margin-right: 12;
    }

    Container {
        layout: vertical;
        width: 160;

        Label { text: "钻石"; height: 20; font-size: 0.95; color: #DCE3E9; }
        Label { text: "库存 64"; height: 16; font-size: 0.75; color: #8C959D; }
        Label { text: "右键查看合成"; height: 14; font-size: 0.65; color: #6E767D; }
    }
}
```

渐变背景的卡片可以不用边框图：

```sil
Panel {
    width: 220;
    height: 72;
    radius: 8;
    background: linear-gradient(to bottom, #2B3036, #1A1D21);
    shadow: 0 4 14 #00000080;
}
```

要点：

- `border-image` 的切片单位是纹理像素；`ui/card.png` 必须位于 `SetRootPath` 指定的模组目录内。
- 磁盘图片会被复制进自动生成的资源包；首次加载时资源包可能尚未就绪，图片位置先显示占位框。
- `Image` 会把贴图铺满矩形，不保持比例；按素材宽高比写 `width` / `height`。

## 5. 状态徽标（关键帧）

`@keyframes` 只支持 `opacity` 与 `transform: scale`，适合做呼吸、强调提示。

```sil
@keyframes breathe {
    0%   { opacity: 1; }
    50%  { opacity: .5; }
    100% { opacity: 1; }
}

Container {
    layout: horizontal;
    width: 240;
    height: 28;
    padding: 6;
    background: #171B1F;
    radius: 6;

    Label {
        text: "●";
        width: 20;
        height: 20;
        font-size: 0.8;
        color: #6E767D;
        visible: @muted;
    }
    Label {
        text: "●";
        width: 20;
        height: 20;
        font-size: 0.8;
        color: #7AD97A;
        animation: breathe 1.6s infinite;
        visible: !@muted;
    }
    Label {
        text: @muted ? "已静音" : "声音开启";
        width: 120;
        height: 20;
        font-size: 0.75;
        color: #C8CFD5;
    }
    Button {
        text: "切换";
        width: 64;
        height: 20;
        margin-left: 8;
        font-size: 0.7;
        onClick: { @muted = !@muted; };
    }
}
```

要点：

- 关键帧名区分大小写，写错不会报错，只是不播放。
- 缩放只影响绘制，不改变命中区域；需要点击的目标不要依赖缩放后的视觉边界。
- 想让颜色变化平滑，用普通 `background` / `color` 绑定并声明 `transition`；状态色是瞬时的。

## 6. 标签页

标签头由 `ForEach` 生成，点击写入当前索引；内容区用比较表达式控制可见性。

```sil
Container {
    layout: vertical;
    width: 360;

    Container {
        layout: horizontal;
        height: 30;

        ForEach (data: @tabs, item: @tab_info, index: @i) {
            layout: horizontal;

            Button {
                text: @tab_info.name;
                width: 88;
                height: 28;
                margin-right: 4;
                font-size: 0.72;
                background: @tab == @i ? "#4B535A" : "#2A2F34";
                color: @tab == @i ? "#F0F3F5" : "#9AA3AA";
                onClick: { @tab = @i; };
            }
        }
    }

    Container {
        layout: vertical;
        width: 360;
        height: 140;
        margin-top: 8;
        padding: 12;
        background: #1B1E22;
        border: 1 solid #34383D;

        Label { text: "综合设置"; height: 18; visible: @tab == 0; }
        Label { text: "按键绑定"; height: 18; visible: @tab == 1; }
        Label { text: "关于"; height: 18; visible: @tab == 2; }
    }
}
```

| 数据键 | 类型 | 说明 |
| --- | --- | --- |
| `tabs` | 数组 | 每项含 `name` 字段 |
| `tab` | 数字 | 当前索引，默认 0 |

要点：

- `ForEach` 内部注入的 `index` 参数名由 `index:` 决定；这里叫 `@i`，脚本里也用 `@i`。
- `onClick: { @tab = @i; };` 能工作，因为脚本读取时会先在行作用域里找 `i`。
- 标签内容是固定三个节点；标签数量由数据决定时，内容区也改成 `ForEach`。
- 隐藏的页面不占布局空间；容器高度固定时，切换标签不会带动整体尺寸变化。

## 7. 确认弹窗

用绝对布局叠一层遮罩和对话框，`z-order` 保证对话框在遮罩之上。

```sil
Container {
    width: 420;
    height: 240;
    layout: absolute;
    visible: @confirm_open;

    Button {
        x: 0;
        y: 0;
        width: 420;
        height: 240;
        text: "";
        background: #00000099;
        hover-background: #00000099;
        pressed-background: #00000099;
        onClick: { @confirm_open = false; };
    }

    Container {
        x: 100;
        y: 50;
        width: 220;
        height: 140;
        z-order: 10;
        layout: vertical;
        padding: 14;
        background: #23272B;
        border: 1 solid #3A4046;
        radius: 8;
        shadow: 0 10 24 #00000099;

        Label {
            text: "删除这个存档？";
            height: 22;
            font-size: 0.9;
            color: #E8EDF1;
        }
        Label {
            text: "此操作无法撤销。";
            height: 16;
            margin-top: 4;
            font-size: 0.7;
            color: #8C959D;
        }

        Container {
            layout: horizontal;
            height: 26;
            margin-top: 12;

            Button {
                text: "取消";
                width: 88;
                height: 26;
                margin-right: 8;
                onClick: { @confirm_open = false; };
            }
            Button {
                text: "删除";
                width: 88;
                height: 26;
                background: #7A3434;
                onClick: {
                    @confirm_open = false;
                    @deleted = @deleted + 1;
                };
            }
        }
    }
}
```

| 数据键 | 类型 | 说明 |
| --- | --- | --- |
| `confirm_open` | 布尔 | 弹窗是否显示，C++ 可随时置位 |
| `deleted` | 数字 | 删除次数，供宿主响应 |

要点：

- 遮罩用一个铺满的 `Button` 而不是 `Panel`：`Panel` 不接收点击，无法实现点遮罩关闭。
- 遮罩与对话框是兄弟节点，`z-order: 10` 让对话框后绘制并优先命中。
- 根容器隐藏时，整棵子树都不渲染、不接收输入；宿主只需维护一个布尔键。
- 对话框高度 140 足以容纳两行文字与按钮行；内容增加时同步调高，竖排容器不会自动撑开固定高度的根。

## 组合建议

- **先组件后页面**：把重复的行、卡片、开关做成 `Component`，页面只负责排列。
- **键名稳定**：`@volume`、`@results` 这类键一旦确定，C++ 与 `.sil` 都围着它写，不要在两侧各起一套名字。
- **一个页面一个根**：把弹窗、提示条作为绝对布局里的兄弟节点，避免为每个浮层单独加载一份文档。
- **用日志验证**：改完 `.sil` 后先看有没有 `warning: unknown property`，再看效果；拼错的属性不会报错，只会被跳过。

---

[上一章：常见问题与故障排查](troubleshooting.md) · [回到首页](index.md)
