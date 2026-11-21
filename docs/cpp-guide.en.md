# C++ integration

This chapter is for developers who integrate Silica into a LeviLamina client mod. It assumes you can already load a `.dll` as a mod, and that you can obtain a `MinecraftUIRenderContext` during the game's UI render stage.

The split of responsibilities comes first:

| Part | Responsibility |
| --- | --- |
| Your mod | lifetime, render/input events, data source, business handlers |
| Silica | parsing `.sil`, building the widget tree, layout, drawing, input routing, bindings and animation |

Silica does not take over the screen stack and does not register its own window. It only draws one widget tree into the render context you provide.

## 1. Build integration

Silica ships as the static library `Silica.lib`, with headers in the repository's `include/` directory.

```text
<your mod>/
├── src/
├── ui/                 # .sil documents
├── build.bat
└── ...
```

Build requirements:

| Item | Value |
| --- | --- |
| Compiler | clang-cl (MSVC ABI) |
| C++ standard | C++20 |
| Runtime library | `/MD` |
| Linking | `Silica.lib` + `LeviLamina.lib` |

Reference compile and link flags:

```text
-I<Silica>/include
clang-cl ... /LD /MD /std:c++20 /I<Silica>/include ...
  /link <Silica>/dist/Silica.lib LeviLamina.lib ...
```

`Silica.lib` contains the Bedrock render and symbol resolution calls; it and your mod must use the same set of LeviLamina SDK headers and the same ABI.

## 2. Initialization

Do three things during mod load: set the log sink, set the document root directory, and compile a document.

```cpp
#include "silica/Silica.h"

silica::Silica::SetLogSink([](std::string_view line) {
    myModLog("%.*s", static_cast<int>(line.size()), line.data());
});

silica::Silica::SetRootPath(modDirectory);   // relative paths start from here

silica::WidgetPtr root = silica::Silica::CompileFile(
    "ui/main.sil",
    R"(
Panel {
    width: 200; height: 80;
    background: #2C2F33;
    radius: 6;
    Label { text: "UI failed to compile, check the log"; width: 200; height: 20; }
}
)"
);
```

Behavior of `CompileFile(filepath, fallbackSource)`:

- It reads and compiles the file; on failure it compiles `fallbackSource` instead.
- Diagnostics from both attempts go to the log sink.
- The return value is always a usable tree (unless the fallback source also fails to compile).
- The result is recorded under this call, so `ReloadUI` can find the matching entry.

!!! tip "Set the log sink first"
    Call `SetLogSink` before any compilation; otherwise diagnostics go only to stderr and the debug output, and never reach your mod log.

### Pre-importing resources

Documents with images need their images imported before the client resource pack stack is built:

```cpp
#include "silica/ImagePack.h"

if (root) silica::imagepack::Warmup(root);
```

Mod load time is the right moment. Rendering also resolves lazily, but without pre-importing the first frame may show a placeholder box first.

## 3. Compiling and diagnostics

When you need to inspect diagnostics one by one, use `UICompiler`:

```cpp
#include "silica/UICompiler.h"

silica::SilCompileResult result = silica::UICompiler::CompileDetailed(source, "ui/main.sil");
for (auto const& d : result.diagnostics) {
    log("%s", silica::FormatDiagnostic(d, result.origin).c_str());
}
if (!result.ok()) {
    // result.ok() requires a root node and no Error diagnostics
}
```

| Field | Description |
| --- | --- |
| `root` | the widget tree that was built; empty when an error prevents building |
| `diagnostics` | errors and warnings |
| `origin` | file name shown in diagnostics |
| `ok()` | has a root node and no `Error` |
| `Report()` | one formatted line per diagnostic |

Design principle: unknown properties, unknown styles, missing arguments and the like produce **warnings** and keep as much of a usable widget tree as possible; syntax errors, circular inheritance, recursive components and the like produce **errors**.

## 4. Data and bindings

`DataStore` is the global reactive state:

```cpp
#include "silica/DataStore.h"
#include "silica/Variant.h"

silica::DataStore::Set("playerCount", silica::Variant{42});
auto count = silica::DataStore::Get("playerCount");   // Variant
bool has   = silica::DataStore::Has("playerCount");
```

- `Set` notifies observers only when the value actually changes (deep comparison).
- Notifications are synchronous and happen on the writing thread; a handler may read and write `DataStore` again.
- Widgets subscribe weakly, so subscriptions expire automatically once the widget tree is destroyed.

### Arrays and objects

`Variant` supports Null / Bool / Number / String / Array / Object:

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

!!! warning "Containers are immutable"
    The arrays and objects of a `Variant` have no mutation API. To change one element, build a new array/object and `Set` it. Only a whole replacement triggers a `ForEach` rebuild and a binding refresh.

## 5. Event handlers

Write the handler name in `.sil` and register it by name in C++:

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

| Overload | Used for | Value received |
| --- | --- | --- |
| `ActionHandler` | `onClick` | none |
| `ValueHandler` | `onChange`/`onSubmit` | `"true"`/`"false"`, numeric text, entered text, or the selected index |

- `Invoke(name)` falls back to the value handler and passes an empty string.
- `Unregister(name)`, `Has(name)` and `Clear()` handle lifetime management.
- When the `.sil` document has an event script, the script runs first and the named handler is not called.

