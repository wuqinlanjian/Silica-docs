# Silica

面向 LeviLamina 基岩版模组开发者的嵌入式声明式 UI 框架。

Silica 让模组作者用自研的 `.sil` 描述文件声明界面结构、样式与交互，由 C++ 运行时在 Minecraft Bedrock 客户端内直接绘制控件树。语法接近 QML 与 QSS：一个节点就是一层界面，属性用 `键: 值;` 书写，数据绑定以 `@` 开头，事件用同名回调交给 C++ 处理。

```sil
Container {
    name: hello_panel;
    width: 320;
    height: 120;
    layout: vertical;
    padding: 12;
    background: #0F1215F2;
    border: 1 solid #353A40;

    Label {
        text: "Silica 已加载";
        height: 18;
        font-size: 0.9;
        color: #D9F2D9;
    }

    Button {
        text: "点我";
        width: 96;
        height: 24;
        onClick: "hello_click";
    }
}
```

## 项目定位

| 维度 | 说明 |
| --- | --- |
| 运行环境 | Minecraft Bedrock 客户端 + LeviLamina 原生模组 |
| 语言分层 | 界面描述：`.sil`（领域特定 DSL）；集成与逻辑：C++ |
| 渲染后端 | Minecraft 客户端 UI 渲染上下文（`MinecraftUIRenderContext`） |
| 设计目标 | 声明式、可复用、数据驱动，避免在 C++ 中手写坐标与绘制调用 |
| 目标用户 | 需要自定义游戏内界面的模组作者；无需修改游戏文件 |

Silica 不是对游戏原生 JSON-UI 的封装，也不是网页/HTML 渲染方案。控件树由 Silica 自己布局并绘制，只复用游戏的字体、纹理与渲染通道。

## 核心特性

- **声明式界面**：节点 + 属性描述界面，布局按容器规则计算，无需手写像素坐标。
- **响应式绑定**：`@key` 读取 `DataStore`，数据变化时自动重新求值并刷新控件。
- **事件脚本**：`onClick: { @count = @count + 1; }` 可直接修改变量；复杂逻辑交给 C++ 命名回调。
- **组件复用**：顶层 `Component` 定义参数化子树，`Slot` 接收调用方子节点。
- **样式系统**：`Var` 常量、`Style` 样式块（支持 `extends` 与 `class`），内联属性优先。
- **列表渲染**：`ForEach` 按数组生成行，数组变化时整段重建。
- **装饰部件**：`Part` 在控件盒内叠加装饰层，可用状态条件控制显示。
- **动画与过渡**：`@keyframes` 关键帧、`transition` 属性过渡、滑块与列表的平滑插值。
- **内置控件**：`Container`、`Panel`、`Label`、`Button`、`Image`、`ScrollView`、`Checkbox`、`Slider`、`TextBox`、`Dropdown`、`ForEach`、`Part`。
- **图片资源**：支持游戏纹理名与模组目录下的 `file:` 图片，后者打包为模组资源包加载。
- **可诊断加载**：编译错误带文件、行、列输出到自定义日志；文档损坏时回退到内置界面，不会中断游戏。

## 适用场景

- 模组需要独立的设置面板、物品检索界面、HUD 浮层或弹窗。
- 界面需要频繁跟随运行时数据刷新（背包、血量、配置项、搜索结果）。
- 希望界面与逻辑分离：`.sil` 交给界面作者修改，C++ 只处理数据和回调。
- 需要统一风格的多页面界面，通过 `Style` 与 `Component` 复用结构。

## 不适合做什么

- **替代游戏原生菜单**：不接管 OreUI / JSON-UI，不修改游戏自带界面。
- **网页内容渲染**：不支持 HTML、CSS、JavaScript。
- **3D 场景渲染**：只绘制二维界面，不做世界内物件或模型。
- **独立于模组运行**：必须由 LeviLamina 模组加载并驱动渲染与输入。
- **所见即所得的编辑器**：`.sil` 是纯文本文件，修改后需重载界面查看效果。

## 版本与兼容性

- 框架依赖 LeviLamina 与 Bedrock 客户端渲染接口，游戏版本更新后需使用对应的框架版本。
- `.sil` 新增关键字与 C++ 接口变更会在版本说明中列出。
- 已知限制汇总见[常见问题与故障排查](troubleshooting.md)。

当前能力覆盖：基础语法与布局、全部内置控件、响应式绑定与脚本、文档级 `Var` / `Style` / `Component`、`ForEach`、`Part`、关键帧与过渡、外部图片、诊断与回退加载。

## 快速体验

推荐以 Silica 仓库中的 **SilicaTest** 作为参考实现：它演示了从模组加载、界面渲染、输入路由到数据绑定的完整链路。

1. 构建框架静态库（在 Silica 仓库根目录执行）：

    ```bat
    build.bat
    ```

2. 构建示例模组（会自动先构建框架并运行宿主测试）：

    ```bat
    SilicaTest\build.bat
    ```

3. 将 `SilicaTest/dist/SilicaTest/` 整个目录复制到 LeviLamina 实例的 `mods/SilicaTest/`。
4. 启动游戏，进入世界后按 `P` 打开示例面板。

<!-- 此处插入从零接入 Silica 的最小模组代码片段 -->

完整的依赖、目录结构与加载步骤见[快速开始](getting-started.md)。

!!! tip "只想写界面？"
    若模组已经接入 Silica，界面作者只需编辑 `.sil` 文件并使用模组提供的重载入口（SilicaTest 中为 `F5`），无需重新编译 C++。

## 文档导航

| 章节 | 内容 |
| --- | --- |
| [项目总览](overview.md) | 架构简述、渲染流程、Widget / 绑定 / 脚本 / 组件 / 样式等概念 |
| [快速开始](getting-started.md) | 环境依赖、最小可运行示例、加载一份 `.sil` 界面 |
| [DSL 使用教程](dsl/index.md) | 基础语法、布局、控件、绑定、脚本、组件、样式、动画、列表、图片 |
| [控件参考](widgets.md) | 各内置控件的作用与常用属性 |
| [C++ 库使用教程](cpp-guide.md) | 模组集成、加载 UI、事件回调、DataStore、诊断日志 |
| [常见问题与故障排查](troubleshooting.md) | 报错解读、绑定不生效、图片失败、控件无响应、已知限制 |
| [示例集锦](examples.md) | 设置面板、弹窗、动态列表、开关按钮等可复用片段 |

## 面向读者

文档按两类读者组织：

- **界面作者**：主要阅读 DSL 教程、控件参考与示例；只需理解 `.sil` 语法与数据键约定。
- **模组开发者**：在界面作者基础上阅读 C++ 教程，负责加载文档、注册回调、写入数据与收集日志。

文档只描述对外使用方式，不涉及编译器、解析器与控件树的内部实现。

## 许可证

以仓库发布时的 `LICENSE` 文件为准。

---

[下一章：项目总览](overview.md)
