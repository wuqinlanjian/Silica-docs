# Project overview

This chapter gives a whole-picture view of what Silica consists of, how a `.sil` document turns into widgets on the screen, and the key terms that recur throughout the documentation.

## What problem it solves

Hand-writing interfaces in a Bedrock Edition client mod usually means calling the game's UI render interface directly: computing the coordinates of every button yourself, handling hover and click yourself, and maintaining your own notion of "which part must be redrawn when the data changes". The more complex the interface, the harder the C++ code is to read, and changing a single spacing value requires a recompile.

Silica splits the work into two layers:

| Layer | Responsible for | Typical editor |
| --- | --- | --- |
| Interface description (`.sil`) | Structure, layout, text, colors, bindings, interaction logic | UI author |
| Integration logic (C++) | Loading documents, providing data, registering callbacks, wiring up rendering and input | Mod developer |

The `.sil` side describes "what the interface looks like and which parts follow when data changes"; C++ handles "when the interface appears, where the data comes from, and how game events get in". The two are decoupled through **data keys** and **callback names**: the interface writes `text: @player_name` and `onClick: "open_settings"`, and C++ only has to write `player_name` and register a function named `open_settings`.

## Layered architecture

```text
ui/main.sil
    │  syntax parsing
    ▼
Document model: nodes, Var, Style, Component, @keyframes
    │  style expansion, $variable substitution, component instantiation, binding/script parsing
    ▼
Widget tree: SilicaWidget (Container / Label / Button / ...)
    │  driven by the host every frame
    ▼
Tick (animation) → Update (layout) → input routing → Render → FlushDraws
    │
    ▼
MinecraftUIRenderContext (the game's UI draw interface)
```

The responsibilities of each layer:

- **Document model**: retains each node's type, parameters, property text, child nodes, and source location. Property values are still text at this layer; only the concrete widget knows what `font-size` or `layout` means.
- **Compilation**: expands `Style` and `class` into properties, substitutes `$variables`, instantiates `Component`, registers `@key` values as bindings, `{{ ... }}` as templates, and `{ ... }` as event scripts. Unknown properties are recorded as warnings and unknown widgets as errors, so a single typo never crashes the whole document.
- **Widget tree**: every widget holds its own geometry, styles, bindings, and scripts. Layout is computed by containers; a widget itself only draws itself and responds to input.
- **Host**: Silica does not hook into the game on its own. The mod calls `TickTree`, `Update`, `PointerRouter`, `Render`, and `FlushDraws` from suitable render events, and converts keyboard and mouse events into GUI coordinates before feeding them to the router.

!!! note "Silica does not take over the game interface"
    Silica draws into the render context the mod provides. It does not replace OreUI / JSON-UI, and it does not modify the game's own interface files. The sample SilicaTest mod temporarily pushes an empty native screen to obtain the cursor and focus; that is just one way for a host to integrate, not a requirement of the framework.

## What happens during a load

Take `Silica::CompileFile(path, fallback)` as an example:

1. **Parse**: read the file and produce the node tree plus document-level definitions (`Var`, `Style`, `Component`, `@keyframes`). Syntax errors enter the diagnostic list with line and column numbers.
2. **Expand**: for each node, apply the style chain specified by `class` / `style` (base style first, derived styles after, inline properties overriding last), then substitute `$variables` with text.
3. **Build**: create widgets by node type; set properties; register bindings, templates, and scripts; build child nodes recursively. `ForEach` records its template and later rebuilds rows from array elements; `Component` binds arguments into a child scope and then builds the function body.
4. **Pack images**: `imagepack::Warmup(root)` scans the tree for `file:` images and `Part` textures, copies them into `resource_packs/silica_images` under the mod directory, so the game sees them when it builds the resource pack stack.
5. **Mount**: the host stores the root widget, sets its position and size, and calls `Update()` once to complete the first frame's layout.

`Silica::CompileFile` records the compile result in an internal registry, so you can later call `Silica::ReloadUI(path)` on the same path:

- The file recompiles successfully: it is replaced with the new tree.
- Reading or compiling the file fails: the diagnostic is recorded, **the previous tree is kept**, and the interface does not disappear.

If you use `UICompiler::CompileFile(path, fallback)` directly, a failing main document causes the embedded `fallback` document to be compiled, guaranteeing that the mod has at least one usable interface.

## What to do every frame

The host's per-frame order is usually as follows (SilicaTest follows exactly this order):

