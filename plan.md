# Equalize Pane Sizes

## Problem

WezTerm's split panes can end up with unequal sizes when splits are nested.
For example, splitting A→B→C horizontally (non-top-level) produces A=50%, B=25%, C=25%
because each split divides its own subtree, not the total space.

## Desired Behavior

A global config option that automatically equalizes pane sizes:
- On every split: equalize along the split direction
- On every pane close: equalize along the direction of the removed pane's parent split
- On window/tab resize: equalize both directions (horizontal first, then vertical)

### Lua API

```lua
config.equalize_panes = true
```

No per-action flags needed. When enabled, all splits and closes auto-equalize.

### Algorithm: Scoped Leaf-Counting

Equalization works on one axis at a time (determined by the split direction).
A different-direction split acts as a **scope boundary**:

1. Walk the tree. At each split node matching the target direction:
   - Count "leaf-columns" (horizontal) or "leaf-rows" (vertical) in each subtree,
     recursing only through same-direction splits. A different-direction child counts as 1.
   - Distribute the available space (minus separators) proportionally by leaf count.
2. When a different-direction split is encountered:
   - Don't count through it for the current axis.
   - **Do recurse into it** to find and equalize same-direction splits nested inside
     (these form a new independent scope with the different-direction node's size as their budget).
3. `apply_sizes_from_splits` is called at the end to resize the actual panes.

### Examples

**A | (B | C)** — horizontal equalize:
- Root H-split: left=1 leaf, right=2 leaves, total=3 → A=1/3, B=1/3, C=1/3

**A | (B / (C | D))** — horizontal equalize:
- Root H-split scope: A=1, V-split=1 (opaque), total=2 → A=50%, right=50%
- V-split is a scope boundary; recurse into children:
  - Leaf(B): nothing
  - H-split (new scope, budget=50% of total): C and D split equally → C=25%, D=25%

**A | (B / (D | E)) | C** — horizontal equalize:
- Root H-split scope: A=1, V-split=1 (opaque), C=1, total=3 → A=1/3, V-split=1/3, C=1/3
- Inner H-split scope (budget=1/3): D=1/6, E=1/6
- B spans full 1/3 column

## Implementation Checklist

### Core algorithm — `mux/src/tab.rs`

- [x] `count_leaves(tree, direction)` — counts leaf-columns/rows, opaque at cross-direction splits
- [x] `equalize_tree(tree, direction, cell_dims)` / `equalize_impl(...)` — budget-based scoped equalization
- [x] `TabInner::equalize_panes(direction)` — calls `equalize_tree` + `apply_sizes_from_splits`
- [x] `Tab::equalize_panes(direction)` — public delegate
- [x] Tests for standalone `equalize_panes` (10 tests)

### Global config option — `config/src/config.rs`

- [x] Add `equalize_panes: bool` field (default `false`)

### Auto-equalize on split — `mux/src/tab.rs`

- [x] In `split_and_insert`: after inserting, read `configuration().equalize_panes_on_split_close`. If enabled, call `self.equalize_panes(request.direction)`.
- [x] Tests: `config_equalize_on_split_horizontal_three`, `config_equalize_disabled_no_auto_equalize`, `config_equalize_on_split_mixed_directions`

### Auto-equalize on pane close — `mux/src/tab.rs`

- [x] In `remove_pane_if`: the `parent` from `unsplit_leaf` is a `SplitDirectionAndSize` — collect its `.direction`. After the removal loop, if config flag is enabled, call `equalize_panes` for each collected direction.
- [x] Tests: `config_equalize_on_close_horizontal_three_to_two`, `config_equalize_on_close_middle_pane`, `config_equalize_on_close_vertical`, `config_equalize_on_close_vertical_inside_horizontal`, `config_equalize_disabled_no_equalize_on_close`, `config_equalize_on_close_nested_complex`, `config_equalize_on_close_to_single_pane`

### Auto-equalize on window/tab resize — `mux/src/tab.rs`

- [ ] In `TabInner::resize`, in the non-zoomed branch, after `adjust_x_size`/`adjust_y_size` and after `self.size = size` (equalize reads `self.size` via `cell_dimensions`), but **before** `apply_sizes_from_splits`, when `configuration().equalize_panes` is enabled, call `equalize_impl` directly on the tree for both directions:
  - `equalize_impl(tree, SplitDirection::Horizontal, None, &cell_dims)`
  - `equalize_impl(tree, SplitDirection::Vertical, None, &cell_dims)`
  - Do **not** call `equalize_panes` (which wraps `equalize_impl` + `apply_sizes_from_splits`) — the existing `apply_sizes_from_splits` at the end of `resize` already handles pane resizing.
- [ ] Ensure this happens only for non-zoomed tree resize path (zoomed pane keeps existing behavior).
- [ ] Keep minimum-size clamping behavior unchanged.
- [ ] Add/enable tests:
  - `config_equalize_on_resize_horizontal_three` — grow width, verify ~thirds
  - `config_equalize_on_resize_vertical_three` — grow height, verify ~thirds
  - `config_equalize_on_resize_mixed_directions` — grow both, verify H and V scopes
  - `config_equalize_on_resize_shrink_horizontal` — shrink width, verify equalization under reduced space
  - `config_equalize_on_resize_same_size` — resize to same dimensions, verify skew is corrected
  - `config_equalize_disabled_no_auto_equalize_on_resize` — flag off, verify skew preserved

### Clean up per-split API

- [x] Remove `equalize` field from `SplitPane` in `config/src/keyassignment.rs`
- [x] Remove `SplitSize::Equalize` variant from `mux/src/tab.rs`
- [x] Remove `Equalize` mapping in `wezterm-gui/src/termwindow/mod.rs`
- [x] Remove `do_split_equalize` helper and `split_equalize_*` tests (replaced by config-flag tests)

## Files to Modify

- `config/src/config.rs` — add `equalize_panes: bool`
- `config/src/keyassignment.rs` — remove `equalize` field from `SplitPane`
- `mux/src/tab.rs` — remove `SplitSize::Equalize`, hook equalize into `split_and_insert`, `remove_pane_if`, and (planned) `resize` via config flag; update tests
- `wezterm-gui/src/termwindow/mod.rs` — remove `Equalize` mapping from `SplitPane` handler
