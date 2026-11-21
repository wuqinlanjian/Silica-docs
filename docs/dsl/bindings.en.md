# Reactive bindings

Bindings let the UI follow data changes: in `.sil` you write a value as an expression starting with `@`, and whenever the data updates the widget re-evaluates automatically, so you no longer have to remember "which Label to update" in C++.

## Basic form

```sil
Label {
    text: @player_name;
    color: @online ? "#5CD65C" : "#87919A";
}

TextBox {
    text: @search_text;
}
```

The parsing rule is simple: **if a property value starts with `@`, the whole value is the expression**. So all of the following are bindings:

```sil
text: @health;
text: @health > 0 ? "alive" : "dead";
width: 120 + @padding * 2;
visible: @show_hint;
background: @theme == "dark" ? "#171B1F" : "#E8EAED";
```

The `@` inside an expression is just part of the name; an `@` inside a string is not treated as a binding.

## Where names come from

A name is resolved in the following order:

1. **Event payload**: `event.value`, `event.checked`, `event.text`, `event.index`, `event.name`, which exist only while an event script is running.
2. **Local scope**: the `item` / `index` injected by `ForEach`, and the parameter names injected by `Component`.
3. **`DataStore`**: global data keys.
4. **Widget state**: `hovered`, `pressed`, `focused`, `visible`, `disabled`, and the widget's own states.

```sil
Label {
    // health comes from DataStore
    text: @health;

    // checked is the Checkbox's own state, not a data key
    color: @checked ? "#5CD65C" : "#C8CFD5";
}
```

!!! warning "Same-name shadowing"
    If `DataStore` also contains `checked`, it shadows the Checkbox's `checked` state. Prefer domain-specific data key names (`sound_enabled`, `theme_dark`) and avoid names that collide with widget state.

## Expression syntax

The following constructs are supported:

| Category | Form |
| --- | --- |
| Literals | Numbers `42`, `3.14`; strings `"text"`; `true` / `false` / `null` |
| Names | `@health`, `@item.name`, `$local`, `event.value` |
| Field paths | `@item.name`, `@config.audio.volume`, `@tags.0` |
| Arithmetic | `+ - * / %` |
| Comparison | `== != > < >= <=` |
| Logic | Unary `!`; conditionals use the ternary `? :` |
| Grouping | `( ... )` |

Examples:

```sil
text: @count + " items";
color: @health >= 80 ? "#5CD65C" : (@health > 0 ? "#E0A23A" : "#E05252");
visible: !@compact;
```

!!! note "No && or ||"
    Expressions only provide unary `!` and the ternary operator. Use nested ternaries when you need compound conditions:

    ```sil
    visible: @logged_in ? (@name != "" ? true : false) : false;
    ```

    Complex conditions are better placed in event scripts, or condensed by C++ into a single boolean data key.

### Types and conversion

- When a number and a string are added, concatenation applies if either side is a string: `@count + " items"`.
- Comparison is lexicographic when both sides are strings; otherwise it compares numerically.
- Truthiness: a non-zero number is true; a string is true when it is non-empty and is not `"false"` / `"0"`; a non-empty array or object is true; `null` is false.
- When a binding result is written to a property: numbers are output in the usual format, booleans output `true` / `false`, and `null` **does not overwrite** the property's current value (it waits for the data to arrive).

### Field paths and arrays

`@item.name` resolves `item` first, then takes fields one by one. Arrays are accessed with numeric indices:

```sil
text: "{{@items.0.name}}";
text: @row.tags.1;
```

A path that does not exist yields `null`; it does not throw and does not crash the UI.

## Templates {#templates}

When a value is assembled from several text fragments and expressions, use `{{ ... }}`:

```sil
Label {
    text: "HP: {{@health}} / 100 ({{@health > 0 ? "alive" : "dead"}})";
}
```

- Inside `{{ }}` is a complete expression, and all operators are available.
- Text outside the braces is kept as is.
- Each expression segment is parsed once; later evaluations only substitute the results.
- When a data key referenced in the template changes, the whole string is reassembled.