| Step | Call | Effect |
| --- | --- | --- |
| 1 | `TickTree(root, deltaTime)` | Advance keyframes and transitions, recompute conditional values |
| 2 | `root->Update()` | Re-layout and write absolute coordinates into child widgets |
| 3 | `PointerRouter::Update` / `Wheel` | Hover, press, release, wheel, and drag |
| 4 | `root->Render(context)` | Draw background, border, text, child widgets, and decoration layers in order |
| 5 | `FlushDraws(context)` | Submit the fill and text batches accumulated this frame |

Keyboard input goes to the currently focused widget: `FocusManager::Get().Current()`, then call its `OnKeyDown(key)`. The focus ring `FocusRing` handles Tab/arrow-key ordering and Enter activation; whether to use it is up to the host.

!!! tip "Tick before layout"
    Keyframe scaling is a pure render transform, but transitions and conditional values change properties such as color and opacity. Calling `Tick` before `Update` guarantees that layout in the same frame sees the latest state.

## Core concepts

| Concept | Description | Details |
| --- | --- | --- |
| Interface document | A `.sil` file; the top level may contain definitions, followed by one or more interface nodes | [Basic syntax](dsl/basics.md) |
| Node / widget | One layer of the interface, such as `Button { ... }`; a node type corresponds to one widget implementation | [Widget reference](widgets.md) |
| Property | `key: value;`, for example `width: 120;`, `background: #202428;` | [Basic syntax](dsl/basics.md) |
| `Var` | A compile-time constant within the document, referenced as `$name` | [Styles](dsl/styles.md) |
| `Style` | A reusable set of properties that can `extends` another and is applied with `class:` / `style:` | [Styles](dsl/styles.md) |
| `Component` | A parameterized, reusable subtree; `Slot` receives the caller's child nodes | [Components](dsl/components.md) |
| Binding | A value starting with `@` that reads `DataStore` or widget state and is recomputed when the data changes | [Reactive bindings](dsl/bindings.md) |
| Template | `{{ expression }}` inside literal text, used for concatenation | [Reactive bindings](dsl/bindings.md) |
| Event script | Interaction logic in the form `{ ... }` that can assign, add and subtract, and use `if/else` | [Event scripts](dsl/scripts.md) |
| Data key | A name in `DataStore`, such as `health`, or the root `settings` of `settings.theme` | [C++ integration](cpp-guide.md) |
| Callback name | A string written in `.sil` and implemented by a C++ function with the same name | [C++ integration](cpp-guide.md) |
| `ForEach` | Generates rows from an array and rebuilds the whole block when the array is replaced | [Lists](dsl/lists.md) |
| `Part` | A decoration layer attached to any widget that can be shown according to state | [Widget reference](widgets.md#part) |
| Keyframes / transitions | `@keyframes` and `transition`, used for animation | [Animation and transitions](dsl/animations.md) |

## How data flows

```text
C++ DataStore::Set("health", 80)
    │
    ├─ bindings reading health are re-evaluated immediately (synchronously, on the writing thread)
    │      text: @health            → Label.text = "80"
    │      color: @health > 50 ? …  → the color is applied again
    │
    └─ the next frame's Render draws the new state

.sil script: @health += 10
    │
    └─ also goes through DataStore, triggering the same notification chain
```

When an expression resolves a name, it looks it up in this order:

1. **Event payload** `event.*` (exists only while an event script is running; highest priority).
2. **Local scope**: `item` / `index` injected by `ForEach`, and parameter names injected by `Component`.
3. **`DataStore`**: global data keys.
4. **Widget state**: `hovered`, `pressed`, `focused`, `disabled`, plus widget-specific state (`checked`, `expanded`, `editing`, and so on).

A data key with the same name shadows widget state with that name, so when naming data keys it is best to avoid state names such as `checked` and `visible`.

## Deliberate omissions

- HTML, CSS, and JavaScript are not supported, and no web pages are parsed.
- No 3D content is drawn; only 2D widgets are.
- No visual editor is provided; `.sil` is plain text.
- The game's menus, input method, and cursor are not taken over automatically; the host mod decides these.
- The parser's and compiler's internal data structures are not public; the documentation only describes the externally visible syntax and interfaces.

## Further reading

- Want to get it running quickly: [Getting started](getting-started.md)
- Want to write interfaces right away: [DSL tutorial](dsl/index.md)
- Want to integrate your own mod: [C++ library tutorial](cpp-guide.md)
- Running into problems: [Troubleshooting](troubleshooting.md)

---

[Previous chapter: Home](index.md) · [Next chapter: Getting started](getting-started.md)
