# 常见问题与故障排查

界面“没出来”“不刷新”“点不动”通常不是随机问题，而是四类原因之一：文档没编译通过、数据键对不上、布局算成了 0 尺寸、宿主没有把事件送进来。本页给出排查顺序、日志对照表和常见现象的处理方法。

## 排查顺序

1. **看日志**。Silica 的所有编译错误、警告与运行时问题都通过 `runtime::SetLogSink` 输出；先确认最近一次加载有没有报错。
2. **确认编译成功**。任何一条 `error:` 都会让 `CompileFile(filepath, fallback)` 放弃文件内容、改用回退界面。界面“长得像默认样”往往就是回退了。
3. **核对数据键**。绑定读不到键时得到 `null`，不会报错；写错名字只会表现为“值一直是默认值”。
4. **核对布局**。绝对布局里没有显式尺寸的节点就是 0×0；容器沿布局方向不会自动撑开子节点之外的空间。
5. **缩小到最小文档**。删到只剩出问题的那个节点，再逐个加回属性，通常能立刻定位是哪个值导致的。

!!! tip "先字面值，后绑定"
    先用常量把位置和尺寸摆对，再把值换成 `@绑定`。同时调布局和绑定，很难判断问题出在哪一层。

## 日志格式

一条诊断的格式是：

```text
来源:行:列: 级别: 消息
```

例如：

```text
D:/LeviLamina/mods/SilicaTest/ui/panel.sil:857:21: error: unknown style 'demo_row_box'
```

- `error:` 表示这次编译失败。使用回退加载时，整份文件都会被替换。
- `warning:` 表示某个属性或脚本被跳过，控件继续使用默认值，文档其余部分正常加载。
- 单次解析最多保留 200 条诊断，修复前面的错误后重新加载再看。

把日志接到自己的日志系统：

```cpp
silica::Silica::SetLogSink([](std::string_view line) {
    mymod::Log(line); // 写文件或控制台
});
```

未设置时，日志写到标准错误和 Win32 调试输出。

## 编译期错误

| 消息 | 原因 | 处理 |
| --- | --- | --- |
| `cannot open '<路径>'` | 文件不存在或路径解析错误 | 检查 `SetRootPath` 与相对路径；确认文件名大小写 |
| `the document does not define any component` | 文件里只有定义，没有任何界面节点 | 至少写一个顶级节点（`Container`、`Panel`…） |
| `unknown component 'X' (register it with WidgetFactory or define a Component)` | 节点类型拼错，或自定义控件未注册 | 对照[控件参考](widgets.md)的类型名；自定义控件先调用 `WidgetFactory::Register` |
| `unknown style 'X'` | `class:` / `style:` 引用了不存在的样式 | 检查样式名；名称不区分大小写并忽略 `-`/`_`，但仍需确实定义过 |
| `style 'X' extends itself` | 样式继承成环 | 去掉自引用，或改成另一个基础样式 |
| `'X' cannot contain 'Y'` | 把子控件放进了叶控件 | 只有 `Container`、`ScrollView`、`ForEach` 能装任意子控件；叶控件只接受 `Part` |
| `'Slot' can only be used inside a Component definition` | 在组件定义之外写了 `Slot` | 把 `Slot` 移到 `Component` 的根节点内 |
| `component 'X' instantiates itself recursively` | 组件直接或间接引用自身 | 组件不能递归实例化，把递归部分改成 `ForEach` 或手写层级 |
| `components nest too deeply (limit 32)` / `(limit 64)` | 组件嵌套层数超过上限 | 拆分组件，减少一层套一层 |
| `styles nest too deeply` | `extends` 链过长 | 压缩继承层级 |
| `styles cannot run scripts; 'X' is only valid on a component` | 在 `Style` 里写了事件脚本 | 脚本只能写在界面节点或 `Component` 的节点上 |
| `'ForEach' needs a data parameter` | `ForEach` 缺少 `data:` | 写成 `ForEach (data: @rows, item: @row) { ... }` |
| `multiple component roots need a 'Container' component` | 组件有多个根节点，但 `Container` 未注册 | 注册内置控件或自定义 `Container`；正常情况下调用 `WidgetFactory::RegisterBuiltins()` |
| `missing ';' after property 'X'` | 属性值后缺少分号，且下一行被当成新属性 | 补齐 `;`；一条属性里出现空格分隔的多个值也会触发 |
| `missing '}' for 'X' opened at line N` | 节点块或脚本块没闭合 | 从报出的起始行开始找未闭合的 `{` |
| `unterminated string` / `unterminated block comment` | 字符串或 `/* */` 注释没有结束 | 补上引号或 `*/` |
| `expected ':' after parameter 'X'`、`expected ',' or ')' after parameter 'X'` | 参数列表书写错误 | 对照 `Component Name(a, b: 默认值)` 的格式 |
| `expected hex digits after '#'` | 颜色写成 `#` 后没有合法十六进制 | 使用 `#RGB`、`#RGBA`、`#RRGGBB`、`#RRGGBBAA` |
| `expected a name after '@'` / `'$'` | 单独写了 `@` 或 `$` | 补上键名；文本里要显示 `@` 时放进引号字符串 |

