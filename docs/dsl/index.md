# DSL 使用教程

`.sil` 是 Silica 的界面描述语言。它接近 QML 的节点结构，又借用了 QSS 的属性写法：一个节点是一层界面，属性写成 `键: 值;`，以 `@` 开头的值是响应式绑定，以 `{ ... }` 作为值的是事件脚本。

本教程按主题拆分，建议按顺序阅读；已经熟悉某个主题时可以直接跳转。

## 章节导航

| 章节 | 内容 | 适合 |
| --- | --- | --- |
| [基础语法](basics.md) | 节点、属性、值类型、颜色与数字、注释、顶层定义 | 所有人 |
| [布局](layout.md) | 垂直/水平/绝对布局、尺寸与边距、内边距、层叠顺序 | 所有人 |
| [常用控件](widgets.md) | 文本、按钮、勾选、滑动条、输入框、下拉框、滚动视图的用法 | 所有人 |
| [响应式绑定](bindings.md) | `@key`、表达式、模板、双向绑定、控件状态 | 所有人 |
| [事件脚本](scripts.md) | `onClick: { ... }`、赋值、`if/else`、`event` 载荷 | 所有人 |
| [组件](components.md) | `Component`、参数默认值、`Slot`、实例覆盖 | 界面作者 |
| [样式](styles.md) | `Var`、`Style`、`extends`、`class`、内联覆盖 | 界面作者 |
| [动画与过渡](animations.md) | `transition`、`@keyframes`、缓动、可动画属性 | 界面作者 |
| [列表](lists.md) | `ForEach`、行作用域、增删改数据、性能边界 | 界面作者 |
| [图片](images.md) | 游戏纹理名、`file:` 图片、资源包、失败表现 | 所有人 |

## 书写约定

- 文件使用 UTF-8 编码，扩展名 `.sil`。
- 节点类型区分大小写，使用内置类型名（`Container`、`Label`、`Button` …）。
- 属性名使用**小写加连字符**：`font-size`、`margin-left`、`z-order`。大小写与下划线不会自动换算成连字符。
- 事件属性允许 `onClick`、`on-click`、`ONCLICK` 等写法，内部会统一成同一个事件名。
- 样式名与 `class` 名不区分大小写，并忽略 `-` 与 `_`：`DemoRowBox`、`demo-row-box`、`demo_row_box` 指向同一个样式。
- 字符串使用双引号。文本包含空格、冒号或特殊字符时务必加引号。
- 颜色支持 `#RGB`、`#RGBA`、`#RRGGBB`、`#RRGGBBAA` 与少量颜色名（`white`、`black`、`red`、`green`、`blue`、`gray` 等）。
- 数字是 GUI 单位，可带 `px`、`em`、`pt`、`dp` 后缀，后缀不改变数值含义，只便于书写。
- 每条属性以 `;` 结尾。最后一条属性可以省略分号，但统一加上更不容易出错。

!!! note "属性值与源码文本"
    绑定与脚本保留原始表达式文本：字符串的引号、运算符两侧的空格都会原样交给求值器。因此 `text: @health > 0 ? "存活" : "死亡";` 与 `text:@health>0?"存活":"死亡";` 等价。

## 一个完整的最小文档

```sil
Var accent: #5CD65C;

Container {
    name: demo_panel;
    layout: vertical;
    width: 300;
    height: 150;
    padding: 12;
    background: #0F1215F2;
    border: 1 solid #353A40;

    Label {
        text: "生命值：{{@health}}";
        height: 16;
        font-size: 0.8;
        color: $accent;
    }

    Slider {
        height: 20;
        margin-top: 8;
        min: 0;
        max: 100;
        value: @health;
        onChange: { @health = event.value; };
    }

    Button {
        text: "重置";
        width: 80;
        height: 22;
        margin-top: 10;
        onClick: { @health = 100; };
    }
}
```

这段文档同时用到了 `Var`、绑定、模板、事件脚本与控件状态。各部分的细节见后续章节。

## 学习建议

- **先结构，后数据**：先用普通的字面值把界面摆出来，确认布局正确，再把需要随数据变化的值改成 `@绑定`。
- **命名稳定**：给需要 C++ 查找或测试的节点加 `name`，例如 `name: settings_panel;`。名字在整棵树内应唯一。
- **先看日志**：属性拼错只会产生警告，控件会保留默认值；编译错误会在日志中指明行与列。遇到“没生效”先查日志。
- **小步重载**：宿主支持重载时（示例为 `F5`），一次只改一小块，避免把多个问题混在一起。

---

[上一章：快速开始](../getting-started.md) · [下一章：基础语法](basics.md)
