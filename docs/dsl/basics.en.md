# Basic syntax

This page explains what a `.sil` file consists of, how nodes and properties are written, how values are classified, and the rules for numbers, colors, strings and comments.

## Document structure

A document consists of two kinds of content:

1. **Top-level definitions**: `Var`, `Style`, `Component`, `@keyframes`. They do not create widgets directly; later nodes only reference them.
2. **UI nodes**: one or more top-level widgets. The most common case is a tree rooted at a `Container`.

```sil
// Top-level definitions
Var accent: #5CD65C;

Style Card {
    background: #171B1F;
    border: 1 solid #2A3036;
    padding: 8;
}

// UI nodes
Container {
    name: root;
    layout: vertical;
    width: 360;
    height: 200;

    Container {
        class: "card";
        height: 60;
        Label { text: "Styles and variables"; height: 16; color: $accent; }
    }
}
```

Definitions may appear before or after the UI nodes; references are resolved at compile time, so no declaration first is required. A document must have at least one top-level node, otherwise compilation fails with `the document does not define any component`.

If you write more than one top-level node, Silica puts them into a vertical `Container` in order, so they do not all stack at `(0, 0)`. When you need precise control, it is best to always keep a single root node.

## Nodes

Node syntax:

```text
Type parameter-list? { property; child nodes... }
```

- **Type**: a built-in widget name (`Container`, `Panel`, `Label`, `Button`, `Image`, `ScrollView`, `Checkbox`, `Slider`, `TextBox`, `Dropdown`, `ForEach`, `Part`), or the name of a `Component` defined in the document.
- **Parameter list**: used only by `ForEach` and `Component`, in the form `(data: @rows, item: @row)`. A parameter list on an ordinary widget has no meaning.
- **Properties**: `key: value;`, written inside the braces.
- **Child nodes**: write the next node directly after the properties; no comma is needed.

```sil
Container {
    layout: horizontal;
    height: 28;

    Label { text: "Name"; width: 80; height: 16; margin-top: 6; }
    Button { text: "OK"; width: 72; height: 24; onClick: "confirm"; }
}
```

When a node type is neither a built-in widget nor a defined component, compilation produces the error `unknown component '...'` and the node is not created.

## Classifying properties and values

The same `key: value;` line falls into one of four categories depending on how the value is written, and each behaves differently:

| Syntax | Category | Example | Behavior |
| --- | --- | --- | --- |
| Starts with `@` | Reactive binding | `text: @health;` | Parsed as an expression; re-evaluated when the related `DataStore` key changes |
| Contains `{{ ... }}` | Template | `text: "Health: {{@health}}"` | Only the expressions inside the braces are replaced, the rest of the text is kept as written |
| The value is `{ ... }` | Event script | `onClick: { @health = 100; }` | Bound to the event name; the statements run when it fires |
| Anything else | Literal property | `width: 120;` | Passed straight to the widget's property handler |

!!! warning "Only event properties can carry scripts"
    Scripts must be written on an event property, such as `onClick`, `onChange` or `onSubmit`. Writing `text: { ... }` produces the warning: `scripts attach to an event property (...)`. Scripts do not support `$variable` substitution; inside a script, `$` denotes a local variable of the current run.

`$name` in a literal property is replaced by the value of the `Var` at compile time:

```sil
Var gap: 8;
Var accent: #5CD65C;

Container {
    padding: $gap;
    background: #171B1F;
    Label { text: "Compile-time substitution"; color: $accent; }
}
```

## Strings

- Enclosed in double quotes: `text: "Save and close";`
- Escapes are supported: `\"`, `\\`, `\n`, `\t`.
- Characters such as `;`, `:` and `{` inside text are only treated as syntax symbols when they appear outside quotes.
- To show a double quote inside text, write `\"`.

```sil
Label {
    text: "Notice: press \"OK\" to continue";
    height: 16;
}
```

## Numbers and sizes

