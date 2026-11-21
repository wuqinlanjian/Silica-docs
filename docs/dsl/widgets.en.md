# Common widgets

This page covers the built-in widgets in a take-it-and-use-it way: when to use each one, what the minimal form looks like, and which behaviors tend to trip you up. For the complete property tables, see the [widget reference](../widgets.md).

## At a glance

| Widget | Purpose | Interaction |
| --- | --- | --- |
| `Container` | Layout container | Does not respond to clicks itself (can host a `Part`) |
| `Panel` | Solid color block / border / divider | No child widgets |
| `Label` | Single-line or wrapped text | None |
| `Button` | Button | Click, Enter/Space activation |
| `Checkbox` | Checkbox or switch | Click, Space toggles |
| `Slider` | Continuous numeric value | Drag, arrow keys |
| `TextBox` | Single-line text input | Click to focus, type, Enter to submit |
| `Dropdown` | Drop-down selection | Click to expand, select, Esc to collapse |
| `ScrollView` | Clipped scrolling area | Wheel, drag the scrollbar |
| `Image` | Texture or image file | None |
| `Part` | Decoration layer | Not laid out, does not respond to clicks |
| `ForEach` | Generates rows from an array | Widgets inside each row interact individually |

The examples below assume a vertical `Container` and omit the outer container.

## Label

Displays a single line of text. By default it does not wrap; text that is too wide is truncated and gets an ellipsis.

```sil
Label {
    text: "Health: {{@health}} / 100";
    height: 16;
    font-size: 0.85;
    color: #C8CFD5;
}
```

When you need multiple lines, enable `wrap` and give a height; the height determines how many lines are shown at most, and anything beyond it is not drawn:

```sil
Label {
    text: "This is a longer explanation that wraps at the widget width once wrap is enabled.";
    width: 260;
    height: 40;
    font-size: 0.6;
    wrap: true;
    color: #87919A;
}
```

- When the height is `0`, a plain Label takes the height of one line; `wrap: true` takes the height of three lines.
- `text-align` accepts `left` / `center` / `right`.
- Common properties such as color, font, and opacity apply to the text.

## Button

```sil
Button {
    text: "Apply settings";
    width: 112;
    height: 24;
    background: #2F6B37;
    border: 1 solid #5CD65C;
    color: #D9F2D9;
    hover-background: #3C8446;
    pressed-background: #275B2E;
    onClick: "apply_settings";
}
```

- Click detection: press inside the button and release inside the button.
- An event can hold a callback name (a string) or a script (`{ ... }`). The script wins; when a script is present, a C++ callback with the same name is not called.
- The default hover and pressed highlights are overridden by `hover-background` / `pressed-background`.
- Once selected through the focus ring, Enter or Space triggers a click.

## Checkbox

Use it as a checkbox, or hide the box and use it as a switch:

```sil
Checkbox {
    text: "Enable green theme";
    width: 150;
    height: 18;
    checked: @theme_green;
    accent: #5CD65C;
    onChange: { @theme_color = event.checked ? "#5CD65C" : "#E05252"; };
}
```

