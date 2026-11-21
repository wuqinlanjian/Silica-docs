# Common problems and troubleshooting

A UI that "does not show up", "does not refresh" or "does not respond to clicks" is rarely random. It is usually one of four causes: the document did not compile, the data key does not match, the layout computed to a zero size, or the host does not deliver the events. This page gives the investigation order, a log reference table and fixes for common symptoms.

## Investigation order

1. **Read the log**. Silica sends every compile error, warning and runtime problem through `runtime::SetLogSink`; first check whether the most recent load reported anything.
2. **Confirm the compile succeeded**. A single `error:` makes `CompileFile(filepath, fallback)` discard the file content and use the fallback interface instead. A UI that "looks like the default" has usually fallen back.
3. **Check the data keys**. A binding that cannot read its key gets `null` and reports nothing; a misspelled name only shows up as "the value stays at the default".
4. **Check the layout**. In an absolute layout a node without an explicit size is 0×0; a container does not automatically stretch beyond its children along the layout direction.
5. **Reduce to a minimal document**. Delete everything except the failing node, then add the properties back one at a time; this usually pinpoints which value causes the problem immediately.

!!! tip "Literals first, bindings later"
    First get position and size right with constants, then replace the values with `@bindings`. Tuning layout and bindings at the same time makes it hard to tell which layer the problem is in.

## Log format

A diagnostic has the form:

```text
source:line:column: severity: message
```

For example:

```text
D:/LeviLamina/mods/SilicaTest/ui/panel.sil:857:21: error: unknown style 'demo_row_box'
```

- `error:` means the compile failed. With fallback loading, the whole file is replaced.
- `warning:` means a property or script was skipped, the widget keeps its default value, and the rest of the document loads normally.
- A single parse keeps at most 200 diagnostics; fix the earlier errors, reload and look again.

Route the log into your own logging system:

```cpp
silica::Silica::SetLogSink([](std::string_view line) {
    mymod::Log(line); // write to a file or the console
});
```

When no sink is set, logs go to standard error and the Win32 debug output.

## Compile-time errors

| Message | Cause | Fix |
| --- | --- | --- |
| `cannot open '<path>'` | the file does not exist, or the path resolved incorrectly | check `SetRootPath` and the relative path; confirm the file name casing |
| `the document does not define any component` | the file only has definitions and no UI node | write at least one top-level node (`Container`, `Panel`...) |
| `unknown component 'X' (register it with WidgetFactory or define a Component)` | the node type is misspelled, or the custom widget is not registered | compare against the type names in the [widget reference](widgets.md); register custom widgets with `WidgetFactory::Register` first |
| `unknown style 'X'` | `class:` / `style:` references a style that does not exist | check the style name; names are case-insensitive and ignore `-`/`_`, but the style must actually be defined |
| `style 'X' extends itself` | style inheritance forms a cycle | remove the self-reference, or extend another base style |
| `'X' cannot contain 'Y'` | a child widget was put into a leaf widget | only `Container`, `ScrollView` and `ForEach` accept arbitrary child widgets; leaf widgets accept only `Part` |
| `'Slot' can only be used inside a Component definition` | `Slot` was written outside a component definition | move `Slot` into the root node of the `Component` |
| `component 'X' instantiates itself recursively` | the component references itself directly or indirectly | a component cannot be instantiated recursively; turn the recursive part into `ForEach` or write the levels out by hand |
| `components nest too deeply (limit 32)` / `(limit 64)` | the number of nested components exceeds the limit | split the components to remove one level of nesting |
| `styles nest too deeply` | the `extends` chain is too long | flatten the inheritance chain |
| `styles cannot run scripts; 'X' is only valid on a component` | an event script was written in a `Style` | a script can only be written on a UI node or on a node of a `Component` |
| `'ForEach' needs a data parameter` | `ForEach` is missing `data:` | write `ForEach (data: @rows, item: @row) { ... }` |
| `multiple component roots need a 'Container' component` | the component has several root nodes, but `Container` is not registered | register the built-in widgets or a custom `Container`; normally you call `WidgetFactory::RegisterBuiltins()` |
| `missing ';' after property 'X'` | the property value lacks a semicolon, and the next line is taken as a new property | add the `;`; several space-separated values in one property also trigger this |
| `missing '}' for 'X' opened at line N` | a node block or script block is not closed | starting at the reported line, look for the unclosed `{` |
| `unterminated string` / `unterminated block comment` | a string or a `/* */` comment is not closed | add the closing quote or `*/` |
| `expected ':' after parameter 'X'`, `expected ',' or ')' after parameter 'X'` | the parameter list is malformed | follow the `Component Name(a, b: default)` form |
| `expected hex digits after '#'` | no valid hexadecimal digits after the `#` in a color | use `#RGB`, `#RGBA`, `#RRGGBB`, `#RRGGBBAA` |
| `expected a name after '@'` / `'$'` | a bare `@` or `$` was written | add the key name; to show `@` in text, put it in a quoted string |

