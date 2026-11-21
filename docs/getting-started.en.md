# Getting started

This chapter ties environment, build, deploy, and a minimal integration into one executable path. When you are done, you will have a custom panel you can open in game, and you will know how to verify changes to a `.sil` file.

## Prerequisites

| Item | Requirement |
| --- | --- |
| Operating system | Windows 10 / 11 x64 |
| Game | Minecraft Bedrock client with a working LeviLamina installation |
| Build environment | Visual Studio 2022 Build Tools + LLVM (clang-cl), using the MSVC ABI |
| Language standard | C++20 |
| Runtime library | Dynamic runtime library (`/MD`) |
| Silica | The source directory and build scripts, or an already built `Silica.lib` |
| Reference implementation | The `SilicaTest` mod in the Silica repository |

Silica links into the mod as a **static library**. It depends on the LeviLamina client SDK headers (`ll/api/...` and `mc/...`) and a set of third-party header-only libraries. The `build.bat` in the Silica repository already contains the local paths and xmake package cache paths; on another machine, adjust the `INC` and package version directories at the top of the script to match the actual installation.

!!! warning "ABI must match"
    Silica is compiled with the MSVC ABI. Your mod must link with the same ABI (clang-cl's `-Xclang -fms-extensions` already selects the MSVC ABI by default) and the same runtime library settings, otherwise it will crash while loading or when passing widget pointers.

## Directory layout

```text
Silica/
├── include/silica/        public headers (Silica.h, Widget.h, UICompiler.h ...)
├── src/                   static library implementation
├── tests/                 host tests (parser, fallback loading, etc.)
├── build.bat              build the static library and run tests
├── dist/Silica.lib        build output
└── SilicaTest/
    ├── src/main.cpp       mod entry point (ll_mod_load / enable / disable)
    ├── src/TestOverlay.cpp reference integration for rendering, input, data, and callbacks
    ├── ui/panel.sil       sample interface
    ├── ui/logo.png        external image example
    ├── manifest.json      mod manifest
    └── build.bat          build and package the sample mod
```

This documentation and its site configuration live in a separate documentation repository and are not shipped with the framework sources.

## Building the framework

From the Silica repository root, run:

```bat
build.bat
```

The script performs the following in order:

1. Compiles the framework sources with clang-cl;
2. Archives them into `dist\Silica.lib`;
3. Compiles and runs the parser tests and the file fallback tests;
4. Checks exported symbols to confirm the key classes and interfaces are present.

`BUILD PASS` means the build succeeded.

!!! note "The tests really run"
    The parser tests and fallback tests are host programs and do not require starting the game. If they fail, fix the framework before building the mod.

## Building the sample mod

```bat
SilicaTest\build.bat
```

This script first builds the current framework by calling `build.bat` in the parent directory, then:

1. Compiles the mod sources;
2. Links `Silica.lib`, `LeviLamina.lib`, and the system libraries;
3. Copies `manifest.json` and `ui/` into `SilicaTest\dist\SilicaTest\`;
4. Verifies the exported functions `ll_mod_load`, `ll_mod_enable`, `ll_mod_disable` and the packaged files.

## Deploying

Copy the whole build output directory into the instance's mod directory:

```text
<instance>/mods/SilicaTest/
├── SilicaTest.dll
├── manifest.json
└── ui/
    ├── panel.sil
    └── logo.png
```

Do not overwrite a DLL that is in use while the game is running. Close the game before copying, or copy it under a temporary name and rename it after quitting the game.

Start the game and enter a world, then press `P` to open the panel. The sample has other shortcuts too: `F5` reloads the document, `Esc` closes it, and Enter inside a text box submits.

## Checking the load result

SilicaTest writes `SilicaTest.log` in the mod directory; the interface-related lines start with `[sil]` / `[ui]`. A first load usually shows:

```text
[mod] loaded from ./mods/SilicaTest
[sil] file: path=ui/panel.sil root=yes name=test_panel widgets=...
[sil] fallback: path=ui/missing.sil root=yes name=test_panel widgets=...
[sil] reload: path=ui/panel.sil root=yes name=test_panel widgets=...
[ui] panel bound: hint=yes status=yes scroller=yes
[ui] ready (press P)
```

If you see `unknown property` or `unknown style`, the document contains a typo or a style name that does not match; the diagnostic includes the file, line, and column.

!!! tip "Fallback documents"
    The sample mod ships an embedded fallback document for `ui/panel.sil`. Even if the main document is missing or has a syntax error, pressing `P` still opens a simplified panel and the real error appears in the log. When integrating your own mod, keep the same safety net.

## Minimal integration

### 1. Initialize the root directory and logging

When the mod loads, tell Silica where its directory is and route compiler diagnostics to the mod log:

```cpp
#include "silica/Silica.h"

extern "C" __declspec(dllexport) bool ll_mod_load(ll::mod::Mod& self) {
    auto const dir = self.getModDir();

    silica::Silica::SetLogSink([](std::string_view line) {
        // Write to the mod's own log file or debug output
    });
    silica::Silica::SetRootPath(dir.string());
    return true;
}
```

Afterwards, all relative paths (`ui/main.sil`, `file:ui/logo.png`) are resolved against this directory.

### 2. Compile the document

```cpp
std::string const fallback = R"(
Container {
    layout: vertical;
    width: 320; height: 120;
    padding: 12;
    background: #0F1215F2;
    Label { text: "Failed to load the interface, check the log"; height: 18; color: #E05252; }
}
)";

silica::WidgetPtr root = silica::Silica::CompileFile("ui/main.sil", fallback);
```

When you need diagnostic details, use the detailed interface instead:

```cpp
silica::SilCompileResult result = silica::UICompiler::CompileFile("ui/main.sil");
for (auto const& diagnostic : result.diagnostics) {
    // severity / line / column / message
}
bool usable = result.ok();
```

### 3. Register callbacks

`onClick: "open_settings"` in a `.sil` file only stores a name. The mod registers a function with the same name at a suitable point:

```cpp
silica::SilicaHandlers::Register("open_settings", [] {
    // Open the settings page
});

silica::SilicaHandlers::Register("set_volume", [](std::string_view value) {
    // Values from widgets such as Slider / TextBox arrive as text
});
```

Action callbacks and value callbacks share one registry; on a name collision, the action callback is tried first and the value callback is used as the fallback.

### 4. Drive rendering and input

Silica does not hook into the game by itself. Your host must hand the render context to the widget tree in its own UI render events, and convert coordinates into GUI units and feed them to the router in keyboard and mouse events:

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

<!-- Insert the minimal wiring code snippet for LeviLamina render and input events here -->

!!! warning "Coordinate system"
    Silica geometry uses GUI units, matching the game's interface scale. Window pixel coordinates must be multiplied by `GuiData::mInvGuiScale` before being passed in. Refresh this ratio whenever the interface scale changes, otherwise click positions will be offset.

`SilicaTest::TestOverlay.cpp` demonstrates a complete integration: pushing a native screen to obtain the cursor, cancelling the native view draw in render events, converting mouse coordinates, and routing keys to the focused widget. You can use it as the starting point for your own mod.

## Writing your first interface

Create `ui/main.sil` in the mod directory:

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
        text: "Hello, Silica";
        height: 18;
        font-size: 1.0;
        color: #E6EAEE;
        margin-bottom: 8;
    }

    Button {
        text: "Close";
        width: 96;
        height: 24;
        background: #2F6B37;
        border: 1 solid #5CD65C;
        color: #D9F2D9;
        onClick: "close_panel";
    }
}
```

Register `close_panel` in C++, then place and draw the root widget the way your host does. When you need hot reload, call `Silica::ReloadUI("ui/main.sil")` and replace the old tree with the returned new one; the sample binds this to `F5`.

## Next steps

- Learn the syntax systematically: [DSL tutorial](dsl/index.md)
- Look up widget properties: [Widget reference](widgets.md)
- Finish the mod-side integration: [C++ library tutorial](cpp-guide.md)
- Startup failures or a blank interface: [Troubleshooting](troubleshooting.md)

---

[Previous chapter: Project overview](overview.md) · [Next chapter: DSL tutorial navigation](dsl/index.md)
