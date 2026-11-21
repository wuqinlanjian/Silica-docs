# 基础语法

本篇说明 `.sil` 文件的组成、节点与属性的写法、值如何被分类，以及数字、颜色、字符串和注释的规则。

## 文档结构

一份文档由两类内容组成：

1. **顶层定义**：`Var`、`Style`、`Component`、`@keyframes`。它们不直接产生控件，只供后面的节点引用。
2. **界面节点**：一个或多个顶层控件。最常见的是一棵以 `Container` 为根的树。

```sil
// 顶层定义
Var accent: #5CD65C;

Style Card {
    background: #171B1F;
    border: 1 solid #2A3036;
    padding: 8;
}

// 界面节点
Container {
    name: root;
    layout: vertical;
    width: 360;
    height: 200;

    Container {
        class: "card";
        height: 60;
        Label { text: "样式与变量"; height: 16; color: $accent; }
    }
}
```

定义可以出现在界面节点之前或之后；引用在编译时解析，不要求先声明。文档至少要有一个顶层节点，否则编译报错 `the document does not define any component`。

如果写了多个顶层节点，Silica 会把它们依次放进一个垂直 `Container`，避免全部叠在 `(0, 0)`。需要精确控制时，建议始终只保留一个根节点。

## 节点

节点语法：

```text
类型 参数列表? { 属性; 子节点... }
```

- **类型**：内置控件名（`Container`、`Panel`、`Label`、`Button`、`Image`、`ScrollView`、`Checkbox`、`Slider`、`TextBox`、`Dropdown`、`ForEach`、`Part`），或文档中定义的 `Component` 名。
- **参数列表**：仅 `ForEach` 与 `Component` 使用，形如 `(data: @rows, item: @row)`。普通控件的参数列表没有意义。
- **属性**：`键: 值;`，写在花括号内。
- **子节点**：属性之后直接写下一个节点即可，不需要逗号。

```sil
Container {
    layout: horizontal;
    height: 28;

    Label { text: "名称"; width: 80; height: 16; margin-top: 6; }
    Button { text: "确定"; width: 72; height: 24; onClick: "confirm"; }
}
```

节点类型不是内置控件、也不是已定义组件时，编译产生错误 `unknown component '...'`，该节点不会创建。

## 属性与值的分类

同一行 `键: 值;` 会根据值的写法被归为四类，行为不同：

| 写法 | 类别 | 示例 | 行为 |
| --- | --- | --- | --- |
| 以 `@` 开头 | 响应式绑定 | `text: @health;` | 解析为表达式；`DataStore` 相关键变化时重新求值 |
| 包含 `{{ ... }}` | 模板 | `text: "生命：{{@health}}"` | 仅替换花括号内的表达式，其余文本原样保留 |
| 值是 `{ ... }` | 事件脚本 | `onClick: { @health = 100; }` | 绑定到事件名；触发时执行语句 |
| 其他 | 字面属性 | `width: 120;` | 直接交给控件的属性处理器 |

!!! warning "只有事件属性能挂脚本"
    脚本必须写在事件属性上，例如 `onClick`、`onChange`、`onSubmit`。写成 `text: { ... }` 会得到警告：`scripts attach to an event property (...)`. 脚本不支持 `$变量` 替换，`$` 在脚本内是本次执行的局部变量。

字面属性里的 `$name` 会在编译期被 `Var` 的值替换：

```sil
Var gap: 8;
Var accent: #5CD65C;

Container {
    padding: $gap;
    background: #171B1F;
    Label { text: "编译期替换"; color: $accent; }
}
```

## 字符串

- 使用双引号包围：`text: "保存并关闭";`
- 支持转义：`\"`、`\\`、`\n`、`\t`。
- 文本里的 `;`、`:`、`{` 等字符只有出现在引号外才会被当作语法符号。
- 需要在文本里显示双引号时写 `\"`。

```sil
Label {
    text: "提示：按 \"确定\" 继续";
    height: 16;
}
```

## 数字与尺寸

