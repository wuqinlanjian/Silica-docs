# Lists and loops

`ForEach` builds a subtree repeatedly from the data in an array. It suits content whose item count varies with the data, such as friend lists, search results, and inventory slots.

## Basic syntax

```sil
ForEach (data: @friends, item: @friend, index: @i) {
    Container {
        layout: horizontal;
        height: 32;
        padding: 6;

        Image { src: "textures/ui/icon_steve"; width: 24; height: 24; }
        Label { text: @friend.name; width: 180; height: 24; }
        Label { text: "{{@i}}"; width: 32; height: 24; color: #888; }
    }
}
```

`friends` is an array data key written to `DataStore` by the host (`DataStore::Set("friends", Variant::Array(...))`); it is unrelated to document-level `Var`. `Var` only performs compile-time text substitution and carries no runtime data.

Syntax:

```text
ForEach (data: expression, item: row variable name, index: index variable name) { template nodes... }
```

| Parameter | Required | Default |
| --- | --- | --- |
| `data` | Yes | None; omitting it is an error |
| `item` | No | `item` |
| `index` | No | `index` |

You can write the variable name as `@friend` or `friend`; the `@` only marks a scoped variable.

## Scope

Each row creates a child scope that injects `item` and `index`:

- Read them directly inside the row with `@friend` and `@i`.
- Name resolution order: **row scope → outer row scope → `DataStore` → widget state**.
- A row variable can shadow a global key of the same name; inside a nested `ForEach`, the inner variable shadows the outer one.

`index` starts at 0 and is a number.

## Data source

`data` is an ordinary expression and supports paths:

```sil
ForEach (data: @profile.friends, item: @friend) { ... }
ForEach (data: @groups.0.members, item: @member) { ... }
```

Rules:

- The expression must evaluate to an array; objects, strings, numbers, and null are treated as an empty list and produce no rows.
- The root keys referenced by the expression are observed. When `@profile` updates, the whole list is rebuilt.
- Row variables provided by an outer `ForEach` do not become `DataStore` subscriptions; an inner list updates cascadingly when the outer list is rebuilt.

## When rebuilds happen

A data change does not destroy rows on the spot:

1. A `DataStore` write triggers `OnDataChanged`, and `ForEach` only marks itself dirty.
2. All rows are rebuilt on the next `Update`.

This way, a row can only be destroyed after a click script has finished running inside it, so a widget is never released halfway through a script. The cost is one frame of delay between a data change and the updated picture.

!!! warning "No key recycling"
    Every rebuild destroys and recreates all rows; no ID comparison is done. Transient state of widgets inside rows (text being typed, focus, scroll position) is lost. Put any edit state you need to keep in `DataStore`.

## Row limit

A single rebuild produces at most **512** rows; the excess is dropped and logged once:

```text
silica: ForEach truncated 900 rows to 512 (the data array is too large)
```

If you need to show a larger data set, filter or paginate first instead of relying on `ScrollView` to hold thousands of widgets.

## Layout

`ForEach` is itself a container and defaults to `vertical`, stacking rows from top to bottom:

```sil
ForEach (data: @items, item: @it) {
    Label { text: @it.name; height: 20; }
}
```

- Set `layout: horizontal` on the `ForEach` to arrange rows side by side.
- Without an explicit `height`, the height accumulates from the row content; if you set `height`, the document value wins.
- If the template has several top-level nodes, they all become sibling nodes of the row; wrap them in a container yourself if you want them combined within one row.

### Inside a scroll view

When there are many rows, put the `ForEach` inside a `ScrollView`:

```sil
ScrollView {
    width: 260;
    height: 200;
    show-scrollbar: true;

    ForEach (data: @entries, item: @entry) {
        Label { text: @entry.title; height: 26; }
    }
}
```

`ScrollView` scrolls by content height and clips rows that are not visible.

## Interaction inside rows

Buttons and checkboxes inside a row can read and write `DataStore`:

```sil
ForEach (data: @friends, item: @friend, index: @i) {
    Checkbox {
        text: @friend.name;
        height: 28;
        checked: @friend.selected;        // read-only: a path binding does not write back
        onChange: { @selectedIndex = @i; }  // the script writes a global key
    }
}
```

Limits:

- Two-way binding write-back only supports a single identifier. `checked: @friend.selected` is a path, so it only reads in one direction; the checked state is not written back to the array element.
- A script assignment target must also be a single identifier (such as `@selectedIndex`); you cannot write `@list.0.done`.
- To modify an array element, replace the whole array from the C++ side and let `ForEach` rebuild.

## Nested lists

```sil
ForEach (data: @groups, item: @group) {
    Container {
        layout: vertical;

        Label { text: @group.name; height: 24; }
        ForEach (data: @group.members, item: @member) {
            Label { text: @member.name; height: 20; color: #AAA; }
        }
    }
}
```

The inner data comes from the outer row variable and does not subscribe to `DataStore`; the inner list is rebuilt together with the outer one. The nesting depth is limited by the component nesting depth.

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| No rows at all | `data` is not an array, or the key name is misspelled | Check that `DataStore` holds an array |
| Row content does not update | Only an element inside the array was changed, without calling `Set` again | Write the whole array again |
| Checked state is not written back | The binding target is a path | Write a global key in the script, or handle it in C++ |
| Rows are truncated | More than 512 rows | Filter, paginate, or use another approach |
| Scroll position jumps back to the top | A rebuild changed the content height | Keep edit state and scroll position outside the list |

Next: [Images](images.md) · [Components](components.md)
