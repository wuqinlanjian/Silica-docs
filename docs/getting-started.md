# 快速开始

本章把环境、构建、部署与最小集成串成一条可执行的路径。完成后你会得到一个能在游戏内打开的自定义面板，并知道修改 `.sil` 后如何验证。

## 前置条件

| 项目 | 要求 |
| --- | --- |
| 操作系统 | Windows 10 / 11 x64 |
| 游戏 | Minecraft Bedrock 客户端，已安装可用的 LeviLamina |
| 编译环境 | Visual Studio 2022 Build Tools + LLVM（clang-cl），使用 MSVC ABI |
| 语言标准 | C++20 |
| 运行库 | 动态运行库（`/MD`） |
| Silica | 源码目录与构建脚本，或已经构建好的 `Silica.lib` |
| 参考实现 | Silica 仓库中的 `SilicaTest` 模组 |

Silica 以**静态库**形式链接进模组。它依赖 LeviLamina 客户端 SDK 头文件（`ll/api/...` 与 `mc/...`），以及一组第三方头文件库。Silica 仓库里的 `build.bat` 已经写好了本机路径与 xmake 包缓存路径；换机器时，按实际安装位置调整脚本开头的 `INC` 与包版本目录即可。

!!! warning "ABI 必须一致"
    Silica 使用 MSVC ABI 编译。模组必须用相同的 ABI（clang-cl 的 `-Xclang -fms-extensions` 默认即 MSVC ABI）与相同的运行库设置链接，否则会在加载或传递控件指针时崩溃。

## 目录结构

```text
Silica/
├── include/silica/        公开头文件（Silica.h、Widget.h、UICompiler.h …）
├── src/                   静态库实现
├── tests/                 宿主测试（解析器、回退加载等）
├── build.bat              构建静态库并运行测试
├── dist/Silica.lib        构建产物
└── SilicaTest/
    ├── src/main.cpp       模组入口（ll_mod_load / enable / disable）
    ├── src/TestOverlay.cpp 渲染、输入、数据与回调的参考接入
    ├── ui/panel.sil       示例界面
    ├── ui/logo.png        外部图片示例
    ├── manifest.json      模组清单
    └── build.bat          构建并打包示例模组
```

本文档与其站点构建配置维护在单独的文档仓库中，不随框架源码发布。

## 构建框架

在 Silica 仓库根目录执行：

```bat
build.bat
```

脚本依次完成：

1. 用 clang-cl 编译框架源文件；
2. 归档为 `dist\Silica.lib`；
3. 编译并运行解析器测试与文件回退测试；
4. 检查导出符号，确认关键类与接口存在。

看到 `BUILD PASS` 即构建成功。

!!! note "测试会真正跑起来"
    解析器测试与回退测试是宿主程序，不需要启动游戏。如果它们失败，先修好框架再构建模组。

## 构建示例模组

```bat
SilicaTest\build.bat
```

该脚本会先在上层目录调用 `build.bat` 构建当前框架，然后：

1. 编译模组源文件；
2. 链接 `Silica.lib`、`LeviLamina.lib` 与系统库；
3. 把 `manifest.json` 与 `ui/` 复制到 `SilicaTest\dist\SilicaTest\`；
4. 校验导出函数 `ll_mod_load`、`ll_mod_enable`、`ll_mod_disable` 与打包文件。

## 部署

把构建输出目录整体复制到实例的模组目录：

```text
<实例>/mods/SilicaTest/
├── SilicaTest.dll
├── manifest.json
└── ui/
    ├── panel.sil
    └── logo.png
```

不要在游戏运行时直接覆盖正在使用的 DLL。关闭游戏后再复制，或先复制为临时文件、退出游戏后改名替换。

启动游戏并进入世界，按 `P` 打开面板。示例中还有其他快捷键：`F5` 重载文档、`Esc` 关闭、文本框内回车提交。

## 查看加载结果

SilicaTest 会在模组目录写 `SilicaTest.log`，其中与界面相关的行以 `[sil]` / `[ui]` 开头。首次加载通常会看到：

```text
[mod] loaded from ./mods/SilicaTest
[sil] file: path=ui/panel.sil root=yes name=test_panel widgets=...
[sil] fallback: path=ui/missing.sil root=yes name=test_panel widgets=...
[sil] reload: path=ui/panel.sil root=yes name=test_panel widgets=...
[ui] panel bound: hint=yes status=yes scroller=yes
[ui] ready (press P)
```

如果出现 `unknown property` 或 `unknown style`，说明文档里有拼写或样式名不一致；诊断会带文件、行号与列号。

!!! tip "回退文档"
    示例模组给 `ui/panel.sil` 准备了一份内嵌回退文档。即使主文档缺失或语法错误，按 `P` 仍然能打开一个简化面板，并在日志中看到真实报错。接入自己的模组时，建议保留同样的兜底策略。

## 最小集成

### 1. 初始化解根目录与日志

模组加载时告诉 Silica 自己的目录在哪里，并把编译器诊断接到模组日志：

```cpp
#include "silica/Silica.h"

