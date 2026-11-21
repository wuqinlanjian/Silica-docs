# Animations and transitions

Silica provides two layers of time-driven capability:

| Mechanism | Good for | Scope |
| --- | --- | --- |
| `transition` | Smooth interpolation when a value changes | Colors, opacity |
| `@keyframes` | Motion played in a loop or a fixed number of times | Opacity, render scale |

Both are advanced by the host calling `TickTree` every frame, using the real frame interval, so animation speed is independent of frame rate. The frame interval is clamped to 0.25 seconds, so the animation does not jump after a stutter.

## Transition `transition`

Declare the properties to smooth on a widget or in a style:

```sil
Style MenuButton {
    background: #33363A;
    color: #E6E6E6;
    radius: 4;
    transition: background .15s ease-out, color .15s ease-out;
}
```

Syntax:

```text
transition: property duration [easing][, property duration [easing]]...
```

- A duration is written as `0.3s`, `300ms`, or a bare number (unit: seconds).
- Easing can be omitted and defaults to `linear`.
- `all` means every supported property.
- Property names accept `background`/`background-color`/`bg` and `color`/`text-color`/`foreground`; they are normalized internally.

### Properties that support interpolation

| Property | Notes |
| --- | --- |
| `background` | Solid fill (gradients are not interpolated) |
| `color` | Text color |
| `border-color` | Stroke color |
| `opacity` | Overall opacity |

Geometric properties such as `width`, `height`, `radius`, and `padding` are not interpolated; changing them takes effect immediately. When you need size motion, use `transform: scale` in `@keyframes`.

### Which changes go through a transition

Transitions apply when a runtime value changes:

```sil
Var hot: false;

Panel {
    width: 160;
    height: 40;
    transition: background .25s ease-out;
    background: @hot ? "#8A3B3B" : "#33363A";  // smooth transition when the state switches
}
```

- The `@state ? a : b` form is registered as a dynamic rule evaluated every frame, so switching it goes through the transition.
- Ordinary bindings (`background: @color`) also go through the transition when the `DataStore` updates.
- On the first build the value jumps straight to the target; no animation plays from a default value.

!!! warning "State colors do not take part in transitions"
    `hover-background`, `pressed-background`, and `disabled-background` replace the fill color directly, so the switch is instantaneous. For a smooth hover highlight, drive the ordinary `background` with a `when` condition or a binding, together with `transition`.

## Keyframes `@keyframes`

Define them at the top level of the document:

```sil
@keyframes pulse {
    0%   { opacity: 1; }
    50%  { opacity: .55; }
    100% { opacity: 1; }
}

@keyframes pop {
    0%   { transform: scale(.8); opacity: 0; }
    60%  { transform: scale(1.06); opacity: 1; }
    100% { transform: scale(1); opacity: 1; }
}
```

Play them on a widget:

```sil
Label {
    text: "Loading";
    animation: pulse 1.5s infinite;
}
```

Syntax:

```text
animation: name duration [count|infinite]
```

- The name must match the name after `@keyframes` exactly (case-sensitive).
- If the duration is omitted it is 1 second; if the count is omitted the animation plays once.
- `infinite` means an endless loop.
- `animation: none` stops and clears the current animation.

### Properties supported in keyframes

| Form | Meaning |
| --- | --- |
| `opacity: 0..1` | Multiplied with the widget's own opacity |
| `transform: scale(0.9)` | Scales around the widget's center |
| `scale: 0.9` | Equivalent to the previous row |

Any other property written in a keyframe is ignored. For a color loop, change the value periodically from a script or through the `DataStore`, and rely on `transition` for the interpolation.

### Percentages and interpolation

- Steps are written as `0%`, `50%`, `100%`, and so on; the compiler sorts them by offset.
- Properties that no step covers stay at 1 for the whole animation (both opacity and scale).
- Interpolation between two steps is linear; easing cannot be written inside keyframe steps.

### Scale is a pure render transform

`transform: scale` affects drawing only and does not change layout or the hit area:

```sil
Button {
    text: "Confirm";
    animation: pop .3s;
}
```

- Scaling is centered on the widget.
- While drawing the child, the parent container temporarily enlarges its rectangle and restores it as soon as the drawing finishes.
- Clicks, hover, and layout are always computed from the original size.

So scaling does not push sibling widgets aside, and the click area does not "grow with the visuals".

### Opacity multiplication

Keyframe opacity is multiplied with the widget's base opacity:

```sil
Label {
    opacity: .5;
    animation: pulse 2s infinite;   // with a keyframe maximum of 1, the final value is .5
}
```

If the keyframe minimum is `.55`, the final minimum is `.275`.

## Easing

| Name | Curve |
| --- | --- |
| `linear` | Constant speed |
| `ease-out` / `easeout` / `out` | Fast first, then slow (quadratic) |
| `ease-in-out` / `ease` / `cubic` | Slow at both ends, fast in the middle (cubic) |
| `ease-in-out-sine` / `sine` | Sine easing |

`transition` and the ScrollView scroll animation both use these names; `@keyframes` uses linear interpolation.

## Re-triggering an animation

```sil
Button {
    text: "Play";
    animation: pop .4s;
    onClick: { @plays = @plays + 1; }
}
```

- Re-applying an `animation` property with the same name resets the timing and plays from the start.
- There is currently no declarative syntax for "replay automatically when the state changes"; to replay, set the `animation` property again from a script or from C++.
- After you change the contents of `@keyframes`, the document has to be reloaded; widgets that are already built do not pick up the new steps automatically.

## Combined example

```sil
Var muted: false;

Style ToggleRow {
    height: 44;
    padding: 8;
    background: #2C2F33;
    radius: 6;
    transition: background .2s ease-out, color .2s ease-out;
}

@keyframes breathe {
    0%   { opacity: .45; }
    50%  { opacity: 1; }
    100% { opacity: .45; }
}

Container {
    class: ToggleRow;
    layout: horizontal;

    Label {
        text: "Mute hint";
        width: 160;
        height: 28;
        color: @muted ? "#777" : "#E8E8E8";
    }
    Label {
        text: "●";
        width: 24;
        height: 28;
        color: @muted ? "#666" : "#7AD97A";
        animation: breathe 1.2s infinite;
    }
    Button {
        text: "Toggle";
        width: 72;
        height: 28;
        onClick: { @muted = !@muted; }
    }
}
```

## Common issues

| Symptom | Cause | Fix |
| --- | --- | --- |
| The transition has no effect | You changed a geometric property, or a hover state color | Use transitions only for colors and opacity |
| The animation does not play | The name is misspelled or has the wrong case | Make the `animation` and `@keyframes` names match exactly |
| The animation plays only once | The count was omitted | Write `infinite` or specify a count |
| The widget looks bigger but cannot be clicked | Scale does not take part in hit testing | This is expected; hits use the original rectangle |
| Animations are expensive with deep nesting | The whole subtree is sampled every frame | Add animations only to the nodes that need motion |

Next: [Lists and loops](lists.md) · [Images](images.md)
