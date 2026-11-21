# C++ 集成

这一章面向在 LeviLamina 客户端模组里接入 Silica 的开发者。假设你已经能让模组加载 `.dll`，并且能在游戏 UI 渲染阶段拿到 `MinecraftUIRenderContext`。

分工先明确：

| 部分 | 职责 |
| --- | --- |
| 你的模组 | 生命周期、渲染/输入事件、数据来源、业务处理器 |
| Silica | 解析 `.sil`、构建控件树、布局、绘制、输入路由、绑定与动画 |

Silica 不接管屏幕栈，也不注册自己的窗口。它只把一棵控件树画进你提供的渲染上下文里。

## 一、接入构建

Silica 以静态库 `Silica.lib` 的形式提供，头文件在仓库的 `include/` 目录。

```text
<你的模组>/
├── src/
├── ui/                 # .sil 文档
├── build.bat
└── ...
```

构建要求：

| 项 | 值 |
| --- | --- |
| 编译器 | clang-cl（MSVC ABI） |
| C++ 标准 | C++20 |
| 运行库 | `/MD` |
| 链接 | `Silica.lib` + `LeviLamina.lib` |

编译和链接参数参考：

```text
-I<Silica>/include
clang-cl ... /LD /MD /std:c++20 /I<Silica>/include ...
  /link <Silica>/dist/Silica.lib LeviLamina.lib ...
```

`Silica.lib` 里包含 Bedrock 渲染与符号解析调用；它和你的模组必须使用同一套 LeviLamina SDK 头文件与 ABI。

## 二、初始化

在模组加载阶段完成三件事：设置日志出口、设置文档根目录、编译文档。

```cpp
#include "silica/Silica.h"

silica::Silica::SetLogSink([](std::string_view line) {
    myModLog("%.*s", static_cast<int>(line.size()), line.data());
});

silica::Silica::SetRootPath(modDirectory);   // 相对路径都从这里开始

silica::WidgetPtr root = silica::Silica::CompileFile(
    "ui/main.sil",
    R"(
Panel {
    width: 200; height: 80;
    background: #2C2F33;
    radius: 6;
    Label { text: "UI 编译失败，请查看日志"; width: 200; height: 20; }
}
)"
);
```

`CompileFile(filepath, fallbackSource)` 的行为：

- 读取并编译文件；失败时改用 `fallbackSource` 再编译一次。
- 两次尝试的诊断都写入日志出口。
- 返回值总是一棵可用的树（除非回退源码也编译失败）。
- 记录在这次调用下，`ReloadUI` 才能找到对应条目。

!!! tip "先设置日志出口"
    `SetLogSink` 应在任何编译之前调用，否则诊断只会写到 stderr 和调试输出，模组日志里看不到。

### 资源预导入

带图片的文档需要在客户端资源包栈建立前把图片导入：

```cpp
#include "silica/ImagePack.h"

if (root) silica::imagepack::Warmup(root);
```

模组加载阶段是合适的时机。渲染时也会惰性解析，但没有预导入时首帧可能先显示占位框。

## 三、编译与诊断

需要逐条检查诊断时，使用 `UICompiler`：

```cpp
#include "silica/UICompiler.h"

silica::SilCompileResult result = silica::UICompiler::CompileDetailed(source, "ui/main.sil");
for (auto const& d : result.diagnostics) {
    log("%s", silica::FormatDiagnostic(d, result.origin).c_str());
}
if (!result.ok()) {
    // result.ok() 要求有根节点且没有 Error
}
```

| 字段 | 说明 |
| --- | --- |
| `root` | 构建出的控件树；出现阻止构建的错误时为空 |
| `diagnostics` | 错误与警告 |
| `origin` | 诊断里显示的文件名 |
| `ok()` | 有根节点且无 `Error` |
| `Report()` | 每条诊断一行，已格式化 |

设计原则：未知属性、未知样式、缺少参数等产生**警告**，尽量保留可用的控件树；语法错误、循环继承、递归组件等产生**错误**。

## 四、数据与绑定

`DataStore` 是全局响应式状态：

```cpp
#include "silica/DataStore.h"
#include "silica/Variant.h"

silica::DataStore::Set("playerCount", silica::Variant{42});
auto count = silica::DataStore::Get("playerCount");   // Variant
bool has   = silica::DataStore::Has("playerCount");
```