数字按 GUI 单位解释，可带 `px`、`em`、`pt`、`dp` 后缀，后缀只是书写方便，不改变数值：

```sil
width: 120;
height: 24px;
padding: 8dp;
font-size: 0.9;
```

负值在多数尺寸属性上无效（视为不合法，产生警告）。`margin-*` 的负值表示“回退到 `margin` 默认值”，而不是负边距。

`0` 有特殊含义，取决于属性：

- `width: 0` / `height: 0`：尺寸交给布局或控件自身决定。垂直于布局方向时为 0 会被父容器拉伸；沿布局方向为 0 的子控件按 0 计算，只有容器自身沿布局方向为 0 时才按内容自动计算；有默认尺寸的控件（`Button`、`Slider`…）回落到默认值（见[布局](layout.md)）。
- `x: 0` / `y: 0`：绝对坐标为零，只在绝对布局中有效。

## 颜色

支持以下写法：

| 形式 | 示例 | 说明 |
| --- | --- | --- |
| `#RGB` | `#5C5` | 每位扩展为两位 |
| `#RGBA` | `#5C58` | 末位为透明度 |
| `#RRGGBB` | `#5CD65C` | 常用形式 |
| `#RRGGBBAA` | `#0F1215F2` | 末两位为透明度，`F2` 接近不透明 |
| 颜色名 | `white`、`black`、`red`、`green`、`blue`、`yellow`、`orange`、`purple`、`pink`、`cyan`、`magenta`、`gray` / `grey` | 固定色调 |
| 透明 | `transparent`、`none` | 完全透明；`none` 也用于清除阴影/边框等可选效果 |

透明度与 `opacity` 属性相乘：一个 `#FFFFFF80` 的控件再设置 `opacity: 0.5`，最终绘制透明度约为 25%。

## 布尔值

`true` / `false` 之外，`1` / `0`、`yes` / `no`、`on` / `off` 也可作为布尔值使用，大小写不敏感。常见于 `visible`、`disabled`、`wrap`、`smooth` 等属性。

## 注释

支持两种注释：

```sil
// 单行注释

/*
   多行注释
   /* 不支持嵌套 */
*/
```

注释可以出现在顶层、节点内或属性之间，也可以写在事件脚本的语句之间。

## `name` 属性

`name`（也接受 `id`）给节点一个标识，便于 C++ 用 `FindByName` 查找，或在测试与日志中识别：

```sil
Container {
    name: settings_panel;
    // ...
}
```

名字不参与布局，也不会自动成为数据键。整棵树中建议保持唯一；重复不会报错，但查找只会返回第一个匹配项。

## 属性覆盖顺序

同一个属性可能来自多处，最终生效顺序为（后者覆盖前者）：

1. 控件默认值。
2. `Style` 基样式（`extends` 链最底层）。
3. 派生样式。
4. 节点内联属性 / 绑定 / 模板。

样式与绑定按“键”覆盖，且 `margin-left` 与 `marginLeft` 被视为同一个键来比较优先级。不过实际应用属性时仍应写标准形式 `margin-left`，否则控件不认识。

## 常见的语法错误

| 日志信息 | 原因 | 处理 |
| --- | --- | --- |
| `expected ':' after ...` | 属性名后缺少冒号 | 补 `: ` |
| `missing ';' after property '...'` | 下一行新属性前缺少分号 | 每条属性以 `;` 结束 |
| `expected '{' after ...` | 节点或定义后缺少花括号 | 补 `{ ... }` |
| `property '...' expects a value, not a block` | 在非事件属性上写了 `{ ... }` | 改成字面值或绑定 |
| `unknown component '...'` | 类型名拼错，或组件未定义 | 检查大小写与定义 |
| `unknown property '...' on '...'` | 控件没有这个属性 | 查[控件参考](../widgets.md) |

!!! note "错误与警告的区别"
    语法错误会让文档无法编译；未知属性、未知样式、缺少组件参数等只是警告，会保留其余界面继续构建。日志中以 `error:` / `warning:` 区分。

---

[上一章：DSL 教程导航](index.md) · [下一章：布局](layout.md)
