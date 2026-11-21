# Images

`Image` draws textures: item icons, achievement icons, UI borders, and external art assets all work. The default size of `Image` is 16×16; if you write width or height as 0, it falls back to the default instead of stretching to fit like a container.

```sil
Image { src: "textures/items/diamond"; width: 32; height: 32; }
```

The texture is stretched over the whole widget rectangle, with no aspect-preserving clipping. To keep the aspect ratio, set the size yourself to match it.

## Three forms of `src`

| Form | Meaning | Example |
| --- | --- | --- |
| Game texture path | Passed straight to the game renderer for lookup | `textures/items/diamond` |
| `file:` prefix | Points to an image file on disk | `file:ui/logo.png` |
| Any `.png` / `.tga` path | Also treated as a file on disk | `ui/logo.png`, `D:/art/cat.png` |

Resolution rules:

1. Starts with `file:` → file on disk.
2. Extension is `.png` or `.tga` (case-insensitive) → file on disk.
3. Anything else is passed to the game as written and looked up as a built-in resource path.

So `textures/ui/icon` uses a game resource, while `textures/ui/icon.png` is treated as a file under the mod directory.

## Using images from disk

Silica does not read files straight into the renderer; it copies the image into an automatically generated resource pack:

```sil
Image { src: "file:ui/banner.png"; width: 240; height: 64; }
```

How it works:

- The target resource pack is at `<mod directory>/resource_packs/silica_images`.
- The file is copied as `textures/silica/<hash>.<ext>` and then drawn under that texture name.
- LeviLamina automatically registers each mod's `resource_packs` directory as a resource pack source, so you do not need to change the global pack list.
- Relative paths are resolved against the directory set by `Silica::SetRootPath`, which is usually the mod directory.
- Content with the same name is cached by size and modification time; after the file changes, the next resolution imports it again.
- Only `.png` and `.tga` are accepted.
- Absolute paths work as long as they are inside the mod directory; a path outside the mod directory is rejected and logged once:

```text
silica: refusing image outside the mod directory: D:/...
```

!!! tip "Prefer relative paths"
    Keep art assets in the mod directory and reference them with a relative path such as `ui/xxx.png`, so nothing needs changing when you deploy to another machine.

### First load and retry

After the document is compiled, `imagepack::Warmup` pre-imports images. If the client resource pack system is not ready yet, resolution fails and the widget retries automatically on later frames; a dark placeholder box is drawn until the image appears.

### Missing resources

- File does not exist or the texture name is misspelled: a placeholder box is drawn and logged once

  ```text
  silica: image texture not available: ui/typo.png
  ```

- A missing game built-in texture name also falls back to the placeholder box.

The placeholder box is a translucent dark rectangle with a stroke, so its position and size are obvious at a glance.

## Tinting

An image uses the widget `color` as a multiplicative tint; the default white means the original colors:

```sil
Image { src: "textures/ui/icon"; width: 20; height: 20; color: "#FF8080"; }
```

The alpha of `color` multiplies with the widget opacity, so you can also fade the whole image out with `opacity`.

## Nine-slice border

When you need a stretchable panel border, use `border-image` (alias `nine-slice`) instead of a solid color stroke:

```sil
Panel {
    width: 240;
    height: 120;
    border-image: ui/panel.png 8;
}
```

Syntax:

```text
border-image: texture slice...
border-image-slice: slice...
```

- The texture may be a game path or an image path on disk; **do not add quotes**.
- Slice units are **texture pixels**, not percentages.
- Number of slices and their meaning:

| Count | Meaning |
| --- | --- |
| 1 | Same on all four sides |
| 2 | Top and bottom / left and right |
| 3 | Top / left and right / bottom |
| 4 | Top / right / bottom / left |

- The four corners keep their original size, the edges stretch in one direction, and the middle stretches in both.
- When the box is smaller than the sum of the slices on both sides, the slices shrink proportionally.
- `border-image: none` clears the nine-slice border.
- While a nine-slice border is present, a plain `border-width` stroke is no longer drawn.

!!! note "Border images go through the resource pack too"
    A border image on disk is copied into `resource_packs/silica_images` just like an `Image`, and is subject to the mod directory boundary.

## Gradients

`background` accepts a gradient directly:

```sil
Panel {
    width: 200;
    height: 60;
    background: linear-gradient(to right, #23262A, #3A3F45);
}
```

Supported directions and spellings:

| Direction | Spelling |
| --- | --- |
| To the right | `to right` / `right` / `row` |
| To the left | `to left` / `left` |
| Downward | `to bottom` / `bottom` / `down` / `vertical` |
| Upward | `to top` / `top` / `up` |
| Angle | `45deg`, `180deg` (0° points up, clockwise) |

- Each color may be followed by a position (`0`-`1` or a percentage): `#000 0%, #FFF 100%`.
- An angle is snapped to the nearest axis (horizontal or vertical); arbitrary diagonal gradients are not supported.
- If you omit positions, they are spread evenly.
- The older form also works: `background: #23262A #3A3F45` (two or more colors), or `horizontal`/`vertical` to set the direction.
- Use `transparent` for a transparent color.
- Gradient colors do not take part in `transition` interpolation; a hover state color overrides the gradient.

## Stroke and corner radius

```sil
Panel {
    width: 180;
    height: 80;
    background: #26292D;
    radius: 8;
    border-width: 1;
    border-color: #454A50;
}
```

- `border: 1 #454A50` also sets the width and color at once.
- The corner radius applies to both the fill and the stroke.

## Shadow {#shadow}

```sil
Panel {
    width: 180;
    height: 80;
    background: #26292D;
    shadow: 0 4 12 #00000080;
}
```

`shadow` takes 1-4 numbers plus an optional color:

| Number of values | Meaning |
| --- | --- |
| 1 | Blur radius |
| 2 | Offset x / y |
| 3 | Offset x / y + blur |
| 4 | Offset x / y + blur + spread |

`shadow: none` clears the shadow. The shadow follows the rounded outline and is drawn beneath the fill.

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Black-and-white checkerboard or a dark box appears | The texture path does not exist | Check the `src` spelling and the resource pack |
| A file image does not appear | The path is outside the mod directory, or the extension is not png/tga | Move it into the mod directory and use a relative path |
| The image is stretched out of shape | `Image` fills the whole rectangle with the texture | Set width and height to match the original aspect ratio |
| The nine-slice border is out of proportion | Slices are interpreted as texture pixels | Use an image editor to check the edge width in pixels |
| The border image does not appear | The path is quoted | Remove the quotes |

Next: [Widget reference](../widgets.md) · [C++ integration](../cpp-guide.md)