## 编译期警告

警告不会阻止加载，但相关属性或脚本不会生效。`cannot bind` 与 `cannot attach` 的具体原因会跟在冒号后。

| 消息 | 通常原因 | 处理 |
| --- | --- | --- |
| `unknown property 'X' on 'Y'` | 属性名拼错，或该属性不属于这个控件 | 对照[控件参考](widgets.md)的属性表；注意属性名必须是小写加连字符，`fontSize` 不会自动转换 |
| `cannot bind property 'X' on 'Y': ...` | 表达式解析失败，或该属性不支持绑定 | 简化表达式；确认是可用 `ApplyProperty` 设置的属性 |
| `cannot attach the script on 'X' of 'Y': ...` | 脚本语法错误，或事件名不是事件属性 | 检查 `{ ... }` 里的语句、括号与分号；脚本只能挂在 `onClick`、`onChange`、`onSubmit` 等事件上 |
| `cannot evaluate the argument 'X': ...` | 传给组件的参数表达式解析失败 | 修正参数表达式；组件会以空值实例化 |
| `component 'X' is missing the argument 'Y'` | 参数没有默认值，调用时又没传 | 补参数或给参数加默认值 |
| `unknown argument 'X' on component 'Y'` | 调用时传了组件未声明的参数 | 删除该参数或把它加进组件的参数列表 |
| `component 'X' has no Slot; its children are ignored` | 组件定义里没有 `Slot`，调用处却写了子节点 | 在组件根节点里加 `Slot`，或去掉调用处的子节点 |
| `component 'X' has the same name as a builtin widget; the builtin is used` | 组件名与内置控件重名 | 给组件换一个名字 |
| `duplicate property 'X'; the last value wins` | 同一节点重复写了同一属性 | 删掉旧的一条 |
| `duplicate variable 'X'; the last value wins` / `duplicate style 'X'` / `duplicate component 'X'` | 文档级定义重名 | 合并或重命名 |
| `variable '$X' refers to itself` | `Var` 的值直接或间接引用自身 | 改成常量或拆成两个变量 |

!!! warning "警告也是线索"
    界面“有一部分完全没生效”时，先搜日志里的 `warning:`。属性拼错不会让整份文档失败，只会让那一行被悄悄跳过。

## 运行时日志

这些不是编译诊断，只在加载或绘制过程中出现。

| 日志 | 含义 | 处理 |
| --- | --- | --- |
| `silica: '<文件>' is unavailable, compiling the fallback source` | 文件有错误或读不到，已改用回退源码 | 往上翻同一批 `error:`，修好后重载 |
| `silica: reloaded '<绝对路径>'` | 重载成功 | 无需处理 |
| `silica: reload of '<路径>' failed; keeping the previous tree` | 重载后的文档有错误，旧界面被保留 | 按日志修文档，再次重载 |
| `silica: ReloadUI('<文件>') before CompileFile; nothing to replace` | 没加载过就调用了重载 | 先 `CompileFile` 再 `ReloadUI`，或改用 `CompileFile` |
| `silica: ForEach truncated N rows to 512 (the data array is too large)` | 数组超过 512 项，只渲染前 512 行 | 分页、过滤，或不要一次性塞入超大数组 |
| `silica: image texture not available: <src>` | 贴图名不存在，或资源包尚未就绪 | 检查路径拼写与资源包；见[图片](dsl/images.md) |
| `silica: packed image '<路径>' as <贴图名>` | 磁盘图片已导入资源包 | 正常信息，可用于确认路径解析到了哪里 |
| `silica: image import needs SetRootPath (the mod directory)` | 没有根目录，磁盘图片无法导入 | 启动时调用 `SetRootPath` |
| `silica: refusing image outside the mod directory: <路径>` | 图片绝对路径落在根目录之外 | 把资源放进模组目录并用相对路径 |
| `silica: cannot write the image pack manifest in <目录>` / `cannot copy image into the pack: ...` | 资源包目录不可写 | 检查模组目录权限或文件占用 |
| `silica: texture api unavailable (ResourceLocation/TexturePtr symbols not resolved)` | 当前游戏版本的贴图接口未解析到 | 确认框架版本与游戏版本匹配；此时图片只能显示占位框 |

