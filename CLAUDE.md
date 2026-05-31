# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`tdf` is a terminal-based PDF viewer built with Rust and ratatui. It supports async rendering, search, hot reload, and Kitty terminal image protocol for high-quality display.

**Optional features:** `epub` (EPUB support) and `cbz` (comic book archive support) — add via `--features epub,cbz` to cargo commands.

## Build & Run

```bash
# Build (debug)
cargo build

# Build (release)
cargo build --release

# Build (maximally optimized, uses fat LTO — see scripts/build_most_optimized.sh)
cargo build --profile production

# Run
cargo run -- path/to/file.pdf

# Run with features
cargo run --features epub,cbz -- path/to/file.epub
```

**System dependencies required:** `libfontconfig` and `clang`. On Linux: `libfontconfig1-devel` (or `libfontconfig-dev`) and `clang`.

**Toolchain:** Nightly Rust (see `rust-toolchain.toml`).

## Testing & Linting

```bash
# Run tests (only in src/skip.rs currently)
cargo test

# Run a single test
cargo test iter_works

# Lint
cargo clippy

# Format
cargo fmt

# Run benchmarks
cargo bench
```

## Code Formatting

Configured in `.rustfmt.toml`:
- Hard tabs for indentation
- No trailing commas
- `StdExternalCrate` import grouping
- `imports_granularity = "Crate"`

## Architecture

The app runs three concurrent execution contexts that communicate via `flume` channels:

### 1. Renderer thread (`src/renderer.rs`)
A dedicated OS thread (not Tokio) that wraps mupdf (a C library via FFI). It cannot be async because `mupdf::Document` is `!Send`. It runs a `'reload` loop that re-opens the document on hot reload, and an inner `'render_pages` loop that iterates pages using `InterleavedAroundWithMax` — rendering outward from the current page in both directions so pages are ready whether the user pages forward or backward.

Receives `RenderNotif` messages; sends `Result<RenderInfo, RenderError>` back.

### 2. Converter task (`src/converter.rs`)
A Tokio async task. Receives `PageInfo` (raw PNM pixel data + search rects) from the renderer, decodes the image, highlights search results by manipulating pixel data via rayon, then converts to the appropriate terminal image protocol via `ratatui-image`. For Kitty terminals it produces `ConvertedImage::Kitty` (with optional shared-memory transfer); for others it produces `ConvertedImage::Generic`.

Receives `ConverterMsg`; sends `Result<ConvertedPage, RenderError>` back.

### 3. Main/TUI task (`src/tui.rs`, `src/main.rs`)
The main async task runs the `enter_redraw_loop` which `tokio::select!`s between:
- Crossterm keyboard/mouse events → `Tui::handle_event` → `InputAction`
- Renderer messages (new page data, search results, reload events)
- Converter messages (converted/display-ready pages)

`Tui` holds the UI state and renders via ratatui. Kitty image display is handled separately by `display_kitty_images` in `src/kitty.rs` after each ratatui draw call, using z-index cycling to replace old images.

### Channel topology

```
[File watcher (notify)] ──→ RenderNotif ──→ [Renderer thread]
                                                    │
                              RenderInfo ←──────────┘
                                    │
                              ConverterMsg ──→ [Converter task]
                                                    │
                              ConvertedPage ←────────┘
                                    │
                          [Main/TUI loop] ←── crossterm events
```

### Key types

- `RenderNotif` — commands to the renderer (area change, jump, search, reload, invert, rotate, fit/fill)
- `RenderInfo` — renderer output (page count, rendered page, search results, reload notification)
- `ConverterMsg` — commands to the converter (page count, current page, new raw page to convert)
- `ConvertedPage` — converter output (display-ready image, page number, search result count)
- `Tui` — all UI state; `Tui::render` returns `KittyDisplay` describing what Kitty images to show
- `Zoom` — tracks zoom level and pan offsets (cells); only active when in Kitty fill-screen mode

### Zoom system (`src/tui.rs`)
Zoom is Kitty-only. `Zoom::factor()` maps an `i16` level to a float multiplier (ZOOM_RATE = 1.1 for levels > 0, ZOOM_RATE_GRANULAR = 1.05 for levels ≤ 0). Zoom-out uses a two-pass system to detect and clamp at fit-screen rather than zooming further out.

### `InterleavedAroundWithMax` (`src/skip.rs`)
A custom iterator that yields page indices radiating outward from a center point, alternating above/below, wrapping at bounds. Used by both the renderer and converter to prioritize pages near the currently-viewed page.

## Clippy Configuration

The project enables a large set of `clippy::` lints in `Cargo.toml`. `indexing_slicing` is explicitly allowed (required for ratatui APIs). When adding code, expect clippy to be strict — run `cargo clippy` before assuming code is clean.

## Important Notes

- The renderer thread must remain non-async because `mupdf::Document` is `!Send` and cannot cross await points.
- `ratatui` and `ratatui-image` are pinned to custom forks via git revisions in `Cargo.toml` (performance fixes pending upstream).
- `mimalloc` is the global allocator (set in `src/lib.rs`).
- The file watcher watches the **parent directory** (not the file directly) as a workaround for inode-loss on delete/recreate saves.
- Enable logging by setting `RUST_LOG` env var; logs go to `./debug.log`.
- Enable tokio-console tracing with `--features tracing`.
