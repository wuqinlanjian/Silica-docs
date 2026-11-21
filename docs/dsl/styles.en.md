# Global styles

`Style` gives a group of properties a name so that several nodes share the same appearance. It holds only properties, has no child nodes, and cannot run event scripts, so it is safe to use as an "appearance template".

## Defining and using styles

```sil
Style Surface {
    background: #2C2F33;
    radius: 6;
    padding: 10;
}

Style DangerButton extends Surface {
    background: #8A3B3B;
    color: #FFE4E4;
    hover-background: #9E4646;
    pressed-background: #743131;
}
```

```sil
Container {
    class: Surface;
    height: 60;
    Label { text: "Plain panel"; }
}

Button {
    text: "Delete";
    class: DangerButton;
    height: 32;
}
```

`class:` and `style:` are synonymous directives; both take one or more style names, separated by spaces or commas:

```sil
Button { class: DangerButton, Compact; text: "Delete"; }
Button { style: DangerButton Compact; text: "Delete"; }
```

Multiple styles stack in the order written, with later ones overriding earlier ones:

```sil
// Compact is applied first, DangerButton afterwards and overrides the properties with the same name
Button { class: Compact DangerButton; text: "Delete"; }
```

The directives themselves have a fixed order: styles listed in `style` are applied first, those listed in `class` afterwards. When you mix the two, `class` wins.

## Name normalization

Style names ignore case, hyphens, and underscores when matched:

`DangerButton`, `danger-button`, and `danger_button` all refer to the same style.

!!! warning "Property keys are not normalized"
    Normalization applies only to style names and to the precedence decision of which style overrides which. Property keys themselves must still use the canonical form (lowercase, hyphenated), for example `margin-left`. Writing `marginLeft` is not recognized as a spacing property.

## Inheritance chain

`extends` forms a single-inheritance chain:

```sil
Style Base { padding: 8; radius: 4; color: #DDD; }
Style Panel extends Base { background: #26282B; }
Style Window extends Panel { border-width: 1; border-color: #3A3D41; }
```

When you apply `Window`, properties stack in the order `Base → Panel → Window`, with later entries taking higher precedence.

| Case | Result |
| --- | --- |
| The base style does not exist | Error `unknown style 'X'` |
| Circular inheritance (direct or indirect) | Error `style 'X' extends itself` |
| The inheritance chain exceeds 32 levels | Error `styles nest too deeply` |
| The same style name is defined twice | Warning `duplicate style ...; the last one wins` |

## Precedence with node properties

On the same node, final precedence from lowest to highest:

1. Styles in `style:` (in written order)
2. Styles in `class:` (in written order)
3. Inline node properties

Inline properties override styles, and this works **across categories**: an inline binding can override a literal value from a style, and an inline literal can override a binding from a style.

```sil
Style Card { background: #2C2F33; }
Card { class: Card; background: @highlighted ? "#3A4A2C" : "#2C2F33"; }
```

Here the style value for `background` gives way to the inline binding.

Property override comparison ignores hyphens, so `margin-left` and `marginLeft` are treated as the same key; but as noted above, a non-canonical key still reports an unknown property when it is applied.

## Bindings and templates in styles

Style values also support `@` bindings and `{{ }}` templates:

```sil
Var accent: "#498FC3";

Style AccentText {
    color: @accent;
}

Style Status {
    text: "{{@statusText}}";
}
```

- Bindings are attached to the widget when the node is built, and refresh with the `DataStore` afterwards.
- You cannot write event scripts in a style; doing so reports the error `styles cannot run scripts`.
- `$variables` in style values expand at compile time.

## State appearance

Common widget states are expressed through dedicated properties rather than pseudo-class selectors:

| Property | Effect |
| --- | --- |
| `hover-background` | Background when hovered |
| `pressed-background` | Background when pressed |
| `disabled-background` | Background when disabled |
| `disabled-opacity` | Overall opacity when disabled |

```sil
Style MenuButton {
    background: #33363A;
    color: #E6E6E6;
    hover-background: #40454A;
    pressed-background: #2A2D31;
    disabled-background: #2A2A2A;
    disabled-opacity: .5;
    radius: 4;
}
```

Writing a state color as `none` clears the value an earlier style supplied and restores the default behavior:

```sil
Style FlatButton extends MenuButton {
    hover-background: none;
    pressed-background: none;
}
```

!!! note "State colors only override the background"
    `hover-background`/`pressed-background`/`disabled-background` only affect the fill color. For a more elaborate hover skin (stroke, icon, handle), use the [Part decoration layer](widgets.md#part).

!!! warning "State colors switch instantly"
    Hover, pressed, and disabled state colors replace the fill color directly and do not take part in `transition` interpolation. To make the color change smoothly, drive the ordinary `background` with a binding or `@state ? a : b`; see [Animations and transitions](animations.md) for details.

## Styles do not inherit downward

A style applies only to the node that writes `class:`/`style:` and is not passed to child nodes automatically:

```sil
Container {
    class: Surface;
    Label { text: "The text color has to be set separately"; }   // does not inherit the color from Surface
}
```

When child nodes also need the style, tag them one by one, or make the title a `Component` to encapsulate it uniformly.

## Scope and visibility

- Styles are visible only inside the `.sil` document that defines them.
- Styles cannot be referenced across files.
- Descendant selectors, pseudo-classes, media queries, and conditional style blocks are not supported.
- Express conditional appearance with bindings or `@state ? a : b`, see [Reactive bindings](bindings.md).

## Organization tips

- Define a set of base token styles first (spacing, corner radius, font size), then define business styles that extend them.
- Route colors through `Var` where possible, so the whole theme can be reskinned:

```sil
Var bg: "#23262A";
Var fg: "#E4E6E8";
Var accent: "#498FC3";

Style Surface { background: $bg; color: $fg; radius: 6; }
Style Primary extends Surface { background: $accent; color: "#FFFFFF"; }
```

- Keep state colors and `transition` in the same style, so no button ends up without its transition animation.
- Do not build a deep inheritance chain just to save one property; two or three levels are enough.

Next: [Animations and transitions](animations.md) · [Widget reference](../widgets.md)
