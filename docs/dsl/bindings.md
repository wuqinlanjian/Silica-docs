# 响应式绑定

绑定让界面跟随数据变化：`.sil` 里把值写成以 `@` 开头的表达式，数据更新时控件自动重新求值，不需要在 C++ 里记住“哪个 Label 要改”。

## 基本写法

```sil
Label {
    text: @player_name;
    color: @online ? "#5CD65C" : "#87919A";
}

TextBox {
    text: @search_text;
}
```

解析规则很简单：**属性值以 `@` 开头，整段值就是表达式**。因此下面这些都是绑定：

```sil
text: @health;
text: @health > 0 ? "存活" : "死亡";
width: 120 + @padding * 2;
visible: @show_hint;
background: @theme == "dark" ? "#171B1F" : "#E8EAED";
```

表达式里的 `@` 只是名字的一部分；字符串里的 `@` 不会被当作绑定。

## 名字从哪里来

求值一个名字时按以下顺序查找：

1. **事件载荷**：`event.value`、`event.checked`、`event.text`、`event.index`、`event.name`，只在事件脚本执行期间存在。
2. **局部作用域**：`ForEach` 注入的 `item` / `index`，`Component` 注入的参数名。
3. **`DataStore`**：全局数据键。
4. **控件状态**：`hovered`、`pressed`、`focused`、`visible`、`disabled`，以及控件自己的状态。

```sil
Label {
    // health 来自 DataStore
    text: @health;

    // checked 是 Checkbox 自身状态，不是数据键
    color: @checked ? "#5CD65C" : "#C8CFD5";
}
```

!!! warning "同名遮蔽"
    如果 `DataStore` 里也存在 `checked`，它会遮住 Checkbox 的 `checked` 状态。数据键尽量使用业务化的名字（`sound_enabled`、`theme_dark`），避免与控件状态重名。

## 表达式语法

支持以下结构：

| 类别 | 写法 |
| --- | --- |
| 字面量 | 数字 `42`、`3.14`；字符串 `"文本"`；`true` / `false` / `null` |
| 名字 | `@health`、`@item.name`、`$local`、`event.value` |
| 字段路径 | `@item.name`、`@config.audio.volume`、`@tags.0` |
| 算术 | `+ - * / %` |
| 比较 | `== != > < >= <=` |
| 逻辑 | 一元 `!`；条件用三元 `? :` |
| 分组 | `( ... )` |

示例：

```sil
text: @count + " 个物品";
color: @health >= 80 ? "#5CD65C" : (@health > 0 ? "#E0A23A" : "#E05252");
visible: !@compact;
```

!!! note "没有 && 与 ||"
    表达式只提供一元 `!` 和三元运算。需要复合条件时使用嵌套三元：

    ```sil
    visible: @logged_in ? (@name != "" ? true : false) : false;
    ```

    复杂条件更适合放进事件脚本，或由 C++ 汇总成一个布尔数据键。

### 类型与转换

- 数值与字符串相加时，任一侧是字符串就按字符串拼接：`@count + " 个"`。
- 比较两侧都是字符串时按字典序；否则按数值比较。
- 真值判断：非零数字为真；非空且不是 `"false"` / `"0"` 的字符串为真；非空数组/对象为真；`null` 为假。
- 绑定结果写入属性时：数字按常规格式输出，布尔输出 `true` / `false`，`null` **不会覆盖**属性当前值（等待数据到达）。

### 字段路径与数组

`@item.name` 先解析 `item`，再依次取字段。数组用数字下标访问：

```sil
text: "{{@items.0.name}}";
text: @row.tags.1;
```

路径不存在时得到 `null`，不会抛出异常，也不会让界面崩溃。

## 模板 {#templates}

当值是由多段文本与表达式拼接而成时，用 `{{ ... }}`：

```sil
Label {
    text: "生命：{{@health}} / 100（{{@health > 0 ? "存活" : "死亡"}}）";
}
```

- `{{ }}` 内是完整表达式，可用所有运算符。
- 花括号外的文本原样保留。
- 每段表达式只解析一次，之后每次求值只做替换。
- 模板中引用的数据键变化时，整段文本重新拼接。