## 按现象排查

### 界面完全不显示

| 可能原因 | 判断方法 | 处理 |
| --- | --- | --- |
| 编译失败并回退 | 日志里有 `error:` 和 `is unavailable` | 先修错误 |
| 根节点返回空 | `CompileFile` 返回 `nullptr` | 检查回退源码本身是否有效；`CompileFile` 的日志会指出原因 |
| 宿主没有绘制控件树 | 其他界面也不显示 | 确认每帧调用了 `root->Render(context)` 和 `FlushDraws(context)` |
| 根节点尺寸为 0 | 布局计算后宽或高为 0 | 给顶层容器显式 `width` / `height` |
| 根节点位于屏幕外 | 日志打印 `x/y/width/height` 或临时加描边 | 检查坐标是否是 GUI 单位，而不是像素 |
| 被透明度/可见性隐藏 | `visible: false`、`opacity: 0` 或绑定值为假 | 临时改成字面值验证 |

### 控件不显示或位置不对

- **叶控件里的子节点被拒绝**：日志出现 `'X' cannot contain 'Y'`。只有容器类能装任意子控件。
- **绝对布局没有尺寸**：`Container` 默认 `absolute`，子节点不会自动排布；必须写 `x` / `y` / `width` / `height`，或给容器换 `layout: vertical|horizontal`。
- **交叉轴写 0 才填充**：竖排里 `width: 0` 表示撑满容器宽度；写死数值则保持该宽度。沿布局方向相反，容器会包住子节点内容，子节点不会自动拉高。
- **`margin` 只作用于尾部**：竖排里 `margin` 是底部间距，`margin-top` 才是顶部间距；`margin-left` 在竖排中不生效。
- **`padding` 只对内层布局有效**：叶控件的文字内缩用各自属性（如 `text-padding`），不是 `padding`。
- **被裁剪**：`ScrollView` 会裁剪超出视口的子节点；浮层应放在 `ScrollView` 之外，或留出足够空间。
- **被同层遮挡**：`z-order` 大的后绘制并在命中测试中优先；检查是否有覆盖全屏的容器挡在前面。

### 点击没有反应

1. **按下和释放必须在同一个控件内**。按下后拖出控件再松开，不算点击。
2. **禁用会传给整棵子树**。父容器 `disabled: true` 时，子按钮收不到任何输入。
3. **不可见即不参与命中**。`visible: false` 的节点不渲染也不响应。
4. **没有处理器**。`onClick` 脚本和 C++ 命名回调都没有注册时，点击只产生悬停/按下视觉变化。
5. **脚本会抑制同名 C++ 处理器**。同一事件既有脚本又有命名回调时，脚本先执行且 C++ 回调不再收到。
6. **上层节点截走命中**。高 `z-order` 的透明容器只要矩形覆盖，就会先拿到指针。

### 输入框无法输入

- `TextBox` 必须先获得焦点：点击它，或由宿主的焦点循环切过去。
- 宿主必须把按键转发给 `FocusManager` 当前控件；只转发鼠标不足以输入文字。
- 超过 `max-length`（默认 32）的输入会被截断，不是丢字。
- `disabled` 或任一祖先 `disabled` 时，焦点不会落入，也不会接收键盘。

### 滚轮不滚动

- 光标必须位于 `ScrollView` 的矩形内，宿主必须把滚轮事件交给 `PointerRouter::Wheel`。
- 内容没有超出视口时不会滚动，这是正常行为。
- 光标停在被禁用的行上时，滚轮会继续向上找外层滚动视图；如果外层也没有可滚动区域，就什么都不会发生。