- `Set` 只有在值真正变化（深比较）时才通知观察者。
- 通知是同步的，发生在写入线程上；处理器里可以再次读写 `DataStore`。
- 控件以弱引用订阅，控件树析构后订阅自动失效。

### 数组与对象

`Variant` 支持 Null / Bool / Number / String / Array / Object：

```cpp
using silica::Variant;

VariantArray rows;
for (auto const& entry : entries) {
    rows.push_back(Variant::Object({
        {"name", Variant{entry.name}},
        {"count", Variant{entry.count}},
    }));
}
silica::DataStore::Set("rows", Variant::Array(std::move(rows)));
```

!!! warning "容器是不可变的"
    `Variant` 的数组和对象没有修改接口。改一个元素时，请构造新的数组/对象再 `Set`。只有整体替换才会触发 `ForEach` 重建和绑定刷新。

## 五、事件处理器

`.sil` 里写处理器名，C++ 里按名字注册：

```cpp
#include "silica/Handlers.h"

silica::SilicaHandlers::Register("save", [] {
    // Button { onClick: "save"; }
});

silica::SilicaHandlers::Register("set_volume", [](std::string_view value) {
    // Slider { onChange: "set_volume"; }
    double v = std::stod(std::string(value));
});
```

| 重载 | 适用 | 收到的值 |
| --- | --- | --- |
| `ActionHandler` | `onClick` | 无 |
| `ValueHandler` | `onChange`/`onSubmit` | `"true"`/`"false"`、数字文本、输入文本或选中索引 |

- `Invoke(name)` 会回退到值处理器并传空串。
- `Unregister(name)`、`Has(name)`、`Clear()` 用于生命周期管理。
- `.sil` 里写了事件脚本时，脚本优先执行，命名处理器不会被调用。

## 六、挂载到一帧

每帧顺序固定：推进时间 → 布局 → 指针 → 渲染 → 刷新绘制。

```cpp
MinecraftUIRenderContext& context = /* 你的渲染回调 */;
silica::WidgetPtr const& root = /* 已编译的树 */;

static double last = -1.0;
double const now = frameSeconds();
float const dt = last < 0.0 ? 0.f : static_cast<float>(std::min(now - last, 0.1));
last = now;

silica::TickTree(*root, dt);   // 绑定、动态规则、过渡、关键帧
root->Update();                // 自动尺寸与布局

// 指针按下/抬起边沿先回放，再处理悬停位置
silica::PointerState down;
down.x = px; down.y = py; down.down = true; down.pressed = true;
pointerRouter.Update(root, down);

silica::PointerState up;
up.x = px; up.y = py; up.released = true;
pointerRouter.Update(root, up);

silica::PointerState hover;
hover.x = px; hover.y = py;
pointerRouter.Update(root, hover);

root->Render(context);
silica::FlushDraws(context);   // 帧末提交所有延迟绘制
```

`TickTree` 会遍历整棵树（包括 ScrollView 的内部内容）。帧间隔内部限制在 0.25 秒以内，卡顿后不会跳变。

!!! warning "坐标必须是 GUI 单位"
    `PointerState`、控件几何和 `MinecraftUIRenderContext` 都使用 GUI 单位。Windows 像素坐标需要乘以 `1 / GuiScale`。混淆单位会导致命中偏移和图片错位。

### 输入路由

| 对象 | 用途 |
| --- | --- |
| `PointerRouter::Update(root, state)` | 悬停、按下、抬起、拖动捕获 |
| `PointerRouter::Wheel(root, state, delta)` | 每格滚轮；滚轮事件需要单独调用 |
| `PointerRouter::Hovered()` / `Captured()` | 当前悬停/捕获控件 |
| `FocusManager::Get().Current()` | 当前键盘焦点 |
| `FocusRing` | 收集可聚焦控件，Tab/方向键切换，激活 |

键盘处理建议：

```cpp
if (auto* focused = silica::FocusManager::Get().Current()) {
    if (focused->OnKeyDown(keyCode)) return;
}
```

控件消费掉的按键不要继续传给宿主快捷键，否则输入框里的空格会变成开关界面之类的行为。

