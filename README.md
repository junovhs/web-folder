# web-folder

A pure Rust crate for folder selection and drag-drop in browser WASM applications. Developed to aid the Rust refactor of https://diranalyze.vercel.app/

## Problem

Browser folder selection requires two distinct APIs:
1. `<input webkitdirectory>` for click-to-select
2. `DataTransferItem.webkitGetAsEntry()` for drag-and-drop

Both use callback-based `FileSystemEntry` APIs that are painful to use directly. No Rust crate wraps these into an ergonomic async API.

The `rfd` crate explicitly states `pick_folder()` "Does not exist in WASM32".

## Solution

Wrap the File and Directory Entries API (https://wicg.github.io/entries-api/) in ergonomic async Rust.

## Target

`wasm32-unknown-unknown` only. This crate has no native implementation.

## Dependencies

```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = [
    "DataTransfer",
    "DataTransferItem",
    "DataTransferItemList",
    "DragEvent",
    "File",
    "FileList",
    "FileSystemEntry",
    "FileSystemDirectoryEntry",
    "FileSystemDirectoryReader",
    "FileSystemFileEntry",
    "HtmlInputElement",
    "Document",
    "Window",
] }
js-sys = "0.3"
futures = "0.3"
```

## Public API

```rust
/// A file with its path relative to the selected folder root
pub struct FolderFile {
    pub path: String,      // e.g. "src/main.rs"
    pub name: String,      // e.g. "main.rs"
    pub file: web_sys::File,
}

/// Result of a folder selection (picker or drop)
pub struct FolderContents {
    pub root_name: String,
    pub files: Vec<FolderFile>,
}

/// Open a folder picker dialog. Returns None if user cancels.
pub async fn pick_folder() -> Option<FolderContents>;

/// Extract folder contents from a DragEvent.
/// Call this in your drop handler. Returns empty vec if no folders.
pub async fn from_drop_event(event: &web_sys::DragEvent) -> Vec<FolderContents>;

/// Lower-level: read a FileSystemDirectoryEntry recursively
pub async fn read_directory_entry(entry: web_sys::FileSystemDirectoryEntry) -> Result<FolderContents, JsValue>;
```

## Implementation Notes

### Folder Picker (`pick_folder`)

1. Create `<input type="file" webkitdirectory>` element (hidden)
2. Trigger click programmatically
3. Listen for `change` event
4. Read `input.files()` → `FileList`
5. Each `File` has `webkit_relative_path()` giving path like `"myfolder/src/main.rs"`
6. Strip the root folder name from paths for consistency

**Important**: The `webkitdirectory` attribute flattens the folder - you get ALL files with their relative paths, not a tree structure. This is actually simpler.

### Drag-Drop (`from_drop_event`)

1. Get `event.data_transfer().items()` → `DataTransferItemList`
2. For each item, call `item.webkit_get_as_entry()` → `Option<FileSystemEntry>`
3. Check `entry.is_directory()` 
4. Cast to `FileSystemDirectoryEntry` via `JsCast::unchecked_into()`
5. Call `read_directory_entry()` to recursively read

**Critical**: `webkit_get_as_entry()` returns `None` if called after any `await`. Collect ALL entries synchronously FIRST, then process them async.

### Recursive Directory Reading (`read_directory_entry`)

The `FileSystemDirectoryReader` API is callback-based:

```rust
async fn read_directory_entry(dir: FileSystemDirectoryEntry) -> Result<FolderContents, JsValue> {
    let reader = dir.create_reader();
    let mut all_entries = Vec::new();
    
    // Must call readEntries repeatedly until empty (browser limitation: max 100 per call)
    loop {
        let batch = read_entries_batch(&reader).await?;
        if batch.is_empty() {
            break;
        }
        all_entries.extend(batch);
    }
    
    let mut files = Vec::new();
    for entry in all_entries {
        if entry.is_file() {
            let file_entry: FileSystemFileEntry = entry.unchecked_into();
            let file = get_file_from_entry(&file_entry).await?;
            files.push(FolderFile {
                path: entry.full_path().trim_start_matches('/').to_string(),
                name: entry.name(),
                file,
            });
        } else if entry.is_directory() {
            let subdir: FileSystemDirectoryEntry = entry.unchecked_into();
            let subcontents = Box::pin(read_directory_entry(subdir)).await?;
            files.extend(subcontents.files);
        }
    }
    
    Ok(FolderContents {
        root_name: dir.name(),
        files,
    })
}
```

### Wrapping Callbacks as Futures

Both `FileSystemDirectoryReader.readEntries()` and `FileSystemFileEntry.file()` use callbacks. Wrap with `wasm_bindgen_futures`:

```rust
async fn read_entries_batch(reader: &FileSystemDirectoryReader) -> Result<Vec<FileSystemEntry>, JsValue> {
    let (tx, rx) = futures::channel::oneshot::channel();
    
    let success = Closure::once(move |entries: js_sys::Array| {
        let entries: Vec<FileSystemEntry> = entries
            .iter()
            .map(|e| e.unchecked_into())
            .collect();
        let _ = tx.send(Ok(entries));
    });
    
    let error = Closure::once(move |err: JsValue| {
        // Note: need separate channel or different approach for error
    });
    
    reader.read_entries_with_error_callback(
        success.as_ref().unchecked_ref(),
        error.as_ref().unchecked_ref(),
    );
    
    // Prevent closures from being dropped
    success.forget();
    error.forget();
    
    rx.await.unwrap()
}
```

**Better approach**: Use `wasm_bindgen_futures::JsFuture` if the API returns a Promise, but these APIs use callbacks, so manual channel wiring is needed.

## Browser Compatibility

| Browser | `webkitdirectory` | `webkitGetAsEntry` |
|---------|-------------------|-------------------|
| Chrome  | ✅ | ✅ |
| Firefox | ✅ | ✅ |
| Safari  | ✅ | ✅ |
| Edge    | ✅ | ✅ |

Both APIs are widely supported despite the `webkit` prefix.

## Example Usage

```rust
use web_folder::{pick_folder, from_drop_event};

// Picker button click handler
async fn on_pick_click() {
    if let Some(folder) = pick_folder().await {
        log::info!("Selected: {}", folder.root_name);
        for file in folder.files {
            log::info!("  {}", file.path);
        }
    }
}

// Drop zone handler  
fn on_drop(event: web_sys::DragEvent) {
    event.prevent_default();
    wasm_bindgen_futures::spawn_local(async move {
        let folders = from_drop_event(&event).await;
        for folder in folders {
            log::info!("Dropped: {}", folder.root_name);
        }
    });
}
```

## What This Crate Does NOT Do

- Native (non-WASM) folder picking (use `rfd` for that)
- File System Access API (`showDirectoryPicker`) - unstable, Chrome-only
- Writing files
- Virtual/sandboxed filesystem (that's `web-fs` / `browser-fs`)

## Crate Metadata

```toml
[package]
name = "web-folder"
version = "0.1.0"
edition = "2024"
license = "MIT OR Apache-2.0"
description = "Folder selection and drag-drop for Rust WASM applications"
repository = "https://github.com/USER/web-folder"
keywords = ["wasm", "folder", "directory", "file-picker", "drag-drop"]
categories = ["wasm", "web-programming"]

[lib]
crate-type = ["cdylib", "rlib"]
```

## File Structure

```
web-folder/
├── Cargo.toml
├── README.md
├── LICENSE-MIT
├── LICENSE-APACHE
└── src/
    ├── lib.rs          # Re-exports, FolderFile, FolderContents
    ├── picker.rs       # pick_folder() implementation
    ├── drop.rs         # from_drop_event() implementation  
    └── reader.rs       # read_directory_entry(), callback wrappers
```

## Testing

Manual browser testing required. Create an `examples/` folder with a simple HTML + Trunk setup demonstrating both picker and drag-drop.

## Prior Art

- `rfd` - Cross-platform file dialogs, no WASM folder support
- `gloo-file` - File/Blob utilities, no folder support
- `web-fs` - Sandboxed OPFS filesystem, different use case
