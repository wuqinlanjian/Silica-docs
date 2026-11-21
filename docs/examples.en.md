# Examples

This page provides complete snippets you can adapt directly, covering common UI shapes: a settings page, a search list, a sliding switch, an image card, tabs, and a confirmation dialog. Every example uses only the properties and syntax listed in the documentation; put a snippet into a `.sil` document and have the host set the matching data keys to display it.

!!! note "Data keys in the examples"
    The examples assume the host has already set the required keys in `DataStore` and drives the widget tree every frame. For loading, rendering, and input wiring, see [C++ integration](cpp-guide.md); if you only replace the `.sil` document, use the reload entry point provided by the mod (F5 in SilicaTest) to see your changes.

<!-- Insert the snippet that loads a .sil document and attaches it to the render frame here -->

Initialize the example data:

```cpp
silica::DataStore::Set("volume", silica::Variant{80});
silica::DataStore::Set("muted", silica::Variant{false});
silica::DataStore::Set("ui_scale_index", silica::Variant{1});
```

## 1. Settings panel

Use `Style` to unify the card's appearance and `Component` to reuse a "title + hint + widget" row structure. The checkbox, slider, and dropdown all write user actions back to data keys.

```sil
Var surface: #1B1E22;
Var text_main: #D5DBE1;
Var text_dim: #7C848C;

Style Card {
    background: $surface;
    radius: 6;
    border: 1 solid #34383D;
    shadow: 0 6 18 #00000066;
}

Style RowTitle {
    font-size: 0.72;
    color: $text_main;
}

Style RowHint {
    font-size: 0.62;
    color: $text_dim;
}

Component SettingRow(title, hint: "") {
    Container {
        layout: vertical;
        margin-top: 14;

        Label { text: @title; height: 16; class: RowTitle; }
        Label { text: @hint; height: 8; class: RowHint; }

        Container {
            layout: vertical;
            margin-top: 6;
            Slot
        }
    }
}

Container {
    name: settings_panel;
    layout: vertical;
    width: 380;
    padding: 16;
    class: Card;

    Label {
        text: "Audio and UI";
        height: 24;
        font-size: 0.95;
        color: $text_main;
    }

    SettingRow(title: "Main volume", hint: "Applies while dragging") {
        Slider {
            height: 18;
            min: 0;
            max: 100;
            step: 1;
            smooth: true;
            value: @volume;
            onChange: { @volume = event.value; };
        }
    }

    SettingRow(title: "Mute", hint: "Keeps interface sounds") {
        Checkbox {
            text: "Turn off all game sounds";
            height: 18;
            checked: @muted;
            onChange: { @muted = event.checked; };
        }
    }

    SettingRow(title: "UI scale", hint: "Reopen the UI after changing") {
        Dropdown {
            width: 140;
            height: 22;
            options: "Small, Medium, Large";
            selected: @ui_scale_index;
            onChange: { @ui_scale_index = event.index; };
        }
    }
}
```

| Data key | Type | Description |
| --- | --- | --- |
| `volume` | Number | 0-100, written back by the script as the slider is dragged |
| `muted` | Boolean | Checkbox state |
| `ui_scale_index` | Number | Dropdown index, starting at 0 |

Key points:

- The root container of `SettingRow` is vertical; leaving `width` at 0 lets the parent container stretch it to full width; you do not need to write absolute coordinates by hand.
- The `options:` of a dropdown is comma-separated text, so the options themselves cannot contain commas.
- All three widget bindings are single identifiers, so user actions can write back to the data keys; C++ only needs to observe these keys.

## 2. Search box and result list

The text box handles input, `ForEach` handles rows, and `ScrollView` handles clipping and scrolling. A hint appears when there are no results.

```sil
Component ResultRow(name, meta) {
    Container {
        layout: horizontal;
        height: 34;
        padding: 6;
        margin-top: 4;
        radius: 4;
        background: #FFFFFF08;

        Label {
            text: @name;
            width: 200;
            height: 16;
            color: #D5DBE1;
        }
        Label {
            text: @meta;
            width: 72;
            height: 16;
            font-size: 0.7;
            color: #7C848C;
            text-align: right;
        }
    }
}

Container {
    layout: vertical;
    width: 320;
    padding: 12;
    background: #171B1F;
    border: 1 solid #30363C;

    TextBox {
        height: 24;
        text: @query;
        placeholder: "Search items...";
        onChange: { @query = event.text; };
        onSubmit: { @search_submitted = @search_submitted + 1; };
    }

    Label {
        text: "No matching results";
        height: 18;
        margin-top: 10;
        font-size: 0.75;
        color: #6E767D;
        visible: @result_count == 0;
    }

    ScrollView {
        height: 220;
        margin-top: 8;
        scroll-step: 30;
        scrollbar-width: 3;
        scrollbar-inset: 3;
        scrollbar-thumb-color: #4A5158;

        ForEach (data: @results, item: @result) {
            ResultRow(name: @result.name, meta: @result.meta)
        }
    }
}
```

