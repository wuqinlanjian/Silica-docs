# Layout

Silica computes layout in containers: child widgets do not need to know their absolute coordinates; the container writes position and size into its children in `Update` every frame. Three concepts are enough to lay out the vast majority of UIs: **layout direction**, **size rules**, and **margins and padding**.

## Containers and directions

Only `Container` (and the `ForEach` and `ScrollView` that derive from it) can lay out child widgets. `Panel` is a solid color block that can only carry `Part` decorations and cannot hold child widgets.

The `layout` property of `Container` accepts:

| Value | Meaning |
| --- | --- |
| `vertical`, `column`, `v` | Arrange top to bottom; child widths are stretched by default |
| `horizontal`, `row`, `h` | Arrange left to right; child heights are stretched by default |
| `absolute`, `none` | No layout; child widgets use their own `x` / `y` |

!!! warning "The default is absolute layout"
    The default `layout` of a `Container` is `absolute`. If you forget `layout: vertical;`, all child widgets pile up in the top-left corner. `ForEach` is the exception: it defaults to `vertical`.

```sil
Container {
    layout: vertical;
    width: 280;
    height: 160;
    padding: 10;

    Label { text: "First line"; height: 16; }
    Label { text: "Second line"; height: 16; }
}
```

## Size rules

### Along the layout direction

- A child widget's **main axis size** is decided by the widget itself: `height` in a vertical layout, `width` in a horizontal one.
- If the container's own **main axis size** is `0`, it is computed automatically from the total content length after layout (padding and trailing margins included).

### Perpendicular to the layout direction

- If a child widget's **cross axis size** is `0`, it is stretched to the content width/height of the container.
- A container's own cross axis size is not computed automatically; set it explicitly, or let its parent container stretch it.

So the usual pattern is: in a vertical container write `width: 0;` to let the parent decide the width, and write a fixed `height` or `0` to decide whether it grows with its content.

```sil
// Fill the parent container's width, height follows the content
Container {
    layout: vertical;
    width: 0;
    height: 0;
    padding: 8;
}
```

### Widget default sizes

- When a `Label` has height `0`: in normal mode the height is `14 * font-size`; with `wrap` enabled the default height is 3 lines.
- `Button`, `Checkbox`, `Slider`, `TextBox` and `Dropdown` each have their own default when no height is specified.
- `Image` has no intrinsic size; give it an explicit `width` / `height`, otherwise it is invisible.

## Padding

`padding` is the inset a container leaves on all four sides. The content area starts at `(x + padding, y + padding)`, and cross axis stretching is also based on the content area:

```sil
Container {
    layout: vertical;
    width: 200;
    height: 100;
    padding: 12;
    background: #171B1F;

    Label { text: "12 from the border"; height: 16; }
}
```

`padding` applies to all child widgets and is computed only inside the container; it does not change the meaning of the container's own `width` / `height`: `width: 200` includes the 12 of padding on each side, so the content width is 176.

## Margins

The `margin` family controls the spacing between child widgets:

| Property | Effect |
| --- | --- |
| `margin` | Default spacing in the trailing direction: bottom in a vertical layout, right in a horizontal one |
| `margin-top` | Spacing placed **before** the child widget in a vertical layout |
| `margin-left` | Spacing placed **before** the child widget in a horizontal layout |
| `margin-bottom` | Spacing placed **after** the child widget in a vertical layout; uses `margin` when unset |
| `margin-right` | Spacing placed **after** the child widget in a horizontal layout; uses `margin` when unset |

!!! note "margin only applies in the trailing direction"
    In a vertical layout `margin: 8` is only equivalent to `margin-bottom: 8`; it does not add spacing above. To move the first child widget further from the top of the container, use `padding`, or write `margin-top: 8` on that child.

Spacing between adjacent child widgets adds up: the previous widget's `margin-bottom` and the next widget's `margin-top` both take up space, with no collapsing.

```sil
Container {
    layout: vertical;
    width: 240;
    padding: 8;

    Label { text: "Title"; height: 16; margin-bottom: 6; }
    Label { text: "Description"; height: 12; font-size: 0.7; }
    Button { text: "OK"; width: 80; height: 22; margin-top: 10; }
}
```

## Hiding and disabling

- A widget with `visible: false` is not drawn, does not take part in layout, takes no space, and cannot be clicked.
- A widget with `disabled: true` still takes part in layout and drawing, but ignores pointer, wheel and keyboard input; when a container is disabled, the whole subtree becomes non-interactive. How much a disabled widget is dimmed by default is controlled by `disabled-opacity`.
- Both can be bound: `visible: @show_hint;`, `disabled: @locked;`.

## Stacking and click order

`z-order` (aliases `z`, `layer`, `z-index`) decides the drawing order and click priority of sibling widgets:

- Higher values are drawn later and also receive clicks first.
- Equal values follow insertion order; nodes written later are on top.
- The `layer` property of `Part` means something else (whether the decoration layer sits above or below the widget); do not mix it up with `z-order`.

```sil
Container {
    layout: absolute;
    width: 200;
    height: 120;

    Panel { x: 10; y: 10; width: 120; height: 80; background: #2A3036; }

    Panel {
        x: 60; y: 40; width: 120; height: 80;
        background: #3C8446;
        z-order: 10; // Drawn above the previous panel
    }
}
```

## Absolute layout

With `layout: absolute` the container does not move its child widgets; each child uses its own `x` / `y`. This suits floating buttons, badges and stacked cards.

- `x` / `y` are coordinates relative to the top-left corner of the container's content area (when the container has `padding`, they start from the content area).
- When an ancestor uses a vertical or horizontal layout, the `x` / `y` of child widgets are overridden, so absolute positioning must live inside an `absolute` container.
- An absolute layout container does not compute its own size from its child widgets; set the width and height explicitly.

## No clipping and clipping

Ordinary containers **do not clip** their child widgets: the parts that overflow the container are still drawn and may also be clicked. When you need clipping, use `ScrollView`, which only draws and hit-tests content inside the viewport.

If you really need to confine content to a box, redesign the sizes first; do not rely on the bounds of an ordinary container.

## Layout and animation

`transform: scale(...)` in a keyframe is a pure render transform; it does not change layout or the click area. It scales around the center of the widget, which suits visual effects such as pulses and emphasis; do not let clickable targets depend on the scaled visual bounds.

The smooth slider (`smooth`) and `ScrollView` scrolling likewise only change what is drawn; layout sizes stay stable.

## Troubleshooting checklist

| Symptom | Usual cause |
| --- | --- |
| All child widgets pile up in the top-left corner | The container has no `layout`, and the default is `absolute` |
| Child widgets have width 0 and are invisible | There is no parent container to stretch them on the cross axis; give the child an explicit width |
| The container height is 0 | A vertical container with `height: 0` should grow automatically; check whether all child widgets are `visible: false` |
| The first widget sticks to the top edge | `margin` does not apply in the leading direction; use `padding` or `margin-top` |
| Clicks land in the wrong place | The host passes something other than GUI coordinates, or they are not converted for the GUI scale |
| Overflowing content can still be clicked | Ordinary containers do not clip; use `ScrollView` |

---

[Previous: Basic syntax](basics.md) · [Next: Common widgets](widgets.md)