## Compile-time warnings

Warnings do not prevent loading, but the affected property or script does not take effect. The specific reason for `cannot bind` and `cannot attach` follows after a colon.

| Message | Usual cause | Fix |
| --- | --- | --- |
| `unknown property 'X' on 'Y'` | the property name is misspelled, or the property does not belong to this widget | compare against the property tables in the [widget reference](widgets.md); property names must be lowercase with hyphens, `fontSize` is not converted automatically |
| `cannot bind property 'X' on 'Y': ...` | the expression failed to parse, or the property does not support bindings | simplify the expression; confirm the property is one that `ApplyProperty` can set |
| `cannot attach the script on 'X' of 'Y': ...` | the script has a syntax error, or the event name is not an event property | check the statements, brackets and semicolons inside `{ ... }`; a script can only be attached to events such as `onClick`, `onChange`, `onSubmit` |
| `cannot evaluate the argument 'X': ...` | the argument expression passed to a component failed to parse | fix the argument expression; the component is instantiated with empty values |
| `component 'X' is missing the argument 'Y'` | the parameter has no default value and was not passed | pass the argument, or give the parameter a default value |
| `unknown argument 'X' on component 'Y'` | the call passes an argument the component does not declare | remove the argument, or add it to the parameter list of the component |
| `component 'X' has no Slot; its children are ignored` | the component definition has no `Slot`, but the call site writes child nodes | add a `Slot` in the component root node, or remove the child nodes at the call site |
| `component 'X' has the same name as a builtin widget; the builtin is used` | the component name collides with a built-in widget | give the component a different name |
| `duplicate property 'X'; the last value wins` | the same property is written twice on one node | delete the earlier one |
| `duplicate variable 'X'; the last value wins` / `duplicate style 'X'` / `duplicate component 'X'` | a document-level definition has a duplicate name | merge or rename them |
| `variable '$X' refers to itself` | the value of the `Var` references itself directly or indirectly | make it a constant, or split it into two variables |

!!! warning "Warnings are clues too"
    When part of the UI "has no effect at all", search the log for `warning:` first. A misspelled property does not fail the whole document, it only makes that line be skipped silently.

## Runtime logs

These are not compile diagnostics; they appear only while loading or drawing.

| Log | Meaning | Fix |
| --- | --- | --- |
| `silica: '<file>' is unavailable, compiling the fallback source` | the file has errors or cannot be read, so the fallback source is used | scroll up to the `error:` lines from the same batch, fix them and reload |
| `silica: reloaded '<absolute path>'` | the reload succeeded | nothing to do |
| `silica: reload of '<path>' failed; keeping the previous tree` | the reloaded document has errors, so the old interface is kept | fix the document as the log says and reload again |
| `silica: ReloadUI('<file>') before CompileFile; nothing to replace` | a reload was called before anything was loaded | call `CompileFile` before `ReloadUI`, or use `CompileFile` instead |
| `silica: ForEach truncated N rows to 512 (the data array is too large)` | the array has more than 512 items, so only the first 512 rows are rendered | paginate, filter, or avoid feeding one huge array at once |
| `silica: image texture not available: <src>` | the texture name does not exist, or the resource pack is not ready yet | check the path spelling and the resource pack; see [Images](dsl/images.md) |
| `silica: packed image '<path>' as <texture name>` | the disk image was imported into the resource pack | informational; useful to confirm where the path resolved to |
| `silica: image import needs SetRootPath (the mod directory)` | there is no root directory, so disk images cannot be imported | call `SetRootPath` at startup |
| `silica: refusing image outside the mod directory: <path>` | the absolute image path falls outside the root directory | put the asset inside the mod directory and use a relative path |
| `silica: cannot write the image pack manifest in <directory>` / `cannot copy image into the pack: ...` | the resource pack directory is not writable | check the permissions of the mod directory or whether a file is locked |
| `silica: texture api unavailable (ResourceLocation/TexturePtr symbols not resolved)` | the texture API of this game version was not resolved | confirm the framework version matches the game version; until then images can only show a placeholder box |

