# Components

`Component` gives a reusable subtree a name. Its division of labor with `Style` is strict:

| Construct | What it can hold | Purpose |
| --- | --- | --- |
| `Style` | Properties only | Share appearance |
| `Component` | Nodes, parameters, slots | Share a whole structural block |

Each `Component` definition is visible only inside the current `.sil` document. Instantiating it really creates a widget subtree (it does not copy text), so you can customize it freely with parameters and slots.

## Defining a component

```sil
Component SettingRow(title, detail: "Default description") {
    Container {
        layout: horizontal;
        height: 40;
        padding: 8;

        Container {
            layout: vertical;
            width: 220;        // in a horizontal row the main axis size is decided by the child widget itself, so an explicit width is required
            Label { text: @title; height: 18; }
            Label {
                text: @detail;
                height: 14;
                font-size: .8;
                color: #999;
            }
        }
    }
}
```

Syntax notes:

- `Component name(parameter list) { ... }`; the parentheses after the name are optional.
- Parameters are written as `name` or `name: default`, separated by commas.
- A default value is **raw text**; it can reference `$variables` but is not evaluated.
- A component body must start with a node; you cannot write properties directly. Properties belong to the nodes inside the body.

!!! warning "Component names are case-sensitive"
    Type lookup is case-insensitive (`SettingRow` and `settingrow` both match), but keep the spelling consistent within a document so it stays easy to read and search.

## Parameters

Inside the component body, parameters are referenced as `@parameter`:

```sil
Component Stat(label, value) {
    Container {
        layout: vertical;
        Label { text: @label; color: #AAA; }
        Label { text: @value; font-size: 1.4; }
    }
}
```

Pass parameters when you instantiate:

```sil
Stat(label: "Online players", value: @playerCount);
```

Parameter text first expands `$variables`, then is classified by the shape of the value:

| Passed as | Meaning when referenced in the body |
| --- | --- |
| `@playerCount` | Expression: evaluated in the **caller's** scope and in the `DataStore` |
| `"Online players"` | Literal text (kept quoted in bindings) |
| `true` / `false` | Boolean literal |
| `null` / `none` | Null value |
| `12` / `3.5` | Numeric literal |
| Anything else | Plain text |

Substitution happens before expression parsing. For bindings, scripts, and templates the result is "expression text"; for ordinary properties it is "literal text". That is exactly why `text: @title` is both a valid binding inside the body and able to accept a string parameter.

!!! note "Parameters are not reactive variables"
    `@title` is replaced at compile time by the expression the caller passes. If the caller passes `@playerCount`, the body ends up with `text: @playerCount`, which is still an ordinary reactive binding; if the caller passes a fixed string, it is a constant. The parameter itself never changes at runtime.

### Defaults and missing parameters

```sil
Component Badge(text, tone: "#498FC3") { ... }
Badge(text: "Beta");            // tone uses its default value
```

- Omitting a parameter that has a default: the default value is used.
- Omitting a parameter without a default: the warning `component 'X' is missing the argument 'Y'` is produced, and that parameter expands to empty.
- Passing an undeclared parameter: the warning `unknown argument 'Y' on component 'X'` is produced, and the argument is ignored.

### How `$variables` differ from parameters

| Form | When it expands | Where it applies |
| --- | --- | --- |
| `$color` | Compile-time text substitution | Global `Var`; properties, bindings, templates, parameters, scripts |
| `@param` | Text substitution when the component is instantiated | Component parameters |

The two can be combined: the caller passes `@param` and the component body additionally references `$variables`; in the end everything resolves to concrete text.

## Slot

A `Slot` in the component body marks where the caller's child nodes are inserted:

```sil
Component Card(title) {
    Container {
        layout: vertical;
        padding: 12;
        background: #2A2D31;
        radius: 6;

        Label { text: @title; height: 20; color: #E0E0E0; }
        Slot
    }
}
```

Usage:

```sil
Card(title: "Settings") {
    Label { text: "First item"; }
    Button { text: "Apply"; }
}
```

Key semantics:

