# Silica

An embedded declarative UI framework for LeviLamina Bedrock Edition mod developers.

Silica lets mod authors declare interface structure, styles, and interaction in a purpose-built `.sil` description file, and a C++ runtime draws the widget tree directly inside the Minecraft Bedrock client. The syntax is close to QML and QSS: one node is one layer of the interface, properties are written as `key: value;`, data bindings start with `@`, and events are handed to C++ through named callbacks.

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
        text: "Silica loaded";
        height: 18;
        font-size: 0.9;
        color: #D9F2D9;
    }

    Button {
        text: "Click me";
        width: 96;
        height: 24;
        onClick: "hello_click";
    }
}
```

## What the project is

| Dimension | Description |
| --- | --- |
| Runtime environment | Minecraft Bedrock client + LeviLamina native mod |
| Language layers | Interface description: `.sil` (a domain-specific DSL); integration and logic: C++ |
| Render backend | The Minecraft client UI render context (`MinecraftUIRenderContext`) |
| Design goals | Declarative, reusable, data-driven; no hand-written coordinates or draw calls in C++ |
| Intended users | Mod authors who need custom in-game interfaces; no game files are modified |

Silica is not a wrapper around the game's native JSON-UI, nor a web/HTML rendering solution. Silica lays out and draws the widget tree itself and only reuses the game's fonts, textures, and render passes.

## Core features

- **Declarative interfaces**: nodes plus properties describe the interface, and layout is computed from container rules, so you never hand-write pixel coordinates.
- **Reactive bindings**: `@key` reads from `DataStore`; when the data changes, the value is re-evaluated and the widget refreshes automatically.
- **Event scripts**: `onClick: { @count = @count + 1; }` modifies variables directly; complex logic goes to C++ named callbacks.
- **Component reuse**: a top-level `Component` defines a parameterized subtree, and `Slot` receives the caller's child nodes.
- **Style system**: `Var` constants and `Style` blocks (with `extends` and `class` support), where inline properties take precedence.
- **List rendering**: `ForEach` generates rows from an array and rebuilds the whole block when the array changes.
- **Decoration parts**: `Part` overlays a decoration layer inside the widget box, and its visibility can be controlled by state conditions.
- **Animation and transitions**: `@keyframes` keyframes, `transition` property transitions, and smooth interpolation for sliders and lists.
- **Built-in widgets**: `Container`, `Panel`, `Label`, `Button`, `Image`, `ScrollView`, `Checkbox`, `Slider`, `TextBox`, `Dropdown`, `ForEach`, `Part`.
- **Image assets**: game texture names and `file:` images under the mod directory are supported; the latter are packed as a mod resource pack and loaded.
- **Diagnosable loading**: compile errors are written to a custom log with file, line, and column; when a document is broken, Silica falls back to a built-in interface instead of interrupting the game.

## When to use it

- A mod needs a standalone settings panel, item browser, HUD overlay, or dialog.
- The interface must refresh frequently with runtime data (inventory, health, configuration options, search results).
- You want interface and logic separated: `.sil` files are handed to UI authors, and C++ only handles data and callbacks.
- You need multi-page interfaces with a consistent look, reusing structure through `Style` and `Component`.

## What it is not for

- **Replacing the game's native menus**: it does not take over OreUI / JSON-UI and does not modify the game's own interfaces.
- **Rendering web content**: HTML, CSS, and JavaScript are not supported.
- **Rendering 3D scenes**: it only draws 2D interfaces and does not create world objects or models.
- **Running independently of a mod**: it must be loaded by a LeviLamina mod that drives rendering and input.
- **Being a WYSIWYG editor**: `.sil` is a plain text file; after editing, you must reload the interface to see the result.

## Versions and compatibility

- The framework depends on LeviLamina and the Bedrock client render interface, so after a game version update you must use the matching framework version.
- New `.sil` keywords and C++ interface changes are listed in the release notes.
- For a summary of known limitations, see [Troubleshooting](troubleshooting.md).

Current coverage: basic syntax and layout, all built-in widgets, reactive bindings and scripts, document-level `Var` / `Style` / `Component`, `ForEach`, `Part`, keyframes and transitions, external images, diagnostics, and fallback loading.

## Quick trial

The **SilicaTest** mod in the Silica repository is the recommended reference implementation: it demonstrates the complete chain from mod loading and interface rendering to input routing and data binding.

1. Build the framework static library (from the Silica repository root):

    ```bat
    build.bat
    ```

2. Build the sample mod (this first builds the framework and runs the host tests automatically):

    ```bat
    SilicaTest\build.bat
    ```

3. Copy the entire `SilicaTest/dist/SilicaTest/` directory into the LeviLamina instance's `mods/SilicaTest/`.
4. Start the game, enter a world, and press `P` to open the sample panel.

<!-- Insert the minimal mod code snippet for integrating Silica from scratch here -->

For complete dependencies, directory layout, and loading steps, see [Getting started](getting-started.md).

!!! tip "Only writing interfaces?"
    If the mod already integrates Silica, UI authors only need to edit `.sil` files and use the reload entry point the mod provides (`F5` in SilicaTest); no C++ recompilation is needed.

## Documentation map

| Chapter | Contents |
| --- | --- |
| [Project overview](overview.md) | Architecture summary, render flow, and concepts such as widget / binding / script / component / style |
| [Getting started](getting-started.md) | Environment dependencies, a minimal runnable example, and loading a `.sil` interface |
| [DSL tutorial](dsl/index.md) | Basic syntax, layout, widgets, bindings, scripts, components, styles, animation, lists, images |
| [Widget reference](widgets.md) | What each built-in widget does and its commonly used properties |
| [C++ library tutorial](cpp-guide.md) | Mod integration, loading UI, event callbacks, DataStore, diagnostic logs |
| [Troubleshooting](troubleshooting.md) | Reading error messages, bindings not working, image failures, unresponsive widgets, known limitations |
| [Examples](examples.md) | Reusable snippets such as settings panels, dialogs, dynamic lists, and toggle buttons |

## Who this is for

The documentation is organized around two kinds of readers:

- **UI authors**: mainly read the DSL tutorial, widget reference, and examples; you only need to understand `.sil` syntax and data key conventions.
- **Mod developers**: read the C++ tutorial on top of the UI author material, and are responsible for loading documents, registering callbacks, writing data, and collecting logs.

The documentation only describes the public usage; it does not cover the internal implementation of the compiler, parser, or widget tree.

## License

The `LICENSE` file in the repository at release time is authoritative.

---

[Next chapter: Project overview](overview.md)