If the literal text itself must contain `{{`, there is currently no escape syntax; concatenate it in a script instead, or split the text across several Labels.

## Conditional values

You can pick a value based on state with a ternary expression:

```sil
Button {
    background: @selected ? "#2F6B37" : "#1B2025";
    border-color: @disabled ? "#3A4046" : "#3C5A44";
}

Panel {
    // you can also use widget state directly in the condition; a leading ! negates it
    background: !hovered ? "#171B1F" : "#232A30";
}
```

This form suits simple switches such as color, opacity, and text. Properties are re-applied when state changes, and properties with a `transition` configured transition smoothly (see [Animations and transitions](animations.md)).

!!! tip "State conditions vs. data conditions"
    A condition starting with `@` is handled as a binding; a condition starting with `!` (such as `!hovered`) is handled as a dynamic rule. Both refresh when state changes, so in normal use you do not need to tell them apart.

## Two-way bindings

Most bindings are read-only: the data determines what the widget displays. For properties the **user edits directly**, a binding can also write user input back to `DataStore`:

| Widget | Property that can be written back | Value written |
| --- | --- | --- |
| `Checkbox` | `checked` | boolean |
| `Slider` | `value` | number |
| `TextBox` | `text` | string |
| `Dropdown` | `selected` / `index` / `value` | numeric index |

```sil
Checkbox {
    text: "Mute";
    checked: @muted;          // clicking writes muted
}

Slider {
    min: 0; max: 100;
    value: @volume;           // dragging writes volume
}

TextBox {
    text: @player_name;       // typing writes player_name
}
```

Write-back happens only on **user actions**; changing data from C++ through `DataStore::Set` does not trigger write-back in the other direction.

!!! warning "Only a single data key can be written back"
    Write-back requires the binding expression to be a single name, for example `checked: @muted`. Computed bindings such as `checked: @muted && ...` (unsupported) or `text: @first + @last` are read-only: user input still changes the widget, but nothing is written back to any key.

## When bindings refresh

| Source of change | Behavior |
| --- | --- |
| `DataStore::Set(key, value)` | When the value actually changes, widgets that read the key are notified immediately and re-evaluate synchronously |
| Widget state (`hovered`, `checked` ...) | Checked once per `Tick`; re-evaluated after a change |
| Templates | Same as bindings: reassembled when a data key or state changes |
| Subtrees with `visible: false` | Keep their data subscriptions; when shown again they use the latest values |

`DataStore` notifications run synchronously on the writing thread. If the host writes data from another thread, it should synchronize with the render thread itself and avoid touching the widget tree at the same time inside the notification callback.

## Limits of bindings

- **The property must exist**: `cannot bind property '...' on '...'` means the widget does not have that property, or the expression cannot be parsed.
- **Expression parse failure**: the binding is skipped and the widget keeps its previous property value; the log carries line and column numbers.
- **Do not animate by binding a size**: `width: @value` changes the layout but is not interpolated automatically; for a smooth result use properties supported by `transition`, or the slider's own `smooth`.
- **Avoid creating a new array every frame**: `ForEach` detects changes through whole-array replacement, so you neither need nor can modify the array in place.

## Common mistakes

| Symptom | Cause | Fix |
| --- | --- | --- |
| A binding never updates | The data key was never `Set`, and no widget state has the same name | Make sure C++ writes the same key name |
| `=` raises an error | The expression uses an assignment sign | Use `==` for comparison instead |
| Text shows `"false"` | A boolean was bound directly to text | That is normal output; use a ternary when you need other text |
| A computed binding does not write back | The binding is an expression rather than a single key | Change it to `property: @key` and put the computation in a script or C++ |
| A style change has no effect | The property name is misspelled, or the value type is invalid | Check the log for `unknown property` or binding warnings |

---

[Previous: Common widgets](widgets.md) · [Next: Event scripts](scripts.md)
