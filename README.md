# `tdf`

Personal fork of [itsjunetime/tdf](https://github.com/itsjunetime/tdf).

## Changes from upstream

### Feat: Jump to last page
- Added to existing `G` keybinding (when not zoomed) to jump to the last page of the document -- `src/tui.rs:866`

### Feat: Color schemes
- Enable `invert` mode by default -- `src/renderer.rs:113`
- Add separate RGB search term highlight colors for normal/inverted modes -- `src/converter.rs:136`

### Fix: `gg` hang-up when attempting to jump to top of page
- The `gg` keybind to jump to the first page was broken. The old GoToPage function returned `None` for non-digits without dismissing the GoToPage state, which froze the app. Fixed by allowing an integer as well as `g` to be passed, while dismissing the state for any other non-digit.

### Change: Scroll speed
- Vertical pan/scroll speed (in zoomed mode) was increased from:
    `const PAN_STEP_Y: i16 = 1;` to `const PAN_STEP_Y: i16 = 6;`

### Fix: GoToPage (`g`)
- The `g` keybind to jump to a specific page was broken. Fixed by removing an unnecessary `is_kitty` guard.

### Remapped keybindings
- Motion and zoom-pan keys were remapped according to my personal Vim motions.

| Action | Upstream | This fork |
|---|---|---|
| Next page | `h` | `j` |
| Prev page | `j` | `k` |
| Next screen | `k` | `;` |
| Prev screen | `l` | `l` |
| Pan right (zoom) | `H` | `J` |
| Pan down (zoom) | `J` | `L` |
| Pan left (zoom) | `L` | `:` |
| Pan up (zoom) | `K` | `K` |


### Separate colors for inverted mode (`--inv-black-color`/`-B`, `--inv-white-color`/`-W`)
- Added two new CLI flags for specifying independent colors when inverted mode is active (toggled with `i`). Without these flags, behavior is unchanged — inverted mode swaps the normal `--black-color`/`--white-color` values as before.

```bash
tdf --white-color "F5F5DC" --black-color "1C1C1C" \
    --inv-white-color "AABBCC" --inv-black-color "112233" my.pdf
```
