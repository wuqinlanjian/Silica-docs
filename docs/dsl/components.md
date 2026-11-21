# 组件

`Component` 用来把一个可复用的子树命名下来。它和 `Style` 的分工很明确：

| 结构 | 能存什么 | 用途 |
| --- | --- | --- |
| `Style` | 只有属性 | 共享外观 |
| `Component` | 节点、参数、插槽 | 共享整段结构 |

每个 `Component` 定义只在当前 `.sil` 文档内可见，实例化时会真正创建一份控件子树（不是复制文本），因此可以放心用参数和插槽定制。

## 定义组件

```sil
Component SettingRow(title, detail: "默认说明") {
    Container {
        layout: horizontal;
        height: 40;
        padding: 8;

        Container {
            layout: vertical;
            width: 220;        // 横排的主轴尺寸由子控件自己决定，需要显式宽度
            Label { text: @title; height: 18; }
            Label {
                text: @detail;
                height: 14;
                font-size: .8;
                color: #999;
            }
        }
    }
}
```

语法要点：

- `Component 名字(参数列表) { ... }`，名字之后的圆括号可省略。
- 参数写成 `name` 或 `name: 默认值`，用逗号分隔。
- 默认值是**原始文本**，可以引用 `$变量`，但不求值。
- 组件体必须以节点开头，不能直接写属性；属性属于体内节点。

!!! warning "组件名区分大小写"
    类型查找不区分大小写（`SettingRow` 与 `settingrow` 都能命中），但请保持文档内写法一致，方便阅读和搜索。

## 参数

在组件体内，参数以 `@参数名` 的形式引用：

```sil
Component Stat(label, value) {
    Container {
        layout: vertical;
        Label { text: @label; color: #AAA; }
        Label { text: @value; font-size: 1.4; }
    }
}
```

实例化时传入参数：

```sil
Stat(label: "在线人数", value: @playerCount);
```

参数文本会先展开 `$变量`，然后按值的形态分类：

| 传入写法 | 体内引用时的含义 |
| --- | --- |
| `@playerCount` | 表达式：在**调用方**作用域和 `DataStore` 中求值 |
| `"在线人数"` | 字面文本（绑定中保持带引号的形式） |
| `true` / `false` | 布尔字面量 |
| `null` / `none` | 空值 |
| `12` / `3.5` | 数字字面量 |
| 其他 | 纯文本 |

替换发生在表达式解析之前。对绑定、脚本和模板，替换出来的是“表达式文本”；对普通属性，替换出来的是“字面文本”。这正是 `text: @title` 在体内既是合法绑定、又能接字符串参数的原因。

!!! note "参数不是响应式变量"
    `@title` 在编译期被替换成调用方传入的表达式。如果调用方传的是 `@playerCount`，那么体内最终就是 `text: @playerCount`，仍然是普通的响应式绑定；如果传的是固定字符串，它就是常量。参数本身不会在运行时变化。

### 默认值与缺失参数

```sil
Component Badge(text, tone: "#498FC3") { ... }
Badge(text: "Beta");            // tone 使用默认值
```

- 省略有默认值的参数：使用默认值。
- 省略无默认值的参数：产生警告 `component 'X' is missing the argument 'Y'`，该参数展开为空。
- 传入未声明的参数：产生警告 `unknown argument 'Y' on component 'X'`，被忽略。

### `$变量` 与参数的区别

| 写法 | 展开时机 | 适用范围 |
| --- | --- | --- |
| `$color` | 编译期文本替换 | 全局 `Var`，属性、绑定、模板、参数、脚本 |
| `@param` | 组件实例化时文本替换 | 组件参数 |

两者可以叠加：调用方传 `@param`，组件体内再引用 `$变量`，最终都会落成具体文本。

## 插槽 `Slot`

组件体里的 `Slot` 标记调用方子节点的插入位置：

```sil
Component Card(title) {
    Container {
        layout: vertical;
        padding: 12;
        background: #2A2D31;
        radius: 6;

        Label { text: @title; height: 20; color: #E0E0E0; }
        Slot
    }
}
```

使用：

```sil
Card(title: "设置") {
    Label { text: "第一项"; }
    Button { text: "应用"; }
}
```

关键语义：