- `checked` is a boolean binding; when the user toggles it, the value is written back to the bound data key (two-way binding).
- In an event script, read `event.checked` (boolean) or `event.value`.
- When you use a callback name, the value is passed as the text `true` / `false`.
- `box-size`, `text-gap`, `check-inset`, `show-box: false`, and `accent` control the appearance.
- A hidden box plus a `Part` makes a toggle switch; see the [examples gallery](../examples.md#switch).

## Slider

```sil
Slider {
    width: 220;
    height: 20;
    min: 0;
    max: 100;
    value: @health;
    step: 1;
    accent: #5CD65C;
    smooth: true;
    onChange: { @health = event.value; };
}
```

- Updates continuously while dragging; `step` snaps to the nearest tick; `smooth` makes the handle follow the value smoothly.
- In an event script, read `event.value` (a number).
- When you use a callback name, the value is passed as formatted text (such as `42.5`).
- Once focused, the left and right arrow keys adjust it.

## TextBox

```sil
TextBox {
    width: 180;
    height: 22;
    text: @player_name;
    placeholder: "Enter a name";
    max-length: 32;
    onChange: { @player_name = event.text; };
    onSubmit: "rename_player";
}
```

- Click to focus and start typing; `Backspace` deletes, `Esc` gives up focus.
- `onChange` fires on every text change; when focus is lost and the text changed, it notifies once more.
- `onSubmit` fires on Enter, and the script reads `event.text`.
- `max-length` truncates by bytes, while multi-byte characters such as CJK are handled on whole-character boundaries.
- While focused, arrow keys and character keys are consumed by the text box and do not trigger host shortcuts.

!!! note "Input methods and the cursor"
    Silica draws the cursor itself and does not take over the system input method candidate window. For complex input such as CJK, the host should provide usable character events or take over the text box; the framework only receives character-by-character input.

## Dropdown

```sil
Dropdown {
    width: 160;
    height: 24;
    options: "Low, Medium, High";
    selected: @quality;
    onChange: { @quality = event.index; };
}
```

- `options` is a comma-separated plain-text list; leading and trailing whitespace is stripped; an option cannot contain a comma.
- `selected` (aliases `index` / `value`) is an index starting at `0`; when bound to a data key, the user's choice is written back.
- When expanded, click a list item to select it; click outside or press `Esc` to collapse; losing focus collapses it automatically.
- The expanded list can extend beyond the widget's original height, and hit testing expands with it.
- Event scripts read `event.index` / `event.value` (numbers) and `event.text` (the option text); a callback name receives the index as text.

## ScrollView

```sil
ScrollView {
    name: scroll_area;
    width: 0;
    height: 240;
    padding: 6;
    scroll-step: 28;
    scrollbar-width: 3;
    scrollbar-inset: 3;
    scrollbar-radius: 0;

    // Child nodes go into the vertical content area and are arranged top to bottom
    Label { text: "First row"; height: 16; margin-bottom: 4; }
    Label { text: "Second row"; height: 16; margin-bottom: 4; }
    // ...
}
```

- Child widgets go into an internal content container and are always arranged vertically; a `layout` written on the `ScrollView` is ignored.
- Content outside the viewport is not drawn and cannot be clicked.
- Wheel scrolling and scrollbar dragging both change a target offset first, which eases toward the target each frame; `smooth-scroll: false` jumps immediately.
- `show-scrollbar: false` hides the scrollbar, and the wheel still works.
- The content height is computed from the child widgets automatically; do not give the internal content a fixed height.

## Image

```sil
Image {
    src: "textures/items/diamond";
    width: 32;
    height: 32;
}

Image {
    src: "file:ui/logo.png";
    width: 24;
    height: 24;
}
```

- `src` can be an in-game texture path or an image file prefixed with `file:` (absolute paths and relative `.png` / `.tga` paths are also accepted).
- The image is stretched to the widget rectangle without preserving the aspect ratio; compute the width and height yourself when you need to preserve it.
- File images are copied by Silica into the generated resource pack, so the path must be inside the mod root directory.
- When loading fails, it draws a dark placeholder block and logs once.

See [Images](images.md) for details.

## Part

`Part` is a decoration layer attached to any widget: it does not participate in layout, sticks to the host by anchor every frame, and can be shown or hidden based on the host's state. It suits highlight bars, badges, switch handles, and texture skins.

```sil
Button {
    name: fav_button;
    text: "Favorite";
    width: 96;
    height: 24;

    Part {
        anchor: right;
        width: 20;
        height: 20;
        inset-right: 2;
        src: "textures/ui/icon_star";
        tint: #FFD75A;
        when: "hovered";
    }
}
```

Common anchors: `fill`, `top`, `bottom`, `left`, `right`, `center`, and the four corners. `inset` controls the inset (1-4 numbers following the CSS top/right/bottom/left rule), and `dx` / `dy` apply an extra offset.

`when` separates several conditions with spaces, and the part is shown only when all of them hold; `!` negates:

```sil
Part { anchor: left; width: 2; background: #5CD65C; when: "checked"; layer: under; }
Part { anchor: fill; background: #FFFFFF07; when: "hovered !pressed"; layer: under; }
```

- The available states depend on the host widget: every widget has `hovered`, `pressed`, `focused`, `visible`, `disabled`; `Checkbox` additionally has `checked`, `Dropdown` has `open` / `collapsed`, and `TextBox` has `empty` / `filled` / `editing`.
- `layer: under` draws below the host's own content; by default the part draws above it.
- `Part` itself does not receive clicks; it is only a visual layer.

## Dividing responsibility between widgets and data

Widgets only handle display and user input and do not hold business rules. Recommended:

- UI state (the current page, whether something is expanded) can live in `DataStore` and be changed by scripts.
- Business state (inventory contents, configuration entries) is written to `DataStore` by C++, and the UI only reads it through bindings.
- Input values that must persist across frames are written back through two-way bindings instead of polling widgets from C++.

The next chapter covers the full rules for bindings.

---

[Previous: Layout](layout.md) · [Next: Reactive bindings](bindings.md)
