# Widget reference

This page lists each widget's role, defaults, full property table, and states/events. For usage tutorials, see [DSL widgets](dsl/widgets.md).

## Widget overview

| Widget | Role | Default size | Accepts |
| --- | --- | --- | --- |
| `Container` | Layout container that can paint a background | 0×0 (grows as needed) | Any widget |
| `Panel` | Purely decorative box | 0×0 | `Part` only |
| `Label` | Single-line / multi-line text | Width 0, height 14×font size | `Part` only |
| `Button` | Clickable button | 120×20 | `Part` only |
| `Image` | Texture | 16×16 | `Part` only |
| `Checkbox` | Checkbox / switch | Height 18 | `Part` only |
| `Slider` | Numeric slider | Height 20 | `Part` only |
| `TextBox` | Single-line input | Height 22 | `Part` only |
| `Dropdown` | Drop-down selection | 160×24 | `Part` only |
| `ScrollView` | Vertically scrolling, clipping viewport | 0×0 | Any widget (goes into the inner content container) |
| `ForEach` | Array repeater | 0×0 (height adapts) | Any widget (as a row template) |
| `Part` | Decoration layer attached to a widget | Follows the anchor | None |

!!! warning "Only containers can hold arbitrary child widgets"
    `Panel`, `Button`, `Label`, `Image`, `Checkbox`, `Slider`, `TextBox`, and `Dropdown` accept only `Part` as a child node. Putting any other widget inside raises the error `'X' cannot contain 'Y'`.

## Common properties

All widgets accept the following properties:

| Property | Type | Description |
| --- | --- | --- |
| `name` / `id` | text | Node name, used by `FindByName` |
| `x` / `y` | number | Absolute position coordinates (a layout container overrides them) |
| `width` / `height` | number | Size; 0 means the layout decides |
| `margin` | number | Trailing spacing (bottom of a column, right of a row) |
| `margin-top` / `margin-right` / `margin-bottom` / `margin-left` | number | Per-side spacing; use `margin-top`/`margin-left` for the leading side |
| `visible` | boolean | Excluded from layout, not rendered, not interactive |
| `disabled` / `enabled` | boolean | Disabled state; child widgets inherit the disabled state |
| `z-order` / `z` / `layer` / `z-index` | number | Draw and hit testing order; larger values are on top |
| `opacity` | 0-1 | Overall opacity, takes part in transitions |
| `background` / `bg` | color/gradient | Fill |
| `color` / `text-color` | color | Text color; also the tint of images |
| `radius` / `border-radius` | number | Corner radius |
| `border-width` | number | Stroke width |
| `border-color` | color | Stroke color |
| `border` | `width [color]` | Sets the stroke as well |
| `shadow` | `...` | Shadow; see [Images](dsl/images.md#shadow) |
| `gradient` / `bg-gradient` | gradient | Background gradient |
| `border-image` / `nine-slice` | texture slice... | Nine-slice border |
| `font-family` | text | Font name or `default`/`smooth`/`unicode`/`rune`/`ui` |
| `text-align` / `align` | `left`/`center`/`right` | Text alignment |
| `transition` | list | Color/opacity transitions |
| `animation` | `name duration [count]` | Keyframe animation |
| `class` / `style` | list of style names | Applies global styles |
| `on<event>` / `<event>:` | script or handler name | See [Events](#events) |

`padding` only matters for container layout (`Container`, `ScrollView`, `ForEach`); leaf widgets use their own text padding properties.

## Container

Layout container. Defaults to `absolute` and can paint a background, stroke, and corner radius.

| Property | Type | Description |
| --- | --- | --- |
| `layout` / `direction` | `vertical`/`column`/`v`, `horizontal`/`row`/`h`, `absolute`/`none` | Arrangement, default `absolute` |
| `padding` | number | Padding on all four sides |

```sil
Container {
    layout: vertical;
    padding: 8;
    width: 200;

    Label { text: "Title"; height: 20; }
    Label { text: "Description"; height: 16; }
}
```

- In a vertical layout, a child widget with width 0 stretches to the content width.
- In a horizontal layout, a child widget with height 0 stretches to the content height.
- When the container size on the main axis is 0, it is computed from the accumulated child sizes.
- Hidden child widgets take no space.

## Panel

A pure box that paints a background, stroke, corner radius, shadow, and nine-slice border; it cannot lay out child widgets.

| Property | Description |
| --- | --- |
| Common appearance properties | All available |
| Child nodes | `Part` only |

```sil
Panel {
    x: 10; y: 10;
    width: 120; height: 60;
    background: #2C2F33;
    radius: 6;
}
```

## Label

Text widget. Single-line by default, truncating text wider than the widget; with `wrap` enabled it wraps automatically.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `text` / `value` | text/binding | empty | Displayed content |
| `font-size` / `font` | number | 1 | Font size multiplier |
| `wrap` / `word-wrap` | boolean | `false` | Multi-line wrapping |
| `color` | color | near white | Text color |
| `text-align` | enum | `left` | Alignment |

- Height 0: single-line mode uses `14×font size`; wrapping mode uses the height of three lines (`30×font size`).
- Wrapping breaks at spaces first; without spaces (Chinese text, long words) it breaks between characters.
- For `{{ }}` text templates, see [Reactive bindings](dsl/bindings.md#templates).

## Button

Button, using `UIStyle::PrimaryButton()` by default. A click (mouse released inside it, or the activate key pressed while focused) fires `click`.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `text` / `value` | text | empty | Button label |
| `font-size` | number | 1 | Font size |
| `onClick` / `on-click` / `action` | handler name | none | C++ named handler |
| `onClick: { ... }` | script | none | The script takes precedence over the named handler |
| State appearance | | | `hover-background`/`pressed-background`/`disabled-*` |

Without state colors set, hover blends in 10% white and press blends in 18% black.

## Image

See [Images](dsl/images.md). Main properties:

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `src` / `source` / `texture` | text | empty | Game texture path or image file path |
| `width` / `height` | number | 16 | Stretched to the rectangle; 0 falls back to 16 |
| `color` | color | white | Multiplicative tint |

## Checkbox

Checkbox. Click, space, or Enter toggles it, and `change` reports the new state.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `text` / `label` | text | empty | Caption |
| `checked` / `value` | boolean | `false` | Checked state (a single-identifier binding can write back) |
| `accent` / `check-color` | color | green | Check mark color |
| `box-size` | number | 0 (follows font size) | Box side length; 0 uses `max(14, 14×font size)` |
| `text-gap` / `gap` | number | 8 | Gap between box and text |
| `check-inset` | number | -1 | Check mark inset; negative values are proportional |
| `show-box` | boolean | `true` | Hides the built-in box and leaves drawing to a Part |
| `font-size` | number | .9 | Font size |
| `onChange` | handler name | none | Receives `"true"`/`"false"` |
| `onChange: { ... }` | script | none | Takes precedence over the handler |

States: `checked`/`on`/`selected`, `unchecked`/`off`, plus the common `hovered`, `pressed`, `focused`, and `disabled`.

## Slider

Numeric slider; clicking jumps to a position and dragging updates continuously, quantized to `step` if set.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `min` | number | 0 | Lower bound |
| `max` | number | 100 | Upper bound |
| `value` | number | 50 | Current value (a single-identifier binding can write back) |
| `step` | number | 0 | 0 means continuous |
| `knob-size` | number | 12 | Knob diameter |
| `accent` / `fill-color` | color | green | Color of the filled track |
| `track-height` | number | 4 | Track height |
| `track-color` | color | dark gray | Track background color |
| `knob-color` | color | light gray | Knob color |
| `knob-active-color` | color | near white | Knob color while dragging |
| `smooth` / `animate` | boolean | `false` | The knob eases toward the target value (dragging eases too) |
| `smoothing` / `smooth-speed` | number | 20 | Easing rate (approach factor per second) |
| `onChange` | handler name | none | Receives the formatted value |
| `onChange: { ... }` | script | none | Takes precedence over the handler |

## TextBox

Single-line input box. Click to focus, then type and backspace; `Esc` drops focus and `Enter` submits.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `text` / `value` | text | empty | Content (a single-identifier binding can write back) |
| `placeholder` / `hint` | text | empty | Placeholder hint |
| `max-length` | number | 32 | Maximum number of characters |
| `font-size` | number | .9 | Font size |
| `text-padding` / `padding-x` | number | 5 | Horizontal text padding |
| `caret-color` | color | white | Caret color |
| `caret-width` | number | 1.5 | Caret width |
| `placeholder-color` | color | gray | Placeholder text color |
| `onChange` | handler name | none | On every text change |
| `onSubmit` / `onEnter` | handler name | none | Submitted with Enter |
| `onChange: { ... }` / `onSubmit: { ... }` | script | none | Take precedence over the handlers |

States: `empty`/`blank`, `filled`/`has-text`, `editing`, plus the common states.

## Dropdown

Drop-down selection box. Click to expand; choosing an option collapses it and reports the index through `change`.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `options` / `items` / `choices` | comma-separated text | empty | Options; an option cannot contain a comma |
| `selected` / `index` / `value` | number | 0 | Current index (a single-identifier binding can write back) |
| `item-height` | number | 20 | List row height |
| `item-padding` | number | 8 | Horizontal padding inside a row |
| `font-size` | number | .9 | Font size |
| `accent` | color | green | Also sets the selected background |
| `selection-color` | color | translucent green | Selected row background |
| `arrow` | one or two glyphs | `v` | Collapsed state, expanded state |
| `arrow-closed` / `arrow-open` | text | `v` | Set the arrows separately |
| `arrow-color` | color | light gray | Arrow color |
| `list-background` | color | dark | Expanded list background |
| `open` / `expanded` | boolean | `false` | Initial expanded state |
| `onChange` | handler name | none | Receives the index |
| `onChange: { ... }` | script | none | Takes precedence over the handler |

- Default `z-order: 100`, so the expanded list covers sibling nodes behind it.
- While expanded, the hit test area includes the list area.
- Clicking elsewhere or losing focus collapses it.
- States: `expanded`/`open`, `collapsed`/`closed`.

## ScrollView

Vertically clipping scrolling viewport. Child widgets go into an inner content container (arranged vertically); content outside the viewport is neither drawn nor clickable.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `scroll-step` / `wheel-step` | number | 24 | Distance per wheel notch |
| `smooth-scroll` | boolean | `true` | Whether scrolling eases |
| `scroll-smoothing` / `scroll-easing` | number | 22 | Easing rate |
| `show-scrollbar` / `scrollbar` | boolean | `true` | Show the scrollbar |
| `scrollbar-width` | number | 3 | Scrollbar width |
| `scrollbar-inset` | number | 3 | Distance from the right edge |
| `scrollbar-radius` | number | 0 | Scrollbar corner radius |
| `scrollbar-color` | color | 10% white | Track color |
| `scrollbar-thumb-color` | color | 38% white | Thumb color |
| `offset` / `scroll-y` | number | 0 | Initial scroll offset |
| `padding` | number | 0 | Viewport padding |

- The `layout` property is accepted but ignored: content is always arranged vertically.
- `scrollOffsetY` eases toward `scrollTargetY`; the wheel and the scrollbar only change the target value.
- The scrollbar can be grabbed and dragged.
- The content height accumulates from child widgets; you do not need to set it manually.

```sil
ScrollView {
    width: 240;
    height: 180;
    scroll-step: 32;

    ForEach (data: @items, item: @item) {
        Label { text: @item.name; height: 24; }
    }
}
```

## ForEach

Array repeater; see [Lists and loops](dsl/lists.md) for details.

| Parameter | Required | Description |
| --- | --- | --- |
| `data` | Yes | Expression that produces an array |
| `item` | No | Row element variable name, default `item` |
| `index` | No | Row index variable name, default `index` |

| Property | Default | Description |
| --- | --- | --- |
| `layout` | `vertical` | How rows are arranged |
| `height` | 0 | 0 means accumulate over rows; an explicit value wins |
| Other container properties | | Apply to the repeater itself |

- At most 512 rows per pass; the excess is dropped and logged.
- After a data change, all rows are rebuilt on the next `Update`.

## Part

A decoration layer attached to any widget; it implements skins such as hover, selected, and switch states with structure instead of images.

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `anchor` / `attach` | see the table below | `fill` | Position relative to the host |
| `inset` | 1-4 numbers | 0 | Inset: all sides / vertical and horizontal |
| `inset-x` / `inset-y` | number | 0 | Horizontal/vertical inset |
| `inset-top` / `inset-right` / `inset-bottom` / `inset-left` | number | 0 | Single-side inset |
| `offset-x` / `offset-y` | number | 0 | Fine position adjustment |
| `layer` | `under`/`over` | `over` | Draw beneath or above the host |
| `when` / `if` | condition string | empty | Space-separated; `!` negates |
| `src` | text | empty | Texture |
| `tint` | color | white | Texture tint |
| `text` | text | empty | Centered text |
| `font-size` | number | 1 | Text font size |
| `width` / `height` | number | follows the anchor | Actual size when the anchor is not `fill` |
| `visible` / `show` / `enabled` | boolean | `true` | Static enable/disable |
| Common appearance properties | | | `background`, `radius`, `border-*`, and so on |

Anchor values:

| Value | Position |
| --- | --- |
| `fill` / `stretch` | Fills the host inner box, ignoring width and height |
| `top` | Top edge, full width by default |
| `bottom` | Bottom edge, full width by default |
| `left` | Left side, vertically centered |
| `right` | Right side, vertically centered |
| `center` | Centered |
| `top-left` / `top-right` / `bottom-left` / `bottom-right` | The four corners |

Host states available to `when`:

| Host | States |
| --- | --- |
| All | `hovered`/`hover`, `pressed`/`active`, `focused`/`focus`, `visible`, `disabled` |
| `Checkbox` | `checked`/`on`/`selected`, `unchecked`/`off` |
| `Dropdown` | `expanded`/`open`, `collapsed`/`closed` |
| `TextBox` | `empty`/`blank`, `filled`/`has-text`, `editing` |

```sil
Checkbox {
    text: "Night mode";
    box-size: 40;
    show-box: false;

    Part { anchor: left; width: 40; height: 20; radius: 10; background: #33363A; when: "!checked"; }
    Part { anchor: left; width: 40; height: 20; radius: 10; background: #498FC3; when: "checked"; }
}
```

A Part does not receive mouse events; hover and pressed states follow the host.

## Events {#events}

| Event key | Fires on | Payload |
| --- | --- | --- |
| `onClick:` | Button click | `event.name = "click"` |
| `onChange:` | Value change | `event.name/value/text/checked/index`, filled in per widget |
| `onSubmit:` | Enter in an input box | `event.name = "submit"`, with `event.text/value` as the text |

Event keys must have the `on` prefix (`action` is an alias for `onClick`), and differences in case and hyphens are ignored: `onChange` = `on-change` = `ONCHANGE`. When the value is `{ ... }`, it runs as a script; otherwise it is registered as a C++ named handler name. When a script is present, the C++ handler is no longer called.

For script syntax, payload fields, and write-back rules, see [Event scripts](dsl/scripts.md).

## Next steps

- [Reactive bindings](dsl/bindings.md)
- [Event scripts](dsl/scripts.md)
- [C++ integration](cpp-guide.md)
- [Examples](examples.md)