### 绑定不刷新

| 可能原因 | 处理 |
| --- | --- |
| 数据键拼错 | 绑定读不到时得到 `null`，没有报错；用日志打印 `DataStore::Get` 的结果核对 |
| 直接修改了 `Variant` 内部的数组/对象 | 容器按句柄共享且没有修改接口；构造新容器后 `Set` |
| 数组元素原地改动 | 同上：重建整个数组再 `Set`，才能触发深度比较后的通知 |
| 双向绑定写在路径上 | 只有单个标识符（`@name`）能回写；`@item.name` 只读 |
| 宿主没有逐帧调用 `TickTree` / `Update` | 绑定的动态求值与延迟重建都发生在帧更新里 |
| `ForEach` 数据尚未设置 | 非数组（包括 `null`）时行数为 0，等数据到达后会自动重建 |

### 脚本不生效

- 日志里有 `cannot attach the script`：脚本解析失败，整段脚本被丢弃。
- 赋值目标只能是单个标识符：`@health`、`health`、`$local`；不能写 `@config.volume` 或数组下标。
- 脚本没有 `for`、`while`、函数定义；复杂逻辑请放到 C++。
- 除以 0 或产生非有限数的结果会被规整为 0，不会写入 `NaN` / `Infinity`。
- `$` 变量只在本次脚本执行期间存在，不会保留到下一次事件。

### 图片不显示

- 游戏贴图名与磁盘文件按后缀区分：`textures/ui/icon` 走游戏资源，`textures/ui/icon.png` 走模组目录。写错一种就会落到占位框。
- 磁盘图片只支持 `.png` 与 `.tga`，且必须位于 `SetRootPath` 指定的目录之内。
- 首次加载时资源包可能尚未就绪，控件会先画占位框并在后续帧重试；看到 `packed image` 日志说明导入成功。
- 图片会被拉伸到控件矩形，不做等比裁剪；比例不对时调整 `width` / `height`。

### 列表为空或行内容不变

- `ForEach` 的 `data:` 必须求值为数组；对象、字符串、`null` 都按空列表处理。
- 行模板里的 `item` / `index` 名字由参数决定，默认就是 `item` / `index`；改名后模板里也要用新名字。
- 数组变化后重建被安排在下一帧的 `Update`，同一帧内不会立刻看到新行。
- 行数上限 512，超出的部分会记录截断日志。

### 动画不动

- `@keyframes` 名称必须与 `animation:` 中写的完全一致（大小写敏感）；名字不存在时静默不播放。
- 关键帧只支持 `opacity` 与 `transform: scale` / `scale`，其他属性写了也不会插值。
- `transition` 只对 `background`、`color`、`border-color`、`opacity` 生效，并且只在此类属性因绑定或动态值变化时触发。
- 悬停、按下、禁用状态色是瞬时切换，不走 `transition`；需要平滑变化请用绑定驱动普通 `background`。
- 宿主必须以真实帧间隔调用 `TickTree(root, deltaTime)`，且间隔会被限制在合理范围内。

## 已知限制

- 表达式没有 `&&`、`||` 和函数调用；复合条件用嵌套三元或交给 C++ 汇总。
- 没有配置器之外的内建“列表差量更新”：`ForEach` 每次重建整段行，不做按键复用。
- 脚本不能写字段路径或数组元素，只能写单个标识符。
- `Component` 不能递归实例化；组件嵌套上限 32，文档节点嵌套上限 64。
- 只有统一的 `padding`，没有分方向的内边距。
- `Dropdown` 的 `options:` 以逗号分隔，选项文本自身不能包含逗号。
- `TextBox` 是单行输入，默认上限 32 个字符。
- 图片只接受 `.png` / `.tga`，且必须位于模组目录内。
- 不渲染 HTML/CSS/JavaScript，也不接管游戏原生菜单。

## 反馈问题时请附上

1. 日志中从 `error:` / `warning:` 开始的完整几行，以及文件、行、列。
2. 出问题节点的 `.sil` 片段（可以去掉无关属性）。
3. 宿主是每帧如何调用 `TickTree`、`Update`、`Render`、`FlushDraws` 的。
4. 框架与游戏版本，以及是首次加载还是 `ReloadUI` 后出现。

---

[上一章：C++ 集成](cpp-guide.md) · [下一章：示例集锦](examples.md)