| Data key | Type | Description |
| --- | --- | --- |
| `query` | String | Input contents, written back by `onChange` |
| `search_submitted` | Number | Number of Enter presses, used by C++ to trigger a query |
| `results` | Array | Each item contains `name` and `meta` fields |
| `result_count` | Number | Number of results, used for the empty state hint |

Key points:

- If the `ForEach` data is not an array, it is treated as an empty list; you do not need to check types in the `.sil` document.
- `result_count` is a separate key: `ForEach` itself does not expose a row count, so the empty state is maintained separately by C++ or a script.
- A `Label` inside a row must set its width explicitly in a horizontal layout; the main axis does not stretch it just because `width: 0`.

## 3. Sliding switch {#switch}

Turn off the built-in box of `Checkbox`, draw the track and the thumb with `Part`, and switch between them with `when` based on the `checked` state.

```sil
Checkbox {
    text: "Night mode";
    width: 220;
    height: 28;
    checked: @night;
    show-box: false;
    text-gap: 56;
    font-size: 0.8;
    onChange: { @night = event.checked; };

    Part {
        anchor: right;
        width: 42;
        height: 22;
        radius: 11;
        background: #33363A;
        when: "!checked";
        layer: under;
    }
    Part {
        anchor: right;
        width: 42;
        height: 22;
        radius: 11;
        background: #498FC3;
        when: "checked";
        layer: under;
    }
    Part {
        anchor: right;
        inset-right: 22;
        width: 18;
        height: 18;
        radius: 9;
        background: #87919A;
        when: "!checked";
        layer: under;
    }
    Part {
        anchor: right;
        inset-right: 2;
        width: 18;
        height: 18;
        radius: 9;
        background: #FFFFFF;
        when: "checked";
        layer: under;
    }
}
```

Key points:

- A part with `anchor: right` is vertically centered inside the host widget, and `inset-right` controls its distance from the right edge.
- The `when` condition separates multiple states with spaces, and the `!` prefix negates; available states include `checked`, `hovered`, `pressed`, and `disabled`.
- `Part` does not receive mouse events; clicks are still handled by `Checkbox`.
- The thumb switches instantly; if you need a sliding animation, you can add two more `Part`s together with keyframe scaling, but do not rely on part positions for layout.

## 4. Image card

Use a nine-slice border as an adaptive frame, and lay out the icon and text horizontally inside. Both the border image and the icon can come from the mod directory.

```sil
Container {
    width: 260;
    height: 86;
    padding: 12;
    layout: horizontal;
    border-image: ui/card.png 12;
    shadow: 0 6 16 #00000066;

    Image {
        src: "textures/items/diamond";
        width: 48;
        height: 48;
        margin-right: 12;
    }

    Container {
        layout: vertical;
        width: 160;

        Label { text: "Diamond"; height: 20; font-size: 0.95; color: #DCE3E9; }
        Label { text: "Stock 64"; height: 16; font-size: 0.75; color: #8C959D; }
        Label { text: "Right-click to view recipe"; height: 14; font-size: 0.65; color: #6E767D; }
    }
}
```

A card with a gradient background does not need a border image:

```sil
Panel {
    width: 220;
    height: 72;
    radius: 8;
    background: linear-gradient(to bottom, #2B3036, #1A1D21);
    shadow: 0 4 14 #00000080;
}
```

Key points:

- The slice unit of `border-image` is texture pixels; `ui/card.png` must be inside the mod directory specified by `SetRootPath`.
- Images on disk are copied into an automatically generated resource pack; on first load the resource pack may not be ready yet, so the image position shows a placeholder box first.
- `Image` stretches the texture to fill the rectangle and does not preserve the aspect ratio; write `width` / `height` to match the asset's aspect ratio.

## 5. Status badge (keyframes)

`@keyframes` supports only `opacity` and `transform: scale`, which suits breathing effects and attention-grabbing hints.

```sil
@keyframes breathe {
    0%   { opacity: 1; }
    50%  { opacity: .5; }
    100% { opacity: 1; }
}

Container {
    layout: horizontal;
    width: 240;
    height: 28;
    padding: 6;
    background: #171B1F;
    radius: 6;

    Label {
        text: "●";
        width: 20;
        height: 20;
        font-size: 0.8;
        color: #6E767D;
        visible: @muted;
    }
    Label {
        text: "●";
        width: 20;
        height: 20;
        font-size: 0.8;
        color: #7AD97A;
        animation: breathe 1.6s infinite;
        visible: !@muted;
    }
    Label {
        text: @muted ? "Muted" : "Sound on";
        width: 120;
        height: 20;
        font-size: 0.75;
        color: #C8CFD5;
    }
    Button {
        text: "Toggle";
        width: 64;
        height: 20;
        margin-left: 8;
        font-size: 0.7;
        onClick: { @muted = !@muted; };
    }
}
```