- Nodes inserted by a `Slot` are built from the **caller's** scope, parameters, and `DataStore`, not from the component's inner scope.
- Slot children can access the caller's own `@param` and can also use global bindings.
- When the component body contains several `Slot`s, each one inserts a copy of the caller's child nodes. Usually you write only one.
- If the component body has no `Slot` but children are passed, the warning `component 'X' has no Slot; its children are ignored` is produced, and the children are discarded.
- A `Slot` can be nested inside a container within the component body; the slot position follows the container layout.

!!! tip "Composition over inheritance"
    A component cannot rewrite the structure of slot children directly. When you need customization, express the differences as parameters or slot content instead of letting the component "guess" the caller's intent.

## Instance overrides

Properties on the instantiating node are applied to the component's **root node**, overriding the values written in the component definition:

```sil
Card(title: "Warning") {
    width: 240;         // overrides the width of Card's root node
    class: DangerCard;  // overrides the style source of the root node
}
```

You can therefore:

- Adjust geometry (`width`, `height`, `margin`, and so on).
- Add or override appearance with `class:` / `style:`.
- Attach bindings, templates, and event scripts.
- Name the root node with `name:` so `FindByName` can find it.

Only the root node can be overridden; non-root nodes inside the component body are not affected by instance properties.

## Multiple root nodes

A component body can contain several top-level nodes:

```sil
Component TitleBar(left, right) {
    Label { text: @left; }
    Label { text: @right; }
}
```

In that case the compiler automatically wraps them in a vertical `Container`; the two nodes become its children and instance properties apply to that container. If you need a horizontal or other layout, write your own root `Container`:

```sil
Component TitleBar(left, right) {
    Container {
        layout: horizontal;
        height: 24;
        Label { text: @left; width: 180; }
        Label { text: @right; width: 60; text-align: right; }
    }
}
```

!!! warning "Implicit container for multiple root nodes"
    The automatic container uses the default `vertical` layout, zero padding, and no background. When alignment or background matters, an explicit root container is more controllable.

## Recursion and nesting limits

| Case | Result |
| --- | --- |
| A component instantiates itself directly or indirectly | Error `component 'X' instantiates itself recursively`; that instance is discarded |
| Components nest more than 32 levels deep | Error `components nest too deeply (limit 32)` |
| The node tree is deeper than the limit | Error `components nest too deeply` |

Recursion is detected by component name before construction, so infinite expansion never happens at runtime. For deep composition, split the document or use `ForEach` instead.

## Components and events

Event scripts can be written on the instantiating node or on nodes inside the body:

```sil
Component ActionRow(label, key) {
    Button {
        text: @label;
        height: 32;
        onClick: { @selected = @key; }
    }
}
```

- `$name` inside a body script still means a script local variable; it is not treated as a global `Var` or as a parameter.
- A script on the instantiating node takes effect on the root node; however, if the root node is not an interactive widget (for example `Container`), the event never fires. To respond to clicks, write the event on a `Button` or `Checkbox` inside the component, or make the root node an interactive widget.

## Complete example

```sil
Var rows: 0;

Component Counter(label, step: 1) {
    Container {
        layout: horizontal;
        height: 36;
        padding: 6;
        background: #2C2F33;
        radius: 4;

        Label { text: @label; width: 160; height: 24; }
        Label { text: "{{@rows}}"; width: 48; height: 24; color: #9FD3FF; }
        Button {
            text: "+";
            width: 32;
            height: 24;
            onClick: { @rows = @rows + step; }
        }
    }
}

Counter(label: "Processed", step: 2);
Counter(label: "Skipped");
```

Here `step` can use its default value or be overridden by the caller; `@rows` is a global binding, so all instances share the same data.

## Common issues

| Symptom | Cause | Fix |
| --- | --- | --- |
| Component children do not appear | The component body has no `Slot` | Add a `Slot` where you want them |
| A parameter is not substituted in the body | It was written as `$param` or without `@` | Reference parameters as `@name` |
| Instance properties have no effect | The properties were written on the instance's child nodes | Instance properties can only override the root node |
| Layout breaks with multiple root nodes | The automatic container is vertical | Write an explicit root `Container` and set `layout` |
| Clicks do nothing | The root node is not interactive | Put the event on a widget such as a button |

Next: [Global styles](styles.md) · [Animations and transitions](animations.md)
