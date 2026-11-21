# DSL tutorial

`.sil` is the UI description language of Silica. Its node structure is close to QML, and it borrows the property syntax of QSS: a node is one level of UI, properties are written as `key: value;`, a value starting with `@` is a reactive binding, and a value written as `{ ... }` is an event script.

This tutorial is split by topic and is best read in order; skip ahead when you already know a topic.

## Chapters

| Chapter | What it covers | Audience |
| --- | --- | --- |
| [Basic syntax](basics.md) | Nodes, properties, value types, colors and numbers, comments, top-level definitions | Everyone |
| [Layout](layout.md) | Vertical/horizontal/absolute layout, sizes and margins, padding, stacking order | Everyone |
| [Common widgets](widgets.md) | Using text, buttons, checkboxes, sliders, text boxes, dropdowns and scroll views | Everyone |
| [Reactive bindings](bindings.md) | `@key`, expressions, templates, two-way binding, widget state | Everyone |
| [Event scripts](scripts.md) | `onClick: { ... }`, assignment, `if/else`, the `event` payload | Everyone |
| [Components](components.md) | `Component`, parameter defaults, `Slot`, instance overrides | UI authors |
| [Styles](styles.md) | `Var`, `Style`, `extends`, `class`, inline overrides | UI authors |
| [Animations and transitions](animations.md) | `transition`, `@keyframes`, easing, animatable properties | UI authors |
| [Lists](lists.md) | `ForEach`, row scope, adding, removing and updating data, performance limits | UI authors |
| [Images](images.md) | Game texture names, `file:` images, resource packs, failure behavior | Everyone |

## Writing conventions

- Files are UTF-8 encoded and use the `.sil` extension.
- Node types are case-sensitive; use the built-in type names (`Container`, `Label`, `Button` ...).
- Property names use **lowercase with hyphens**: `font-size`, `margin-left`, `z-order`. Case and underscores are not converted to hyphens automatically.
- Event properties accept `onClick`, `on-click`, `ONCLICK` and similar forms; internally they are normalized to the same event name.
- Style names and `class` names are case-insensitive and ignore `-` and `_`: `DemoRowBox`, `demo-row-box` and `demo_row_box` all refer to the same style.
- Strings use double quotes. Always quote text that contains spaces, colons or special characters.
- Colors support `#RGB`, `#RGBA`, `#RRGGBB`, `#RRGGBBAA` and a few color names (`white`, `black`, `red`, `green`, `blue`, `gray` and so on).
- Numbers are in GUI units and may carry the `px`, `em`, `pt` or `dp` suffix; the suffix does not change the value, it only makes the code easier to write.
- Every property ends with `;`. The last property may omit the semicolon, but adding it everywhere is less error-prone.

!!! note "Property values and source text"
    Bindings and scripts keep the raw expression text: the quotes of strings and the spaces around operators are passed to the evaluator as written. Therefore `text: @health > 0 ? "Alive" : "Dead";` and `text:@health>0?"Alive":"Dead";` are equivalent.

## A complete minimal document

```sil
Var accent: #5CD65C;

Container {
    name: demo_panel;
    layout: vertical;
    width: 300;
    height: 150;
    padding: 12;
    background: #0F1215F2;
    border: 1 solid #353A40;

    Label {
        text: "Health: {{@health}}";
        height: 16;
        font-size: 0.8;
        color: $accent;
    }

    Slider {
        height: 20;
        margin-top: 8;
        min: 0;
        max: 100;
        value: @health;
        onChange: { @health = event.value; };
    }

    Button {
        text: "Reset";
        width: 80;
        height: 22;
        margin-top: 10;
        onClick: { @health = 100; };
    }
}
```

This document uses `Var`, bindings, templates, event scripts and widget state at the same time. The following chapters cover the details of each part.

## Learning tips

- **Structure first, data later**: lay the UI out with plain literals first, confirm that the layout is correct, then turn the values that must follow data into `@bindings`.
- **Stable names**: give nodes that C++ has to look up, or that tests have to find, a `name`, for example `name: settings_panel;`. Names should be unique across the whole tree.
- **Check the log first**: a misspelled property only produces a warning and the widget keeps its default value; compile errors point at the line and column in the log. When something "does not work", check the log first.
- **Reload in small steps**: when the host supports reload (the examples use `F5`), change one small piece at a time instead of mixing several problems together.

---

[Previous: Getting started](../getting-started.md) · [Next: Basic syntax](basics.md)