如果字面文本本身需要包含 `{{`，目前没有转义语法；可以改用脚本拼接，或把文本拆成多个 Label。

## 条件值

可以用三元表达式根据状态选择值：

```sil
Button {
    background: @selected ? "#2F6B37" : "#1B2025";
    border-color: @disabled ? "#3A4046" : "#3C5A44";
}

Panel {
    // 也可以直接在条件里写控件状态，前面的 ! 表示取反
    background: !hovered ? "#171B1F" : "#232A30";
}
```

这种写法适合颜色、透明度、文本等简单切换。状态变化后会重新应用属性；配置了 `transition` 的属性会平滑过渡（见[动画与过渡](animations.md)）。

!!! tip "状态条件与数据条件"
    以 `@` 开头的条件按绑定处理；以 `!` 开头的条件（如 `!hovered`）按动态规则处理。两者都会随状态变化刷新，正常使用无需区分。

## 两向绑定

多数绑定是只读的：数据决定控件显示。对于**用户直接编辑**的属性，绑定还可以把用户输入写回 `DataStore`：

| 控件 | 可写回的属性 | 写入值 |
| --- | --- | --- |
| `Checkbox` | `checked` | 布尔 |
| `Slider` | `value` | 数字 |
| `TextBox` | `text` | 字符串 |
| `Dropdown` | `selected` / `index` / `value` | 数字索引 |

```sil
Checkbox {
    text: "静音";
    checked: @muted;          // 用户点击会写入 muted
}

Slider {
    min: 0; max: 100;
    value: @volume;           // 拖动会写入 volume
}

TextBox {
    text: @player_name;       // 输入会写入 player_name
}
```

写回只发生在**用户操作**时；C++ 通过 `DataStore::Set` 修改数据不会反过来触发写回。

!!! warning "只有单一数据键能写回"
    写回要求绑定表达式就是一个名字，例如 `checked: @muted`。像 `checked: @muted && ...`（不支持）或 `text: @first + @last` 这样的计算绑定是只读的：用户输入仍会改变控件，但不会写回任何键。

## 刷新时机

| 变化来源 | 行为 |
| --- | --- |
| `DataStore::Set(key, value)` | 值真正改变时，立即通知读取了该键的控件，同步重新求值 |
| 控件状态（`hovered`、`checked` …） | 每个 `Tick` 检查一次，变化后重新求值 |
| 模板 | 与绑定相同：数据键变化或状态变化时重新拼接 |
| `visible: false` 的子树 | 仍保持数据订阅；重新显示时使用最新值 |

`DataStore` 的通知在写入线程上同步执行。宿主若从其他线程写数据，应自行保证与渲染线程的同步，避免在通知回调里同时操作控件树。

## 绑定的边界

- **属性必须存在**：`cannot bind property '...' on '...'` 表示控件没有该属性，或表达式无法解析。
- **表达式解析失败**：该绑定被跳过，控件保留之前的属性值；日志带行号与列号。
- **不要绑定尺寸做动画**：`width: @value` 会改变布局，但不会自动插值；需要平滑效果时使用 `transition` 支持的属性或滑块自身的 `smooth`。
- **避免每帧制造新数组**：`ForEach` 依赖数组整体替换来发现变化，不必也不能原地修改数组。

## 常见错误

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 绑定永远不更新 | 数据键从未 `Set`，且没有同名控件状态 | 确认 C++ 写入的是同一个键名 |
| `=` 报错 | 表达式里用了赋值号 | 比较改用 `==` |
| 文本显示 `"false"` | 把布尔直接绑给了文本 | 这是正常输出；需要中文时用三元 |
| 计算绑定不写回 | 绑定了表达式而不是单个键 | 改为 `属性: @key`，把计算放到脚本/C++ |
| 改了样式没反应 | 属性名拼写错误，或值类型不合法 | 查日志中的 `unknown property` / 绑定警告 |

---

[上一章：常用控件](widgets.md) · [下一章：事件脚本](scripts.md)