## Troubleshooting by symptom

### The UI does not show at all

| Possible cause | How to tell | Fix |
| --- | --- | --- |
| the compile failed and fell back | the log contains `error:` and `is unavailable` | fix the errors first |
| the root node came back empty | `CompileFile` returns `nullptr` | check whether the fallback source itself is valid; the log of `CompileFile` states the reason |
| the host does not draw the widget tree | other interfaces do not show either | confirm you call `root->Render(context)` and `FlushDraws(context)` every frame |
| the root node has a zero size | the width or height is 0 after layout | give the top-level container an explicit `width` / `height` |
| the root node is off screen | log `x/y/width/height`, or add a temporary stroke | check whether the coordinates are GUI units rather than pixels |
| hidden by opacity/visibility | `visible: false`, `opacity: 0`, or a binding that evaluates to false | temporarily replace it with a literal to verify |

### A widget does not show or is misplaced

- **Child nodes in a leaf widget are rejected**: the log shows `'X' cannot contain 'Y'`. Only container classes accept arbitrary child widgets.
- **The absolute layout has no size**: `Container` defaults to `absolute`, so child nodes are not arranged automatically; you must write `x` / `y` / `width` / `height`, or switch the container to `layout: vertical|horizontal`.
- **Only a 0 on the cross axis fills**: in a vertical layout `width: 0` means fill the container width; a hard-coded number keeps that width. Along the layout direction it is the opposite: the container wraps its child content, and child nodes are not stretched taller automatically.
- **`margin` only applies to the trailing edge**: in a vertical layout `margin` is the bottom spacing, and `margin-top` is the top spacing; `margin-left` has no effect in a vertical layout.
- **`padding` only affects the inner layout**: text insets in leaf widgets use their own properties (such as `text-padding`), not `padding`.
- **Clipped**: `ScrollView` clips child nodes beyond the viewport; overlays should be placed outside the `ScrollView`, or leave enough space.
- **Covered by a sibling**: a larger `z-order` draws later and wins in hit testing; check whether a container covering the whole screen is in front.

### Clicks do nothing

1. **Press and release must be inside the same widget**. Pressing, dragging out of the widget and then releasing does not count as a click.
2. **Disabled propagates to the whole subtree**. When a parent container has `disabled: true`, child buttons receive no input.
3. **Invisible means no hit testing**. A node with `visible: false` neither renders nor responds.
4. **No handler**. When neither an `onClick` script nor a C++ named callback is registered, a click only produces the hover/press visual change.
5. **A script suppresses the C++ handler of the same name**. When the same event has both a script and a named callback, the script runs first and the C++ callback is no longer called.
6. **An upper node intercepts the hit**. A transparent container with a high `z-order` takes the pointer first as long as its rectangle covers the point.

### The text box cannot be typed into

- `TextBox` must have focus first: click it, or let the focus cycle of the host switch to it.
- The host must forward keys to the current `FocusManager` widget; forwarding the mouse alone is not enough to enter text.
- Input beyond `max-length` (32 by default) is truncated, not dropped.
- When the widget or any ancestor is `disabled`, focus does not land there and the keyboard is not received.

### The wheel does not scroll

- The cursor must be inside the rectangle of the `ScrollView`, and the host must hand wheel events to `PointerRouter::Wheel`.
- Nothing scrolls while the content does not exceed the viewport; that is expected behavior.
- When the cursor rests on a disabled row, the wheel keeps looking outward for an enclosing scroll view; if no outer view can scroll either, nothing happens.

