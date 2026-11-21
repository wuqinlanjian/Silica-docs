# 事件脚本

事件脚本把简单的界面交互留在 `.sil` 里，不必为一个计数器再写 C++ 回调。它挂在事件属性上，用 `{ ... }` 书写，可以读写 `DataStore`、做算术、按条件分支。

## 基本写法

```sil
Button {
    text: "加血 +10";
    width: 96;
    height: 24;
    onClick: { @health += 10; };

    // 多语句用分号分隔
    // onClick: { $gain = 10; @health += $gain; if (@health > 100) { @health = 100; } };
}
```

脚本只在对应事件触发时执行：

| 控件 | 事件 | 触发时机 |
| --- | --- | --- |
| `Button` | `click` | 在按钮内按下并释放，或焦点激活 |
| `Checkbox` | `change` | 用户切换勾选 |
| `Slider` | `change` | 用户拖动或按键改变数值 |
| `TextBox` | `change` | 文本发生变化 |
| `TextBox` | `submit` | 回车提交 |
| `Dropdown` | `change` | 用户选择新项 |

事件属性名不区分大小写，`onClick` / `on-click` / `ONCLICK` 是同一个事件；`action` 是 `click` 的别名。

## 语句

脚本支持三类语句：

1. **赋值**：`@key = 表达式;`
2. **复合赋值**：`@key += 表达式;`，同样支持 `-=`、`*=`、`/=`。
3. **条件**：`if (表达式) { ... } else { ... }`，`else` 后可再接 `if`。

```sil
Checkbox {
    text: "启用主题";
    checked: @theme_on;
    onChange: {
        @theme_on = event.checked;
        if (event.checked) {
            @accent = "#5CD65C";
        } else {
            @accent = "#87919A";
        }
    };
}
```

语句之间用 `;` 分隔，最后一条可以省略分号。脚本内支持 `//` 与 `/* */` 注释。

## 赋值目标

| 写法 | 目标 | 生命周期 |
| --- | --- | --- |
| `@health = 100;` | `DataStore` 键 `health` | 全局，跨脚本、跨控件 |
| `health = 100;` | 同上（`@` 可省略） | 全局 |
| `$gain = 10;` | 本次执行的局部变量 | 只在本脚本这一次运行内有效 |

`$` 变量适合拆解计算步骤：

```sil
onClick: {
    $step = @shift_held ? 10 : 1;
    @health += $step;
    if (@health > 100) { @health = 100; }
};
```

`$` 变量不会写入 `DataStore`，也不会在两次事件之间保留。

!!! warning "$变量与文档级 Var 不是一回事"
    `$accent` 在属性值里是编译期的 `Var` 替换；在脚本里是运行期的局部变量。两者可以同名，但互不影响。

## 读取值

脚本中的表达式与绑定使用同一套语法：

```sil
onClick: {
    $next = @mode + 1;
    if ($next > 2) { $next = 0; }
    @mode = $next;
};
```

可以读取当前控件状态（`checked`、`disabled` 等）、其他数据键，以及事件载荷。

## 事件载荷

事件发生时，脚本可以读取 `event`：

| 字段 | 类型 | 适用事件 | 含义 |
| --- | --- | --- | --- |
| `event.name` | 字符串 | 所有 | 事件名，如 `"click"`、`"change"` |
| `event.value` | 依控件 | 值事件 | 新值：布尔、数字或文本 |
| `event.text` | 字符串 | 文本类事件 | 相关文本（输入内容、选项文字） |
| `event.checked` | 布尔 | `change` | 勾选状态 |
| `event.index` | 数字 | `change`（Dropdown） | 选中索引 |

```sil
Checkbox {
    checked: @theme_on;
    onChange: { @theme_color = event.checked ? "#5CD65C" : "#E05252"; };
};

ForEach (data: @items, item: @item) {
    Button {
        text: "{{@item.name}}";
        onClick: { @selected = @item.id; };
    };
}
```

`event.*` 只在脚本执行期间有效；离开事件后再次求值会得到 `null`。

## 脚本与 C++ 回调的优先级

事件属性可以写脚本，也可以写回调名：

```sil
Button { onClick: "open_settings"; }        // 交给 C++
Button { onClick: { @page = "settings"; }; } // 脚本处理
```

当同一个事件既有脚本又有 C++ 回调时，**脚本优先，且不会再调用回调**。这让你可以先用脚本快速实现，之后再迁移到 C++。

值事件（`change` / `submit`）的顺序相同：脚本存在时，注册的同名 C++ 回调不会被调用。

## 脚本能做什么，不能做什么

**可以：**

- 读写 `DataStore` 的任意键；
- 使用算术、比较、三元表达式与 `if/else`；
- 读取局部作用域（`@item.name`、组件参数）与控件状态；
- 在一次执行中连续修改多个键；每个 `Set` 都会立即通知绑定刷新。

**不可以：**

- 调用 C++ 函数或执行任意系统操作；
- 定义函数、循环或递归；
- 修改控件属性（例如“把按钮变红”）。需要视觉变化时，把状态写进数据键，再用绑定/条件值驱动属性；
- 在脚本里使用 `$变量` 引用文档级 `Var`。

!!! note "把视觉变化交给绑定"
    脚本只改数据；界面如何响应由绑定决定。例如“点击后按钮变绿”应写成：

    ```sil
    Button {
        background: @active ? "#2F6B37" : "#1B2025";
        onClick: { @active = !@active; };
    }
    ```

## 写入与刷新

脚本中的 `@key = ...` 最终调用 `DataStore::Set`：

1. 值真正改变时，读取该键的绑定立即重新求值。
2. 同一事件内后续语句读取该键时，看到的是新值。
3. 控件绘制在下一帧反映新状态。

多个控件读取同一个键时都会收到通知，因此一个按钮的脚本可以同时更新多个显示位置。

## 复杂逻辑放哪里

| 场景 | 建议 |
| --- | --- |
| 计数、取反、简单状态切换 | 写在 `.sil` 脚本 |
| 需要访问游戏对象、文件、网络 | 注册 C++ 回调 |
| 多个界面共享的业务状态 | C++ 写入 `DataStore`，界面只绑定 |
| 需要循环、复杂校验 | C++ 处理，结果写回数据键 |

## 排错

| 报错 / 现象 | 原因 |
| --- | --- |
| `scripts attach to an event property (onClick, onChange, onSubmit, ...)` | 把 `{ ... }` 写在了非事件属性上 |
| `expected '(' after 'if'` | `if` 后缺少括号 |
| `expected '=' ... after '...'` | 赋值缺少运算符，或用了 `==` 当赋值 |
| `in 'xxx ...': use '==' to compare; '=' only assigns inside a script` | 表达式里写了单个 `=` |
| 脚本不执行 | 事件没有触发，或控件被 `disabled` |
| 改了数据但界面没变 | 界面读取的是另一个键，或该属性没有绑定 |

---

[上一章：响应式绑定](bindings.md) · [下一章：组件](components.md)
