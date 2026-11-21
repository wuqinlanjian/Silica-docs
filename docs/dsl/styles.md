# 全局样式

`Style` 把一组属性起个名字，让多个节点共享同一套外观。它只有属性，没有子节点，也不能运行事件脚本，因此可以安全地当作“外观模板”。

## 定义与使用

```sil
Style Surface {
    background: #2C2F33;
    radius: 6;
    padding: 10;
}

Style DangerButton extends Surface {
    background: #8A3B3B;
    color: #FFE4E4;
    hover-background: #9E4646;
    pressed-background: #743131;
}
```

```sil
Container {
    class: Surface;
    height: 60;
    Label { text: "普通面板"; }
}

Button {
    text: "删除";
    class: DangerButton;
    height: 32;
}
```

`class:` 与 `style:` 是同义指令，值都可以写一个或多个样式名，用空格或逗号分隔：

```sil
Button { class: DangerButton, Compact; text: "删除"; }
Button { style: DangerButton Compact; text: "删除"; }
```

多个样式按书写顺序叠加，后面的覆盖前面的：

```sil
// Compact 先应用，DangerButton 后应用并覆盖同名属性
Button { class: Compact DangerButton; text: "删除"; }
```

指令本身有固定顺序：`style` 列出的样式先应用，`class` 列出的后应用。混用两者时以 `class` 为准。

## 名称规范化

样式名在匹配时忽略大小写、连字符和下划线：

`DangerButton`、`danger-button`、`danger_button` 指向同一个样式。

!!! warning "属性键不会自动转换"
    名称规范化只作用于样式名，以及“谁覆盖谁”的优先级判断。属性键本身仍必须是规范写法（小写、连字符），例如 `margin-left`。写成 `marginLeft` 不会被识别为间距属性。

## 继承链

`extends` 形成单继承链：

```sil
Style Base { padding: 8; radius: 4; color: #DDD; }
Style Panel extends Base { background: #26282B; }
Style Window extends Panel { border-width: 1; border-color: #3A3D41; }
```

应用 `Window` 时，属性按 `Base → Panel → Window` 的顺序叠加，越靠后优先级越高。

| 情形 | 结果 |
| --- | --- |
| 基样式不存在 | 错误 `unknown style 'X'` |
| 循环继承（直接或间接） | 错误 `style 'X' extends itself` |
| 继承链超过 32 层 | 错误 `styles nest too deeply` |
| 重复定义同名样式 | 警告 `duplicate style ...; the last one wins` |

## 与节点属性的优先级

同一节点上，最终优先级从低到高：

1. `style:` 中的样式（按书写顺序）
2. `class:` 中的样式（按书写顺序）
3. 节点内联属性

内联属性覆盖样式，且**跨类别**生效：内联绑定可以覆盖样式里的字面值，内联字面值也可以覆盖样式里的绑定。

```sil
Style Card { background: #2C2F33; }
Card { class: Card; background: @highlighted ? "#3A4A2C" : "#2C2F33"; }
```

这里 `background` 的样式值让位给内联绑定。

属性覆盖的比较会忽略连字符，因此 `margin-left` 与 `marginLeft` 被视为同一个键；但如上文所述，非规范键仍会在应用阶段报未知属性。

## 样式里的绑定与模板

样式值同样支持 `@` 绑定和 `{{ }}` 模板：

```sil
Var accent: "#498FC3";

Style AccentText {
    color: @accent;
}

Style Status {
    text: "{{@statusText}}";
}
```

- 绑定在节点构建时挂到控件上，之后随 `DataStore` 刷新。
- 无法在样式中写事件脚本；写了会报错 `styles cannot run scripts`。
- 样式值中的 `$变量` 在编译期展开。

## 状态外观

常用控件状态通过专用属性表达，而不是伪类选择器：

| 属性 | 作用 |
| --- | --- |
| `hover-background` | 悬停时的背景 |
| `pressed-background` | 按下时的背景 |
| `disabled-background` | 禁用时的背景 |
| `disabled-opacity` | 禁用时的整体不透明度 |

```sil
Style MenuButton {
    background: #33363A;
    color: #E6E6E6;
    hover-background: #40454A;
    pressed-background: #2A2D31;
    disabled-background: #2A2A2A;
    disabled-opacity: .5;
    radius: 4;
}
```

把状态色写成 `none` 可以清除之前样式给出的值，恢复默认行为：

```sil
Style FlatButton extends MenuButton {
    hover-background: none;
    pressed-background: none;
}
```

!!! note "状态色只覆盖背景"
    `hover-background`/`pressed-background`/`disabled-background` 只影响填充色。需要更复杂的悬停皮肤（描边、图标、滑块）请使用 [Part 装饰层](widgets.md#part)。

!!! warning "状态色是瞬时切换的"
    悬停、按下和禁用状态色直接替换填充色，不参与 `transition` 插值。想让颜色平滑变化，请用绑定或 `@state ? a : b` 驱动普通 `background`，详见[动画与过渡](animations.md)。

## 样式不向下继承

样式只作用于写了 `class:`/`style:` 的那个节点，不会自动传给子节点：

```sil
Container {
    class: Surface;
    Label { text: "文字颜色需要自己设置"; }   // 不继承 Surface 的 color
}
```

需要子节点也使用样式时，逐个标注，或把标题做成 `Component` 统一封装。

## 作用域与可见性

- 样式只在定义它的 `.sil` 文档内可见。
- 不能跨文件引用样式。
- 不支持后代选择器、伪类、媒体查询或条件样式块。
- 条件外观用绑定或 `@state ? a : b` 表达，见[响应式绑定](bindings.md)。

## 组织建议

- 先定义一套基础令牌样式（间距、圆角、字号），再定义业务样式继承它们。
- 颜色尽量走 `Var`，方便整体换肤：

```sil
Var bg: "#23262A";
Var fg: "#E4E6E8";
Var accent: "#498FC3";

Style Surface { background: $bg; color: $fg; radius: 6; }
Style Primary extends Surface { background: $accent; color: "#FFFFFF"; }
```

- 状态色与 `transition` 放在同一套样式里，避免某个按钮漏掉过渡动画。
- 不要为了少写一个属性制造过深的继承链；两三层足够。

下一步：[动画与过渡](animations.md) · [控件参考](../widgets.md)
