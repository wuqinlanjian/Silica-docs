# 列表与循环

`ForEach` 用一个数组的数据重复构建一段子树。它适合好友列表、搜索结果、背包格子这类数量随数据变化的内容。

## 基本写法

```sil
ForEach (data: @friends, item: @friend, index: @i) {
    Container {
        layout: horizontal;
        height: 32;
        padding: 6;

        Image { src: "textures/ui/icon_steve"; width: 24; height: 24; }
        Label { text: @friend.name; width: 180; height: 24; }
        Label { text: "{{@i}}"; width: 32; height: 24; color: #888; }
    }
}
```

`friends` 是宿主写入 `DataStore` 的数组键（`DataStore::Set("friends", Variant::Array(...))`），与文档级 `Var` 无关；`Var` 只做编译期文本替换，不承载运行期数据。

语法：

```text
ForEach (data: 表达式, item: 行变量名, index: 索引变量名) { 模板节点... }
```

| 参数 | 必需 | 默认 |
| --- | --- | --- |
| `data` | 是 | 无；缺失会报错 |
| `item` | 否 | `item` |
| `index` | 否 | `index` |

变量名写 `@friend` 或 `friend` 都可以，`@` 只是标记作用域变量。

## 作用域

每一行都会创建一个子作用域，注入 `item` 和 `index`：

- 行内直接用 `@friend`、`@i` 读取。
- 名称解析顺序：**行作用域 → 外层行作用域 → `DataStore` → 控件状态**。
- 行变量可以遮蔽同名的全局键；嵌套 `ForEach` 内层变量再遮蔽外层。

`index` 从 0 开始，是数字。

## 数据来源

`data` 是普通表达式，支持路径：

```sil
ForEach (data: @profile.friends, item: @friend) { ... }
ForEach (data: @groups.0.members, item: @member) { ... }
```

规则：

- 表达式结果必须是数组；对象、字符串、数字和空值都视为空列表，不产生行。
- 表达式引用到的根键被订阅。`@profile` 更新时整个列表重建。
- 外层 `ForEach` 提供的行变量不会进入 `DataStore` 订阅，内层列表随外层重建级联更新。

## 重建时机

数据变化不会当场销毁行：

1. `DataStore` 写入触发 `OnDataChanged`，`ForEach` 只把自己标记为脏。
2. 下一次 `Update` 时才重建所有行。

这样，点击脚本在行内执行完毕后才可能销毁该行，避免脚本运行到一半控件被释放。代价是数据变化到画面更新之间有一帧延迟。

!!! warning "没有按键复用"
    每次重建都会销毁并重新创建所有行，不做 ID 对比。行内控件的临时状态（输入中的文字、焦点、滚动位置）会丢失。需要保留的编辑状态请放在 `DataStore` 中。

## 数量上限

单次重建最多生成 **512** 行，超出部分被丢弃，并在日志中记录一次：

```text
silica: ForEach truncated 900 rows to 512 (the data array is too large)
```

需要展示更大的数据集时，请先过滤或分页，而不是依赖 `ScrollView` 承载上千个控件。

## 布局

`ForEach` 本身是容器，默认 `vertical`，行按从上到下排列：

```sil
ForEach (data: @items, item: @it) {
    Label { text: @it.name; height: 20; }
}
```

- 在 `ForEach` 上写 `layout: horizontal` 可改成横向排列。
- 未显式设置 `height` 时，高度由行内容自动累计；写了 `height` 则以文档为准。
- 模板有多个顶层节点时，它们都会成为行的同级兄弟节点；需要一行内组合，请自己包一层容器。

### 放进滚动视图

行数较多时，把 `ForEach` 放进 `ScrollView`：

```sil
ScrollView {
    width: 260;
    height: 200;
    show-scrollbar: true;

    ForEach (data: @entries, item: @entry) {
        Label { text: @entry.title; height: 26; }
    }
}
```

`ScrollView` 会按内容高度滚动，并把不可见行裁剪掉。

## 行内交互

行内的按钮、复选框可以读写 `DataStore`：

```sil
ForEach (data: @friends, item: @friend, index: @i) {
    Checkbox {
        text: @friend.name;
        height: 28;
        checked: @friend.selected;        // 只读：路径绑定不回写
        onChange: { @selectedIndex = @i; }  // 脚本写全局键
    }
}
```

限制：

- 双向绑定的回写只支持单一标识符。`checked: @friend.selected` 是路径，只会单向读取；勾选状态不会写回数组元素。
- 脚本赋值目标也只能是单一标识符（如 `@selectedIndex`），不能写 `@list.0.done`。
- 要修改数组元素，请由 C++ 侧整体替换数组，让 `ForEach` 重建。

## 嵌套列表

```sil
ForEach (data: @groups, item: @group) {
    Container {
        layout: vertical;

        Label { text: @group.name; height: 24; }
        ForEach (data: @group.members, item: @member) {
            Label { text: @member.name; height: 20; color: #AAA; }
        }
    }
}
```

内层数据来自外层行变量，不订阅 `DataStore`；外层重建时内层一起重建。嵌套层数受组件嵌套深度限制。

## 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 一行也没有 | `data` 不是数组，或键名写错 | 确认 `DataStore` 里存的是数组 |
| 行内容不更新 | 只改了数组内部元素，没有重新 `Set` | 整体写入新数组 |
| 勾选状态没写回 | 绑定目标是路径 | 在脚本里写全局键，或由 C++ 处理 |
| 行数被截断 | 超过 512 行 | 过滤、分页或改用其他方案 |
| 滚动位置跳回顶部 | 重建导致内容高度变化 | 编辑状态和滚动位置放到外部管理 |

下一步：[图片](images.md) · [组件](components.md)
