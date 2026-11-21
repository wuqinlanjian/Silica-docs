# 项目总览

本章从整体上说明 Silica 由哪些部分组成、一份 `.sil` 文档如何变成屏幕上的控件，以及文档中反复出现的关键术语。

## 它解决什么问题

在基岩版客户端模组里手写界面，通常意味着直接调用游戏的 UI 渲染接口：自己计算每个按钮的坐标、自己处理悬停与点击、自己维护“数据变了要重画哪一块”。界面越复杂，C++ 代码越难读，改一个间距都要重新编译。

Silica 把这件事拆成两层：

| 层 | 负责 | 典型修改者 |
| --- | --- | --- |
| 界面描述（`.sil`） | 结构、布局、文字、颜色、绑定、交互逻辑 | 界面作者 |
| 集成逻辑（C++） | 加载文档、提供数据、注册回调、接入渲染与输入 | 模组开发者 |

`.sil` 描述“界面长什么样、数据变化时哪里跟着变”；C++ 负责“界面在什么时候出现、数据从哪里来、游戏事件如何送进来”。两者通过**数据键**和**回调名称**解耦：界面里写 `text: @player_name` 与 `onClick: "open_settings"`，C++ 只负责写入 `player_name`、注册名为 `open_settings` 的函数。

## 分层架构

```text
ui/main.sil
    │  语法解析
    ▼
文档模型：节点、Var、Style、Component、@keyframes
    │  样式展开、$变量替换、组件实例化、绑定/脚本解析
    ▼
控件树：SilicaWidget（Container / Label / Button / ...）
    │  宿主每帧驱动
    ▼
Tick（动画） → Update（布局） → 输入路由 → Render → FlushDraws
    │
    ▼
MinecraftUIRenderContext（游戏的 UI 绘制接口）
```

各层的职责：

- **文档模型**：保留每个节点的类型、参数、属性文本、子节点与源码位置。属性值在这一层仍是文本，只有具体控件知道 `font-size` 或 `layout` 的含义。
- **编译**：把 `Style` 与 `class` 展开成属性，替换 `$变量`，实例化 `Component`，把 `@key` 值登记为绑定、把 `{{ ... }}` 登记为模板、把 `{ ... }` 登记为事件脚本。未知属性记为警告，未知控件记为错误，整份文档不会因一个拼写错误而崩溃。
- **控件树**：每个控件持有自己的几何、样式、绑定与脚本。布局由容器计算；控件本身只负责绘制自己并响应输入。
- **宿主**：Silica 不主动挂钩游戏。模组在合适的渲染事件里调用 `TickTree`、`Update`、`PointerRouter`、`Render` 与 `FlushDraws`，并把键鼠事件换算成 GUI 坐标后送入路由器。

!!! note "Silica 不接管游戏界面"
    Silica 绘制到模组提供的渲染上下文里。它不替换 OreUI / JSON-UI，也不修改游戏自带的界面文件。示例模组 SilicaTest 会临时压入一个空的原生界面来获得光标与焦点，这只是宿主的一种接入方式，不是框架的硬性要求。

## 一次加载发生了什么

以 `Silica::CompileFile(path, fallback)` 为例：

1. **解析**：读取文件，生成节点树与文档级定义（`Var`、`Style`、`Component`、`@keyframes`）。语法错误带行号、列号进入诊断列表。
2. **展开**：对每个节点应用 `class` / `style` 指定的样式链（基样式在前、派生样式在后、内联属性最后覆盖），再把 `$变量` 替换成文本。
3. **构建**：按节点类型创建控件；设置属性；登记绑定、模板与脚本；递归构建子节点。`ForEach` 记录模板，稍后按数组元素重建；`Component` 用实参建立子作用域后构建函数体。
4. **打包图片**：`imagepack::Warmup(root)` 扫描树中的 `file:` 图片与 `Part` 贴图，把它们复制到模组目录下的 `resource_packs/silica_images`，让游戏在构建资源包栈时就能看到。
5. **挂载**：宿主保存根控件，设置位置与尺寸，调用一次 `Update()` 完成首帧布局。

`Silica::CompileFile` 会把编译结果记录在内部注册表中，因此之后可以对同一路径调用 `Silica::ReloadUI(path)`：

- 文件重新编译成功：替换为新树。
- 文件读取或编译失败：记录诊断，**保留上一棵树**，界面不会消失。

如果直接使用 `UICompiler::CompileFile(path, fallback)`，主文档失败时会编译内嵌的 `fallback` 文档，保证模组至少有一个可用界面。

## 每帧需要做什么

宿主每帧的顺序通常如下（SilicaTest 即按此顺序）：