## 6. Attaching to a frame

The per-frame order is fixed: advance time → layout → pointer → render → flush draws.

```cpp
MinecraftUIRenderContext& context = /* your render callback */;
silica::WidgetPtr const& root = /* the compiled tree */;

static double last = -1.0;
double const now = frameSeconds();
float const dt = last < 0.0 ? 0.f : static_cast<float>(std::min(now - last, 0.1));
last = now;

silica::TickTree(*root, dt);   // bindings, dynamic rules, transitions, keyframes
root->Update();                // auto sizing and layout

// replay the press/release pointer edges first, then handle the hover position
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
silica::FlushDraws(context);   // submit all deferred draws at the end of the frame
```

`TickTree` walks the whole tree, including the inner content of a ScrollView. The frame delta is internally limited to 0.25 seconds, so nothing jumps after a stall.

!!! warning "Coordinates must be in GUI units"
    `PointerState`, widget geometry and `MinecraftUIRenderContext` all use GUI units. Windows pixel coordinates must be multiplied by `1 / GuiScale`. Mixing the two units shifts hit testing and displaces images.

### Input routing

| Object | Purpose |
| --- | --- |
| `PointerRouter::Update(root, state)` | hover, press, release, drag capture |
| `PointerRouter::Wheel(root, state, delta)` | one wheel notch; wheel events need a separate call |
| `PointerRouter::Hovered()` / `Captured()` | currently hovered/captured widget |
| `FocusManager::Get().Current()` | current keyboard focus |
| `FocusRing` | collects focusable widgets, Tab/arrow keys move, activation |

Suggested keyboard handling:

```cpp
if (auto* focused = silica::FocusManager::Get().Current()) {
    if (focused->OnKeyDown(keyCode)) return;
}
```

Do not pass keys consumed by a widget on to host shortcuts, otherwise a space typed in a text box turns into something like toggling the UI.

For Esc, do both: clear the focus (`FocusManager::Get().Clear()`) and close your overlay.

## 7. Finding and manipulating widgets

```cpp
#include "silica/Tree.h"

if (auto node = silica::FindByName(root, "volume_slider")) {
    if (auto slider = std::dynamic_pointer_cast<silica::SilicaSlider>(node)) {
        slider->SetValue(0.6f);
    }
}
```

- `FindByName` searches depth first and matches `name:` from the `.sil` document.
- `HitTest(root, x, y)` returns the deepest visible widget at that point.
- You can also drive the UI through `DataStore`, which is usually more robust than changing widget properties directly.

## 8. Reloading a document

```cpp
silica::WidgetPtr fresh = silica::Silica::ReloadUI("ui/main.sil");
if (fresh) {
    root = fresh;
    silica::imagepack::Warmup(root);
    focus.Clear();
    focusRing.Collect(root);
}
```

- When the reload fails, the old tree is kept and the failure diagnostics are logged.
- Widgets in the new tree are brand-new objects, so the old focus, drag capture and external pointers should all be reset.
- Run `Warmup` again so newly referenced images enter the resource pack.

## 9. Custom widgets

Widgets are created by `WidgetFactory` from a type name, and you can register yours before compiling a document:

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
            // parse the number...
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

Notes:

- An `ApplyProperty` that returns `false` produces an `unknown property` warning.
- Derive from `SilicaContainer` to hold arbitrary child widgets; with `SilicaWidget`/`SilicaPanel`, `AddChild` accepts only `Part`.
- Use `PaintAlpha()` as the opacity when drawing, so `opacity`, the disabled dimming and keyframe opacity take effect automatically.
- A custom widget must call `PaintParts` itself to show its `Part`s (containers and built-in widgets already do this).

## 10. Threads and ownership

| Rule | Description |
| --- | --- |
| Access the widget tree only on the render thread | do not call `Update`/`Render`/widget setters from a background thread |
| `DataStore` notifications are synchronous | the writing thread calls observers immediately; to write data across threads, switch back to the main thread first |
| Hold the root with `WidgetPtr` | between widgets the child holds a `shared_ptr` and the parent a raw pointer; do not form cycles |
| Subscriptions are weak | no manual unsubscribe is needed, but the widget tree must stay alive |
| Handler registration is process-wide | call `Clear()` or `Unregister` one by one when the mod unloads |

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Clicks land in the wrong place | pixel coordinates were not divided by GuiScale | convert everything to GUI units |
| The text box receives no characters | keys are not handed to `FocusManager::Current()` first | call `OnKeyDown` on the focused widget first |
| Data updates but the UI does not refresh | the binding uses a different key, or an equal value was written | check the key name; replace containers as a whole |
| An image is a placeholder box on the first frame | the resource pack is not ready yet | `Warmup` during load; rendering retries automatically |
| Clicks are wrong after a reload | the old focus/capture still points at destroyed widgets | call `Reset()`/`Clear()` after the reload and collect the focus again |
| A custom widget has no background | `PaintSelf` was not called | call `PaintSelf` first in `Render` |

<!-- Insert the UI render/input event registration snippet for the LeviLamina version you use here -->

Next: [Troubleshooting](troubleshooting.md) · [Examples](examples.md)