extern "C" __declspec(dllexport) bool ll_mod_load(ll::mod::Mod& self) {
    auto const dir = self.getModDir();

    silica::Silica::SetLogSink([](std::string_view line) {
        // 写入模组自己的日志文件或调试输出
    });
    silica::Silica::SetRootPath(dir.string());
    return true;
}
```

之后所有相对路径（`ui/main.sil`、`file:ui/logo.png`）都相对于这个目录解析。

### 2. 编译文档

```cpp
std::string const fallback = R"(
Container {
    layout: vertical;
    width: 320; height: 120;
    padding: 12;
    background: #0F1215F2;
    Label { text: "界面加载失败，请查看日志"; height: 18; color: #E05252; }
}
)";

silica::WidgetPtr root = silica::Silica::CompileFile("ui/main.sil", fallback);
```

需要诊断详情时改用详细接口：

```cpp
silica::SilCompileResult result = silica::UICompiler::CompileFile("ui/main.sil");
for (auto const& diagnostic : result.diagnostics) {
    // severity / line / column / message
}
bool usable = result.ok();
```

### 3. 注册回调

`.sil` 里的 `onClick: "open_settings"` 只会保存一个名字。模组在合适的时候注册同名函数：

```cpp
silica::SilicaHandlers::Register("open_settings", [] {
    // 打开设置页
});

silica::SilicaHandlers::Register("set_volume", [](std::string_view value) {
    // Slider / TextBox 等控件的值以文本形式到达
});
```

动作回调与值回调共用一张注册表；同名时先尝试动作回调，再回退到值回调。

### 4. 驱动渲染与输入

Silica 不自行挂钩游戏。宿主需要在自己的 UI 渲染事件中，把渲染上下文交给控件树；在键鼠事件中，把坐标换算成 GUI 单位并送入路由器：

```cpp
silica::TickTree(*root, deltaTime);
root->Update();

silica::PointerState pointer;
pointer.x = cursorX;
pointer.y = cursorY;
pointer.down = leftDown;
pointer.pressed = leftPressedThisFrame;
pointer.released = leftReleasedThisFrame;
pointerRouter.Update(root, pointer);

root->Render(context);
silica::FlushDraws(context);
```

<!-- 此处插入 LeviLamina 渲染事件与输入事件的最小接入代码片段 -->

!!! warning "坐标系"
    Silica 的几何使用 GUI 单位，与游戏界面缩放一致。窗口像素坐标需要乘以 `GuiData::mInvGuiScale` 后再传入。每次界面缩放变化后都要刷新这个比例，否则点击位置会偏移。

`SilicaTest::TestOverlay.cpp` 展示了一套完整接入：压入原生界面获得光标、在渲染事件里取消原生视图绘制、换算鼠标坐标、把按键交给焦点控件。可以把它作为自己模组的起点。

## 写第一份界面

在模组目录创建 `ui/main.sil`：

```sil
Container {
    name: main_panel;
    layout: vertical;
    width: 320;
    height: 140;
    padding: 12;
    background: #0F1215F2;
    border: 1 solid #353A40;

    Label {
        text: "你好，Silica";
        height: 18;
        font-size: 1.0;
        color: #E6EAEE;
        margin-bottom: 8;
    }

    Button {
        text: "关闭";
        width: 96;
        height: 24;
        background: #2F6B37;
        border: 1 solid #5CD65C;
        color: #D9F2D9;
        onClick: "close_panel";
    }
}
```

在 C++ 中注册 `close_panel`，然后把根控件按你的宿主方式摆放与绘制。需要热重载时，调用 `Silica::ReloadUI("ui/main.sil")` 并用返回的新树替换旧树；示例中该操作绑定在 `F5`。

## 下一步

- 系统学习语法：[DSL 使用教程](dsl/index.md)
- 查控件属性：[控件参考](widgets.md)
- 完成模组侧接入：[C++ 库使用教程](cpp-guide.md)
- 启动失败或界面空白：[常见问题与故障排查](troubleshooting.md)

---

[上一章：项目总览](overview.md) · [下一章：DSL 教程导航](dsl/index.md)