| 步骤 | 调用 | 作用 |
| --- | --- | --- |
| 1 | `TickTree(root, deltaTime)` | 推进关键帧、过渡、条件值重算 |
| 2 | `root->Update()` | 重新布局，写入子控件的绝对坐标 |
| 3 | `PointerRouter::Update` / `Wheel` | 悬停、按下、释放、滚轮与拖拽 |
| 4 | `root->Render(context)` | 依次绘制背景、边框、文字、子控件、装饰层 |
| 5 | `FlushDraws(context)` | 提交本帧累计的填充与文字批次 |

键盘输入交给当前焦点控件：`FocusManager::Get().Current()`，再调用它的 `OnKeyDown(key)`。焦点环 `FocusRing` 负责 Tab/方向键顺序与回车激活，是否使用由宿主决定。

!!! tip "先 Tick 再布局"
    关键帧的缩放是纯渲染变换，但过渡与条件值会改变颜色、透明度等属性。先 `Tick` 再 `Update`，可以保证同一帧内布局看到的是最新状态。

## 核心概念

| 概念 | 说明 | 详见 |
| --- | --- | --- |
| 界面文档 | 一个 `.sil` 文件；顶层可含定义，之后是一个或多个界面节点 | [基础语法](dsl/basics.md) |
| 节点 / 控件 | `Button { ... }` 这样的一层界面；节点类型对应一个控件实现 | [控件参考](widgets.md) |
| 属性 | `键: 值;`，例如 `width: 120;`、`background: #202428;` | [基础语法](dsl/basics.md) |
| `Var` | 文档内的编译期常量，用 `$name` 引用 | [样式](dsl/styles.md) |
| `Style` | 可复用的属性集合，可 `extends`，用 `class:` / `style:` 套用 | [样式](dsl/styles.md) |
| `Component` | 带参数的可复用子树，`Slot` 接收调用方子节点 | [组件](dsl/components.md) |
| 绑定 | 以 `@` 开头的值，读取 `DataStore` 或控件状态，数据变化时自动重算 | [响应式绑定](dsl/bindings.md) |
| 模板 | 字面文本中的 `{{ expression }}`，用于拼接 | [响应式绑定](dsl/bindings.md) |
| 事件脚本 | `{ ... }` 形式的交互逻辑，可赋值、做加减、写 `if/else` | [事件脚本](dsl/scripts.md) |
| 数据键 | `DataStore` 中的名字，如 `health`、`settings.theme` 的根 `settings` | [C++ 集成](cpp-guide.md) |
| 回调名称 | `.sil` 里写的字符串，由 C++ 用同名函数实现 | [C++ 集成](cpp-guide.md) |
| `ForEach` | 按数组生成行，数组替换时整段重建 | [列表](dsl/lists.md) |
| `Part` | 附着在任意控件上的装饰层，可按状态显示 | [控件参考](widgets.md#part) |
| 关键帧 / 过渡 | `@keyframes` 与 `transition`，用于动画 | [动画与过渡](dsl/animations.md) |

## 数据如何流动

```text
C++ DataStore::Set("health", 80)
    │
    ├─ 读取 health 的绑定立即重新求值（同步，写入线程上）
    │      text: @health            → Label.text = "80"
    │      color: @health > 50 ? …  → 重新应用颜色
    │
    └─ 下一帧 Render 绘制出新状态

.sil 脚本：@health += 10
    │
    └─ 同样经过 DataStore，触发上面的通知链
```

表达式解析名字时按以下顺序查找：

1. **事件载荷** `event.*`（仅在事件脚本执行期间存在，优先级最高）。
2. **局部作用域**：`ForEach` 注入的 `item` / `index`，`Component` 注入的参数名。
3. **`DataStore`**：全局数据键。
4. **控件状态**：`hovered`、`pressed`、`focused`、`disabled`，以及具体控件的状态（`checked`、`expanded`、`editing` 等）。

同名的数据键会遮住同名的控件状态，因此数据键命名时最好避开 `checked`、`visible` 这类状态名。

## 有意不做的部分

- 不支持 HTML、CSS、JavaScript，也不解析网页。
- 不绘制 3D 内容，只画二维控件。
- 不提供可视化编辑器；`.sil` 是纯文本。
- 不自动接管游戏的菜单、输入法或光标；这些由宿主模组决定。
- 不公开解析器与编译器的内部数据结构；文档只描述对外可见的语法与接口。

## 继续阅读

- 想尽快跑起来：[快速开始](getting-started.md)
- 想直接写界面：[DSL 使用教程](dsl/index.md)
- 想接入自己的模组：[C++ 库使用教程](cpp-guide.md)
- 遇到问题：[常见问题与故障排查](troubleshooting.md)

---

[上一章：首页](index.md) · [下一章：快速开始](getting-started.md)
