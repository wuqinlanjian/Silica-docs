# Event scripts

Event scripts keep simple UI interactions inside `.sil`, so you do not need a C++ callback just for a counter. A script is attached to an event property, written as `{ ... }`, and can read and write `DataStore`, do arithmetic, and branch on conditions.

## Basic form

```sil
Button {
    text: "Heal +10";
    width: 96;
    height: 24;
    onClick: { @health += 10; };

    // separate multiple statements with semicolons
    // onClick: { $gain = 10; @health += $gain; if (@health > 100) { @health = 100; } };
}
```

A script runs only when its own event fires:

| Widget | Event | When it fires |
| --- | --- | --- |
| `Button` | `click` | Press and release inside the button, or activate through focus |
| `Checkbox` | `change` | The user toggles the check state |
| `Slider` | `change` | The user changes the value by dragging or with keys |
| `TextBox` | `change` | The text changes |
| `TextBox` | `submit` | Submitted with Enter |
| `Dropdown` | `change` | The user selects a new item |

Event property names are case-insensitive, so `onClick` / `on-click` / `ONCLICK` are the same event; `action` is an alias for `click`.

## Statements

Scripts support three kinds of statements:

1. **Assignment**: `@key = expression;`
2. **Compound assignment**: `@key += expression;`, which also supports `-=`, `*=`, `/=`.
3. **Conditionals**: `if (expression) { ... } else { ... }`, and an `else` can be followed by another `if`.

```sil
Checkbox {
    text: "Enable theme";
    checked: @theme_on;
    onChange: {
        @theme_on = event.checked;
        if (event.checked) {
            @accent = "#5CD65C";
        } else {
            @accent = "#87919A";
        }
    };
}
```

Separate statements with `;`, and the trailing semicolon on the last statement is optional. Scripts support `//` and `/* */` comments.

## Assignment targets

| Form | Target | Lifetime |
| --- | --- | --- |
| `@health = 100;` | The `DataStore` key `health` | Global, across scripts and widgets |
| `health = 100;` | The same (`@` is optional) | Global |
| `$gain = 10;` | A local variable for this execution | Valid only for this one run of the script |

`$` variables are handy for breaking a computation into steps:

```sil
onClick: {
    $step = @shift_held ? 10 : 1;
    @health += $step;
    if (@health > 100) { @health = 100; }
};
```

`$` variables are never written to `DataStore` and do not survive between two events.

!!! warning "$ variables and document-level Var are not the same thing"
    In a property value, `$accent` is a compile-time `Var` substitution; in a script it is a runtime local variable. The two may share a name but do not affect each other.

## Reading values

Expressions in scripts use the same syntax as bindings:

```sil
onClick: {
    $next = @mode + 1;
    if ($next > 2) { $next = 0; }
    @mode = $next;
};
```

You can read the current widget state (`checked`, `disabled`, and so on), other data keys, and the event payload.

## Event payload

While an event fires, a script can read `event`:

| Field | Type | Applies to | Meaning |
| --- | --- | --- | --- |
| `event.name` | string | All | Event name, such as `"click"`, `"change"` |
| `event.value` | depends on the widget | Value events | The new value: boolean, number, or text |
| `event.text` | string | Text events | Related text (typed input, option text) |
| `event.checked` | boolean | `change` | Check state |
| `event.index` | number | `change` (Dropdown) | Selected index |

```sil
Checkbox {
    checked: @theme_on;
    onChange: { @theme_color = event.checked ? "#5CD65C" : "#E05252"; };
};

ForEach (data: @items, item: @item) {
    Button {
        text: "{{@item.name}}";
        onClick: { @selected = @item.id; };
    };
}
```

`event.*` is valid only while the script runs; evaluating it after the event yields `null`.

## Precedence between scripts and C++ callbacks

An event property can hold a script or a callback name:

```sil
Button { onClick: "open_settings"; }        // handled by C++
Button { onClick: { @page = "settings"; }; } // handled by the script
```

When the same event has both a script and a C++ callback, **the script wins and the callback is not called**. That lets you implement something quickly with a script and migrate it to C++ later.

Value events (`change` / `submit`) follow the same order: when a script is present, a registered C++ callback with the same name is not called.

## What scripts can and cannot do

**Can:**

- read and write any `DataStore` key;
- use arithmetic, comparison, ternary expressions, and `if/else`;
- read the local scope (`@item.name`, component parameters) and widget state;
- change several keys in one run; each `Set` immediately notifies bindings to refresh.

**Cannot:**

- call C++ functions or perform arbitrary system operations;
- define functions, loops, or recursion;
- modify widget properties (for example, "make the button red"). For visual changes, write the state into a data key and drive the property with a binding or conditional value;
- use a `$variable` in a script to reference a document-level `Var`.

!!! note "Leave visual changes to bindings"
    Scripts only change data; bindings decide how the UI responds. For example, "the button turns green after being clicked" should be written as:

    ```sil
    Button {
        background: @active ? "#2F6B37" : "#1B2025";
        onClick: { @active = !@active; };
    }
    ```

## Writing and refreshing

`@key = ...` in a script ultimately calls `DataStore::Set`:

1. When the value actually changes, bindings that read the key re-evaluate immediately.
2. Later statements in the same event see the new value when they read the key.
3. Widget drawing reflects the new state on the next frame.

Every widget that reads the same key is notified, so the script of one button can update several display locations at once.

## Where complex logic belongs

| Scenario | Recommendation |
| --- | --- |
| Counting, negation, simple state toggles | Write it in a `.sil` script |
| Needs access to game objects, files, or the network | Register a C++ callback |
| Business state shared by several screens | C++ writes to `DataStore`; the UI only binds |
| Needs loops or complex validation | Handle it in C++ and write the result back to a data key |

## Troubleshooting

| Error / symptom | Cause |
| --- | --- |
| `scripts attach to an event property (onClick, onChange, onSubmit, ...)` | A `{ ... }` was written on a property that is not an event |
| `expected '(' after 'if'` | Missing parenthesis after `if` |
| `expected '=' ... after '...'` | The assignment is missing its operator, or `==` was used as assignment |
| `in 'xxx ...': use '==' to compare; '=' only assigns inside a script` | A single `=` appears in the expression |
| The script does not run | The event never fired, or the widget is `disabled` |
| Data changed but the UI did not | The UI reads a different key, or that property has no binding |

---

[Previous: Reactive bindings](bindings.md) · [Next: Components](components.md)