Esc 建议同时做两件事：清空焦点（`FocusManager::Get().Clear()`）和关闭你的覆盖层。

## 七、查找与操作控件

```cpp
#include "silica/Tree.h"

if (auto node = silica::FindByName(root, "volume_slider")) {
    if (auto slider = std::dynamic_pointer_cast<silica::SilicaSlider>(node)) {
        slider->SetValue(0.6f);
    }
}
```

- `FindByName` 深度优先，匹配 `.sil` 里的 `name:`。
- `HitTest(root, x, y)` 返回该点最深的可见控件。
- 也可以通过 `DataStore` 驱动 UI，通常比直接改控件属性更稳。

## 八、重载文档

```cpp
silica::WidgetPtr fresh = silica::Silica::ReloadUI("ui/main.sil");
if (fresh) {
    root = fresh;
    silica::imagepack::Warmup(root);
    focus.Clear();
    focusRing.Collect(root);
}
```

- 重载失败时保留旧树，并记录失败诊断。
- 新树的控件是全新对象，旧的焦点、拖动捕获和外部指针都应重置。
- 重新执行 `Warmup`，让新引用的图片进入资源包。

## 九、自定义控件

控件通过 `WidgetFactory` 按类型名创建，可以在编译文档之前注册：

```cpp
#include "silica/WidgetFactory.h"

class HealthBar : public silica::SilicaWidget {
public:
    void Render(MinecraftUIRenderContext& context) override {
        silica::FillRect(context, x, y, width, height, silica::Color{.2f, .2f, .2f, 1.f});
        silica::FillRect(context, x, y, width * fraction, height, silica::Color{.8f, .2f, .2f, 1.f});
    }

    bool ApplyProperty(std::string_view key, std::string_view value) override {
        if (key == "fraction") {
            // 解析数字……
            return true;
        }
        return silica::SilicaWidget::ApplyProperty(key, value);
    }

    float fraction = 1.f;
};

silica::WidgetFactory::Register("HealthBar", [] {
    return std::make_shared<HealthBar>();
});
```

注意：

- 返回 `false` 的 `ApplyProperty` 会产生 `unknown property` 警告。
- 继承 `SilicaContainer` 才能容纳任意子控件；继承 `SilicaWidget`/`SilicaPanel` 时 `AddChild` 只接受 `Part`。
- 绘制时使用 `PaintAlpha()` 作为透明度，让 `opacity`、禁用变暗和关键帧透明度自动生效。
- 自定义控件需要自己调用 `PaintParts` 才能显示 `Part`（容器和内置控件已处理）。

## 十、线程与所有权

| 规则 | 说明 |
| --- | --- |
| 控件树只在渲染线程访问 | 不要在后台线程调用 `Update`/`Render`/控件 setter |
| `DataStore` 通知同步发生 | 写入线程会立即回调观察者；跨线程写数据请先切回主线程 |
| 根用 `WidgetPtr` 持有 | 控件之间父子是 `shared_ptr` 子 + 裸指针父，不要形成环 |
| 订阅是弱引用 | 不需要手动退订，但控件树必须存活 |
| 处理器注册是进程级 | 卸载模组时 `Clear()` 或逐个 `Unregister` |

## 常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 点击位置偏移 | 像素坐标未除以 GuiScale | 统一换算成 GUI 单位 |
| 输入框收不到字符 | 按键未先交给 `FocusManager::Current()` | 先调用焦点的 `OnKeyDown` |
| 数据更新 UI 不刷新 | 绑定的不是同一根键，或写入了相等值 | 检查键名；容器要整体替换 |
| 图片首帧是占位框 | 资源包尚未就绪 | 在加载阶段 `Warmup`，渲染时会自动重试 |
| 重载后点击错乱 | 旧焦点/捕获仍指向已销毁控件 | 重载后 `Reset()`/`Clear()` 并重新收集焦点 |
| 自定义控件没有背景 | 没有调用 `PaintSelf` | 在 `Render` 里先调用 `PaintSelf` |

<!-- 此处插入你所用 LeviLamina 版本的 UI 渲染/输入事件注册代码片段 -->

下一步：[故障排查](troubleshooting.md) · [示例](examples.md)