Key points:

- Keyframe names are case-sensitive; a typo does not raise an error, the animation simply does not play.
- Scale affects drawing only and does not change the hit area; do not rely on the scaled visual bounds for click targets.
- To make color changes smooth, use ordinary `background` / `color` bindings and declare a `transition`; state colors are instantaneous.

## 6. Tabs

Tab headers are generated by `ForEach`, and a click writes the current index; the content area uses comparison expressions to control visibility.

```sil
Container {
    layout: vertical;
    width: 360;

    Container {
        layout: horizontal;
        height: 30;

        ForEach (data: @tabs, item: @tab_info, index: @i) {
            layout: horizontal;

            Button {
                text: @tab_info.name;
                width: 88;
                height: 28;
                margin-right: 4;
                font-size: 0.72;
                background: @tab == @i ? "#4B535A" : "#2A2F34";
                color: @tab == @i ? "#F0F3F5" : "#9AA3AA";
                onClick: { @tab = @i; };
            }
        }
    }

    Container {
        layout: vertical;
        width: 360;
        height: 140;
        margin-top: 8;
        padding: 12;
        background: #1B1E22;
        border: 1 solid #34383D;

        Label { text: "General"; height: 18; visible: @tab == 0; }
        Label { text: "Key bindings"; height: 18; visible: @tab == 1; }
        Label { text: "About"; height: 18; visible: @tab == 2; }
    }
}
```

| Data key | Type | Description |
| --- | --- | --- |
| `tabs` | Array | Each item contains a `name` field |
| `tab` | Number | Current index, defaults to 0 |

Key points:

- The name of the `index` parameter injected inside `ForEach` is determined by `index:`; here it is `@i`, and the script uses `@i` as well.
- `onClick: { @tab = @i; };` works because the script looks up `i` in the row scope first when reading.
- The tab contents are three fixed nodes; when the number of tabs is driven by data, change the content area to `ForEach` as well.
- Hidden pages take no layout space; with a fixed container height, switching tabs does not change the overall size.

## 7. Confirmation dialog

Use an absolute layout to stack a backdrop and a dialog, and `z-order` keeps the dialog above the backdrop.

```sil
Container {
    width: 420;
    height: 240;
    layout: absolute;
    visible: @confirm_open;

    Button {
        x: 0;
        y: 0;
        width: 420;
        height: 240;
        text: "";
        background: #00000099;
        hover-background: #00000099;
        pressed-background: #00000099;
        onClick: { @confirm_open = false; };
    }

    Container {
        x: 100;
        y: 50;
        width: 220;
        height: 140;
        z-order: 10;
        layout: vertical;
        padding: 14;
        background: #23272B;
        border: 1 solid #3A4046;
        radius: 8;
        shadow: 0 10 24 #00000099;

        Label {
            text: "Delete this save?";
            height: 22;
            font-size: 0.9;
            color: #E8EDF1;
        }
        Label {
            text: "This action cannot be undone.";
            height: 16;
            margin-top: 4;
            font-size: 0.7;
            color: #8C959D;
        }

        Container {
            layout: horizontal;
            height: 26;
            margin-top: 12;

            Button {
                text: "Cancel";
                width: 88;
                height: 26;
                margin-right: 8;
                onClick: { @confirm_open = false; };
            }
            Button {
                text: "Delete";
                width: 88;
                height: 26;
                background: #7A3434;
                onClick: {
                    @confirm_open = false
                    @deleted = @deleted + 1;
                };
            }
        }
    }
}
```

| Data key | Type | Description |
| --- | --- | --- |
| `confirm_open` | Boolean | Whether the dialog is shown; C++ can set it at any time |
| `deleted` | Number | Number of deletions, for the host to respond to |

Key points:

- Use a full-size `Button` for the backdrop rather than a `Panel`: a `Panel` does not receive clicks, so clicking the backdrop cannot close the dialog.
- The backdrop and the dialog are sibling nodes; `z-order: 10` makes the dialog render later and win hit testing.
- When the root container is hidden, the entire subtree is neither rendered nor given input; the host only needs to maintain one boolean key.
- A dialog height of 140 is enough for two lines of text and the button row; raise it in step when you add content, because a vertical container does not stretch a root with a fixed height.

## Composition tips

- **Components before pages**: turn repeated rows, cards, and switches into `Component`s, and let pages only arrange them.
- **Stable key names**: once keys such as `@volume` and `@results` are settled, both C++ and the `.sil` documents are written around them; do not invent a separate set of names on each side.
- **One root per page**: make dialogs and toast bars siblings in an absolute layout, so you do not have to load a separate document for every overlay.
- **Verify with logs**: after editing a `.sil` document, check for `warning: unknown property` first, then look at the result; a misspelled property does not raise an error, it is simply skipped.

---

[Previous chapter: FAQ and troubleshooting](troubleshooting.md) · [Back to home](index.md)