Numbers are interpreted in GUI units and may carry the `px`, `em`, `pt` or `dp` suffix; the suffix is only a convenience for writing and does not change the value:

```sil
width: 120;
height: 24px;
padding: 8dp;
font-size: 0.9;
```

Negative values are not valid on most size properties (they are treated as invalid and produce a warning). A negative `margin-*` value means "fall back to the default value of `margin`", not a negative margin.

`0` has a special meaning that depends on the property:

- `width: 0` / `height: 0`: the size is left to the layout or to the widget itself. Perpendicular to the layout direction, a value of 0 is stretched by the parent container; along the layout direction a child widget with 0 counts as 0, and only a container itself that is 0 along the layout direction is sized automatically from its content; widgets with a default size (`Button`, `Slider` ...) fall back to that default (see [Layout](layout.md)).
- `x: 0` / `y: 0`: absolute coordinates of zero, only meaningful in absolute layout.

## Colors

The following forms are supported:

| Form | Example | Notes |
| --- | --- | --- |
| `#RGB` | `#5C5` | Each digit expands to two |
| `#RGBA` | `#5C58` | The last digit is alpha |
| `#RRGGBB` | `#5CD65C` | The common form |
| `#RRGGBBAA` | `#0F1215F2` | The last two digits are alpha; `F2` is nearly opaque |
| Color names | `white`, `black`, `red`, `green`, `blue`, `yellow`, `orange`, `purple`, `pink`, `cyan`, `magenta`, `gray` / `grey` | Fixed shades |
| Transparent | `transparent`, `none` | Fully transparent; `none` also clears optional effects such as shadows and borders |

Alpha multiplies with the `opacity` property: a widget with `#FFFFFF80` and `opacity: 0.5` is drawn at about 25% alpha in the end.

## Booleans

Besides `true` / `false`, `1` / `0`, `yes` / `no` and `on` / `off` also work as booleans, case-insensitively. They are common on properties such as `visible`, `disabled`, `wrap` and `smooth`.

## Comments

Two kinds of comments are supported:

```sil
// Single-line comment

/*
   Multi-line comment
   /* nesting is not supported */
*/
```

Comments may appear at the top level, inside a node, between properties, and between statements in an event script.

## The `name` property

`name` (also accepted as `id`) gives a node an identifier, so that C++ can find it with `FindByName`, or so that tests and logs can recognize it:

```sil
Container {
    name: settings_panel;
    // ...
}
```

Names do not take part in layout and do not automatically become data keys. Keep them unique across the whole tree; duplicates do not raise an error, but lookup only returns the first match.

## Property override order

A single property may come from several places. The effective order is (later overrides earlier):

1. Widget defaults.
2. `Style` base styles (the bottom of the `extends` chain).
3. Derived styles.
4. Inline properties / bindings / templates in the node.

Styles and bindings override by "key", and `margin-left` and `marginLeft` are compared as the same key for precedence. When actually applying properties, however, still write the canonical `margin-left` form, otherwise the widget does not recognize it.

## Common syntax errors

| Log message | Cause | Fix |
| --- | --- | --- |
| `expected ':' after ...` | Missing colon after the property name | Add `: ` |
| `missing ';' after property '...'` | Missing semicolon before a new property on the next line | End every property with `;` |
| `expected '{' after ...` | Missing braces after a node or definition | Add `{ ... }` |
| `property '...' expects a value, not a block` | `{ ... }` written on a non-event property | Use a literal value or a binding |
| `unknown component '...'` | The type name is misspelled, or the component is not defined | Check case and the definition |
| `unknown property '...' on '...'` | The widget has no such property | See the [widget reference](../widgets.md) |

!!! note "Errors versus warnings"
    Syntax errors keep a document from compiling; unknown properties, unknown styles, missing component parameters and the like are only warnings, and the rest of the UI is still built. The log distinguishes them with `error:` / `warning:`.

---

[Previous: DSL tutorial navigation](index.md) · [Next: Layout](layout.md)