- `Slot` 插入的节点由**调用方**的作用域、参数和 `DataStore` 构建，不是组件内部作用域。
- 插槽子节点可以访问调用方自己的 `@param`，也可以使用全局绑定。
- 组件体内出现多个 `Slot` 时，每个 `Slot` 都会插入一份调用方子节点。通常只写一个。
- 组件体没有 `Slot` 却传了子节点，会警告 `component 'X' has no Slot; its children are ignored`，子节点被丢弃。
- `Slot` 可以嵌在组件体内部的容器里，插槽位置随容器布局走。

!!! tip "组合优于继承"
    组件内无法直接改写插槽子节点的结构。需要定制时，把差异做成参数或插槽内容，而不是让组件“猜测”调用方的意图。

## 实例覆盖

实例化节点上的属性会应用到组件的**根节点**上，覆盖组件定义里写的值：

```sil
Card(title: "警告") {
    width: 240;         // 覆盖 Card 根节点的宽度
    class: DangerCard;  // 覆盖根节点的样式来源
}
```

因此可以：

- 调整几何（`width`、`height`、`margin` 等）。
- 用 `class:` / `style:` 追加或覆盖外观。
- 挂绑定、模板和事件脚本。
- 用 `name:` 给根节点命名，便于 `FindByName` 查找。

可覆盖的是根节点；组件体内部的非根节点不受实例属性影响。

## 多个根节点

组件体可以包含多个顶层节点：

```sil
Component TitleBar(left, right) {
    Label { text: @left; }
    Label { text: @right; }
}
```

这时编译器会自动包一层竖向 `Container`，两个节点成为它的子节点，实例属性应用到这层容器上。如果需要横向或其他布局，请自己写一个根 `Container`：

```sil
Component TitleBar(left, right) {
    Container {
        layout: horizontal;
        height: 24;
        Label { text: @left; width: 180; }
        Label { text: @right; width: 60; text-align: right; }
    }
}
```

!!! warning "多根节点的隐含容器"
    自动容器使用默认的 `vertical` 布局、零内边距和零背景。涉及对齐或背景时，显式写根容器更可控。

## 递归与嵌套限制

| 情形 | 结果 |
| --- | --- |
| 组件直接或间接实例化自己 | 错误 `component 'X' instantiates itself recursively`，该实例被丢弃 |
| 组件嵌套超过 32 层 | 错误 `components nest too deeply (limit 32)` |
| 节点树深度超过限制 | 错误 `components nest too deeply` |

递归在构建前按组件名检测，因此运行时不会出现无限展开。深层组合请拆分文档或改用 `ForEach`。

## 组件与事件

事件脚本写在实例节点或体内节点上都可以：

```sil
Component ActionRow(label, key) {
    Button {
        text: @label;
        height: 32;
        onClick: { @selected = @key; }
    }
}
```

- 体内脚本里的 `$name` 仍然表示脚本局部变量，不会被当成全局 `Var` 或参数。
- 实例节点上的脚本可对根节点生效；但根节点若不是可交互控件（例如 `Container`），事件不会触发。需要响应点击时，把事件写在组件内的 `Button` 或 `Checkbox` 上，或把根节点设成可交互控件。

## 完整示例

```sil
Var rows: 0;

Component Counter(label, step: 1) {
    Container {
        layout: horizontal;
        height: 36;
        padding: 6;
        background: #2C2F33;
        radius: 4;

        Label { text: @label; width: 160; height: 24; }
        Label { text: "{{@rows}}"; width: 48; height: 24; color: #9FD3FF; }
        Button {
            text: "+";
            width: 32;
            height: 24;
            onClick: { @rows = @rows + step; }
        }
    }
}

Counter(label: "已处理", step: 2);
Counter(label: "已跳过");
```

这里 `step` 既可用默认值，也可由调用方覆盖；`@rows` 是全局绑定，所有实例共享同一份数据。

## 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 组件子节点没出现 | 组件体没有 `Slot` | 在期望位置加 `Slot` |
| 参数在体内没有替换 | 写成了 `$param` 或未加 `@` | 参数用 `@name` 引用 |
| 实例属性没生效 | 属性写到了实例的子节点上 | 实例属性只能覆盖根节点 |
| 多个根节点排版错乱 | 自动容器是竖向 | 显式写根 `Container` 设置 `layout` |
| 点击无响应 | 根节点不可交互 | 把事件放到按钮等控件上 |

下一步：[全局样式](styles.md) · [动画与过渡](animations.md)
