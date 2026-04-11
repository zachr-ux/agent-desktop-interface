# gui-tool

## What This Is

A zero-dependency Rust CLI binary for programmatic GUI interaction. Screenshots, window management, mouse control, keyboard input. Designed to be called by AI agents via subprocess. JSON-in/JSON-out. Supports Linux, macOS, and Windows.

## Autonomy

You have broad permissions in .claude/settings.json. Work autonomously — don't ask for confirmation on routine operations. Build, test, commit. Just do it.

## Hard Constraints

These are non-negotiable. Do not violate them for any reason.

### Zero Crates
`Cargo.toml` `[dependencies]` must be **empty**. No exceptions. No "just this one small crate." If you need functionality, implement it using Rust std or raw FFI.

### Zero System Dependencies
Never call external binaries via `std::process::Command`. No `gdbus`, no `xdotool`, no `wl-copy`, no `osascript`, no `screencapture`, no `powershell.exe`. Everything is done through direct FFI or kernel interfaces.

### FFI Policy
- **Linux**: All OS interaction via raw syscalls using `std::arch::asm!`. No `extern "C"` to libc. ioctl calls use inline assembly (syscall number 16 on x86_64, 29 on aarch64).
- **macOS**: `extern "C"` to CoreGraphics, CoreFoundation, and Objective-C runtime frameworks.
- **Windows**: `extern "system"` (stdcall) to user32.dll and gdi32.dll.

### Conditional Compilation
Platform backends use `#[cfg(target_os = "...")]`. No runtime OS detection.

## Architecture

```
src/
├── main.rs              # CLI parser, JSON output, dispatch
├── grid.rs              # Grid overlay: auto-scale, cell parsing, coordinate math
├── json.rs              # JSON serializer + parsing helpers (shared)
├── validate.rs          # Input validation (output paths, coordinates)
├── platform/
│   ├── mod.rs           # cfg dispatch to linux/macos/windows_os
│   ├── png.rs           # Shared PNG read/crop/write (deflate compress + inflate decompress)
│   ├── linux/
│   │   ├── mod.rs       # Public function dispatch
│   │   ├── uinput.rs    # /dev/uinput kernel interface (mouse + keyboard)
│   │   ├── screenshot.rs # XDG Portal Screenshot via raw D-Bus
│   │   ├── windows.rs   # window-calls GNOME extension via raw D-Bus
│   │   └── dbus/        # Raw D-Bus wire protocol client
│   │       ├── mod.rs
│   │       ├── auth.rs
│   │       ├── connection.rs
│   │       ├── message.rs
│   │       └── types.rs
│   ├── macos/
│   │   ├── mod.rs       # Public function dispatch
│   │   ├── ffi.rs       # CoreGraphics/CoreFoundation/ObjC FFI bindings
│   │   ├── input.rs     # CGEvent mouse + keyboard
│   │   ├── screenshot.rs # CGWindowListCreateImage
│   │   └── windows.rs   # CGWindowListCopyWindowInfo + NSRunningApplication
│   └── windows_os/
│       ├── mod.rs       # Public function dispatch
│       ├── ffi.rs       # Win32 types, constants, user32/gdi32 bindings
│       ├── input.rs     # SendInput mouse + keyboard
│       ├── screenshot.rs # BitBlt + GetDIBits
│       └── windows.rs   # EnumWindows + SetForegroundWindow
```

## Commands

```
gui-tool screenshot [--window <title>] [--window-id <id>] [--grid [WxH]] [--cell <ref>] [--output <path>]
gui-tool windows list
gui-tool windows raise <id>
gui-tool mouse move <x> <y> [--window <title>] [--window-id <id>]
gui-tool mouse move --cell <ref> [--grid WxH] --window-id <id>
gui-tool mouse click [--button left|right] [--window <title>] [--window-id <id>]
gui-tool key type <text> [--window <title>] [--window-id <id>]
gui-tool key press <combo> [--window <title>] [--window-id <id>]
```

All output is JSON to stdout. Errors are JSON to stderr.

`--window` and `--window-id` raise the target window before executing the action, all in one process. This eliminates the focus race condition when agents chain commands.

`--grid` overlays a labeled grid (default auto-scaled) on screenshots. `--cell` targets a grid cell for cropping (screenshot) or clicking (mouse move). Supports recursive zoom via dot notation: `B2.C1`. At each zoom level, small crops are scaled up to at least 640x480 before the grid is drawn, and the grid density for both screenshot cropping and mouse targeting uses these scaled dimensions so that cell labels are consistent across zoom and click.

## Public API Contract

Every platform must export these 11 functions from its `mod.rs`:

```rust
pub fn screenshot_full(output: &str) -> Result<String, String>
pub fn screenshot_window(title: &str, output: &str) -> Result<String, String>
pub fn screenshot_window_by_id(id: u64, output: &str) -> Result<String, String>
pub fn list_windows() -> Result<String, String>
pub fn raise_window(id: u64) -> Result<String, String>
pub fn find_window_by_title(title: &str) -> Result<Option<(u64, String)>, String>
pub fn get_window_bounds(id: u64) -> Result<(i32, i32, u32, u32), String>
pub fn mouse_move(x: i32, y: i32) -> Result<String, String>
pub fn mouse_click(button: &str) -> Result<String, String>
pub fn key_type(text: &str) -> Result<String, String>
pub fn key_press(combo: &str) -> Result<String, String>
```

## Key Conventions

- All public functions return `Result<String, String>` where the String is JSON
- JSON output always has a `"status"` field: `"success"` or `"error"`
- Error messages go to stderr as JSON `{"status":"error","message":"..."}`
- The `json.rs` module handles all serialization and parsing — no manual string building elsewhere
- Platform modules re-export via `platform/mod.rs` cfg dispatch
- No panics in production paths — propagate errors via Result
- Window IDs are `u64` across all platforms
- Screen coordinates for mouse are absolute pixels from top-left

## Testing

- `cargo build` must succeed with zero warnings on the target platform
- `cargo test` for unit tests (90 tests: json, png, grid, validation, deflate compression, arg parsing)
- `cargo test -- --ignored` for integration tests (10 tests: require running desktop session)
- Integration tests require platform-specific setup (input group on Linux, permissions on macOS)

## One-Time Setup

Run `./setup.sh` — it detects your OS and handles platform-specific setup.

### Linux
- udev rule for `/dev/uinput`
- User in `input` group (requires logout/login)
- window-calls GNOME extension

### macOS
- Accessibility permission (System Settings > Privacy & Security)
- Screen Recording permission (System Settings > Privacy & Security)

### Windows
- No special setup required