### Bindings do not refresh

| Possible cause | Fix |
| --- | --- |
| the data key is misspelled | a binding that cannot read gets `null` with no error; log the result of `DataStore::Get` to compare |
| the array/object inside a `Variant` was modified directly | containers are shared by handle and have no mutation API; build a new container and `Set` it |
| array elements were changed in place | same as above: rebuild the whole array and `Set` it, so the deep comparison emits a notification |
| a two-way binding was written on a path | only a single identifier (`@name`) can write back; `@item.name` is read-only |
| the host does not call `TickTree` / `Update` every frame | dynamic binding evaluation and deferred rebuilds both happen during the frame update |
| the `ForEach` data is not set yet | anything that is not an array (including `null`) gives 0 rows; the list rebuilds automatically once the data arrives |

### Scripts have no effect

- `cannot attach the script` in the log: the script failed to parse, and the whole block is discarded.
- An assignment target can only be a single identifier: `@health`, `health`, `$local`; `@config.volume` or an array index cannot be written.
- Scripts have no `for`, `while` or function definitions; put complex logic in C++.
- Division by 0, or a result that is not finite, is normalized to 0; `NaN` / `Infinity` is never written.
- A `$` variable exists only during this run of the script and is not kept until the next event.

### Images do not show

- Game texture names and disk files are distinguished by suffix: `textures/ui/icon` uses game assets, `textures/ui/icon.png` uses the mod directory. Getting the wrong form lands you on a placeholder box.
- Disk images support only `.png` and `.tga`, and must be inside the directory given to `SetRootPath`.
- On the first load the resource pack may not be ready yet; the widget draws a placeholder box first and retries on later frames. A `packed image` log line means the import succeeded.
- Images are stretched to the widget rectangle without proportional cropping; adjust `width` / `height` when the aspect ratio is wrong.

### The list is empty or its row content does not change

- `data:` on `ForEach` must evaluate to an array; objects, strings and `null` are all treated as an empty list.
- The `item` / `index` names in the row template are determined by the parameters and default to `item` / `index`; after renaming them, use the new names in the template too.
- After the array changes, the rebuild is scheduled for the `Update` of the next frame, so new rows are not visible immediately within the same frame.
- The row limit is 512, and truncation beyond it is logged.

### Animations do not run

- The `@keyframes` name must match the one written in `animation:` exactly (case-sensitive); when the name does not exist, nothing plays and no message is shown.
- Keyframes support only `opacity` and `transform: scale` / `scale`; other properties are not interpolated even if written.
- `transition` only takes effect on `background`, `color`, `border-color` and `opacity`, and only fires when such a property changes because of a binding or a dynamic value.
- Hover, press and disabled state colors switch instantly and do not go through `transition`; for a smooth change, drive a plain `background` from a binding.
- The host must call `TickTree(root, deltaTime)` with the real frame delta, and the delta is limited to a reasonable range.

## Known limitations

- Expressions have no `&&`, `||` or function calls; aggregate compound conditions with nested ternaries or in C++.
- There is no built-in "list delta update" beyond the configurator: `ForEach` rebuilds all rows each time and does not reuse rows by key.
- Scripts cannot write field paths or array elements, only single identifiers.
- `Component` cannot be instantiated recursively; component nesting is limited to 32, document node nesting to 64.
- There is only a uniform `padding`, no per-side padding.
- `options:` on `Dropdown` is comma-separated, and the option text itself cannot contain a comma.
- `TextBox` is single-line input with a default limit of 32 characters.
- Images accept only `.png` / `.tga`, and must be inside the mod directory.
- HTML/CSS/JavaScript is not rendered, and the game's native menus are not taken over.

## What to include when reporting a problem

1. The complete lines from the log starting with `error:` / `warning:`, with file, line and column.
2. The `.sil` snippet of the failing node (unrelated properties may be removed).
3. How the host calls `TickTree`, `Update`, `Render` and `FlushDraws` every frame.
4. The framework and game versions, and whether the problem appears on the first load or after `ReloadUI`.

---

[Previous chapter: C++ integration](cpp-guide.md) · [Next chapter: Examples](examples.md)
