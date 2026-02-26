# Developer Onboarding — RapidRAW

## What is RapidRAW?

RapidRAW is a non-destructive, GPU-accelerated RAW photo editor (Lightroom alternative) built with Tauri 2. The frontend is React + TypeScript; the backend is Rust. The entire image processing pipeline runs on the GPU via wgpu and a custom WGSL shader. Edits are stored in `.rrdata` JSON sidecar files and never touch original images.

---

## Prerequisites

- **Rust** — install via [rustup](https://rustup.rs/). Minimum version: 1.92 (see `src-tauri/Cargo.toml`).
- **Node.js** — v18+ recommended.
- **Tauri prerequisites** — platform-specific system libraries: https://tauri.app/start/prerequisites/
- **Git submodules** — `rawler` (the RAW decoder) lives at `rawler/rawler/`. Always clone with:
  ```bash
  git clone --recurse-submodules https://github.com/CyberTimon/RapidRAW.git
  # or if already cloned:
  git submodule update --init
  ```

---

## Running the App

```bash
npm install       # Install frontend deps
npm start         # tauri dev — starts Vite dev server + Rust backend together
```

`npm start` is the only command you need for day-to-day development. It:
1. Runs `npm run dev` (Vite on port 1420) as the "beforeDevCommand"
2. Compiles and launches the Tauri/Rust backend
3. Hot-reloads the frontend on TypeScript/TSX changes
4. Recompiles Rust automatically on backend changes (triggers a full app restart)

For release builds:
```bash
npm run tauri build    # Produces a platform installer in src-tauri/target/release/bundle/
```

**Build profiles** — the dev profile uses `opt-level = 3` with incremental compilation so Rust builds stay fast during development. Release builds use LTO + single codegen unit for maximum size/speed optimization.

---

## Project Layout

```
RapidRAW/
├── src/                        # React/TypeScript frontend
│   ├── App.tsx                 # Root component — all app state lives here
│   ├── utils/adjustments.tsx   # Central data model for all image adjustments
│   ├── components/
│   │   ├── adjustments/        # Slider UIs: Basic, Color, Curves, Details, Effects
│   │   ├── modals/             # Modal dialogs (export, denoise, panorama, etc.)
│   │   ├── panel/              # Top-level panels
│   │   │   ├── Editor.tsx      # Edit view (image canvas + toolbar + overlays)
│   │   │   ├── MainLibrary.tsx # Library grid
│   │   │   ├── FolderTree.tsx  # Left sidebar folder browser
│   │   │   ├── right/          # Right panel: Controls, Masks, Presets, AI, Export…
│   │   │   └── editor/         # Editor sub-components: ImageCanvas, Waveform, overlays
│   │   └── ui/                 # Reusable primitives: Slider, Switch, Dropdown, etc.
│   ├── hooks/                  # Custom hooks (history, thumbnails, keyboard shortcuts)
│   ├── context/                # React context (ContextMenuContext)
│   ├── utils/                  # Shared utilities (themes, palette, adjustments model)
│   └── window/                 # TitleBar.tsx — custom frameless window chrome
│
├── src-tauri/
│   ├── tauri.conf.json         # App identifier, window config, file associations
│   ├── Cargo.toml              # Rust dependencies
│   └── src/
│       ├── main.rs             # All #[tauri::command] handlers + AppState definition
│       ├── image_processing.rs # AllAdjustments struct, coordinate transforms, GpuContext
│       ├── gpu_processing.rs   # wgpu pipeline init, texture upload, shader dispatch
│       ├── shaders/
│       │   ├── shader.wgsl     # Main image processing shader (all adjustments)
│       │   ├── blur.wgsl       # Blur pass
│       │   └── flare.wgsl      # Lens flare effect
│       ├── raw_processing.rs   # RAW decoding via rawler
│       ├── image_loader.rs     # Image loading, thumbnail generation, in-memory cache
│       ├── mask_generation.rs  # AI mask inference (ONNX Runtime — SAM, sky, foreground)
│       ├── ai_processing.rs    # AI feature coordination
│       ├── ai_connector.rs     # ComfyUI WebSocket integration
│       ├── tagging.rs          # CLIP-based auto image tagging
│       ├── lens_correction.rs  # Lensfun DB — distortion/TCA/vignette correction
│       ├── exif_processing.rs  # EXIF read/write
│       ├── lut_processing.rs   # 3D LUT (.cube, .3dl, etc.)
│       ├── denoising.rs        # BM3D denoising
│       ├── inpainting.rs       # CPU-based local inpainting
│       ├── negative_conversion.rs  # Film negative conversion
│       ├── panorama_stitching.rs   # Panorama stitcher
│       ├── culling.rs          # Duplicate + blur detection
│       └── file_management.rs  # File system operations
│
└── rawler/rawler/              # Git submodule — RAW format decoder
```

---

## Core Concepts

### 1. Adjustments Data Model

Everything starts in **`src/utils/adjustments.tsx`**. This file defines every adjustment the app supports.

- Adjustments are grouped into TypeScript enums (`BasicAdjustment`, `ColorAdjustment`, `DetailsAdjustment`, `Effect`, `CreativeAdjustment`, `TransformAdjustment`, etc.).
- The `Adjustments` interface is the canonical type for a full set of image edits. It includes flat numeric fields plus complex nested structures: `curves`, `hsl`, `colorGrading`, `colorCalibration`, `masks`, `aiPatches`, and `crop`.
- `INITIAL_ADJUSTMENTS` defines default values for a new/unedited image.
- `normalizeLoadedAdjustments()` safely migrates adjustments loaded from disk, filling in missing fields with defaults. Always use it when loading saved edits.

When adding a new adjustment:
1. Add an entry to the appropriate enum in `adjustments.tsx`.
2. Add the field to the `Adjustments` interface with a default in `INITIAL_ADJUSTMENTS`.
3. Add it to the relevant `ADJUSTMENT_SECTIONS` entry so it appears in the UI panel.
4. Add the corresponding field to `AllAdjustments` in `src-tauri/src/image_processing.rs`.
5. Wire it into `shader.wgsl` (add to `GlobalAdjustments` struct and implement the effect).

### 2. The Edit Loop

Every slider change triggers this cycle:

```
User moves slider
    → setAdjustments() in App.tsx
    → debounced invoke(Invokes.GeneratePreviewForPath, { adjustments, ... })
    → Rust deserializes JSON into AllAdjustments
    → GPU pipeline runs shader with new uniforms
    → Binary image bytes returned via Tauri IPC Response
    → ImageCanvas.tsx renders the new frame
```

The preview path (live editing) uses a dedicated preview worker thread in Rust for low latency. Export-quality renders bypass the cache and use full resolution.

Auto-save writes the `Adjustments` object as JSON to a `.rrdata` sidecar file next to the original image after every change (debounced).

### 3. Tauri IPC

All frontend↔backend communication goes through `invoke()`:

```typescript
import { invoke } from '@tauri-apps/api/core';
import { Invokes } from './components/ui/AppProperties';

const result = await invoke(Invokes.LoadImage, { path: '/path/to/photo.cr3' });
```

All command names are enumerated in `Invokes` (in `src/components/ui/AppProperties.tsx`). Always use this enum — don't use raw string literals.

Images are transferred as raw bytes via Tauri's `Response` type (not Base64). The frontend creates an object URL from the bytes to display in an `<img>` tag.

For backend-to-frontend events, Rust uses `app_handle.emit("event-name", payload)` and the frontend uses `listen()`.

### 4. AppState (Rust)

The `AppState` struct in `main.rs` is the shared state across all Tauri commands. Key fields:

| Field | Purpose |
|---|---|
| `original_image` | The decoded full-resolution image currently loaded |
| `cached_preview` | Last rendered preview (avoids re-running GPU pipeline if adjustments haven't changed) |
| `gpu_context` | wgpu device/queue — lazily initialized on first use |
| `gpu_image_cache` | The original image uploaded as a GPU texture — expensive to re-upload |
| `lut_cache` | Parsed LUT data keyed by file path |
| `mask_cache` | Generated mask bitmaps keyed by content hash |
| `load_image_generation` | Atomic counter used as a cancellation token for in-flight image loads |
| `preview_worker_tx` | Channel sender for the dedicated preview worker thread |

All fields are wrapped in `Mutex` for thread safety (commands are `async` and run on Tokio).

### 5. GPU Shader

`src-tauri/src/shaders/shader.wgsl` is the heart of the image processing pipeline. It receives:
- The source image as a texture
- A `GlobalAdjustments` uniform buffer containing all numeric adjustment values
- LUT texture (if active)
- Mask textures (per mask layer)
- Curve data

The shader applies adjustments in a fixed order: color space conversion → exposure/tone → curves → HSL → color grading → detail enhancement → effects → output.

When adding a new shader-based adjustment, maintain `bytemuck` alignment requirements in the corresponding Rust struct (`AllAdjustments` in `image_processing.rs`) — all fields must be 4-byte aligned, with explicit padding if needed.

### 6. Masking System

Masks are stored as `MaskContainer` objects inside `adjustments.masks`. Each `MaskContainer`:
- Has its own `MaskAdjustments` (same structure as global adjustments minus transforms/crops)
- Contains an array of `SubMask` objects that define the selection (brush strokes, AI segmentation, gradients, etc.)
- Has `visible`, `invert`, `opacity` controls

`SubMask` types: `Brush`, `Linear`, `Radial`, `Color`, `Luminance`, `AiSubject`, `AiSky`, `AiForeground`, `All`.
`SubMaskMode`: `Additive` or `Subtractive` — masks can be combined.

On the Rust side, `mask_generation.rs` computes the final composite mask bitmap from the SubMask definitions and returns it as a grayscale image to be used as a blending weight in the shader.

### 7. History / Undo-Redo

The `useHistoryState` hook (`src/hooks/useHistoryState.tsx`) wraps any state value in an array of snapshots. Each call to `setState` appends a new snapshot. `undo`/`redo` move the index. The hook does a `JSON.stringify` equality check before pushing to avoid duplicate history entries.

In `App.tsx`, the main `adjustments` state is managed through `useHistoryState`. Every adjustment change (slider move, mask edit, etc.) automatically adds an undo step.

---

## Common Development Tasks

### Adding a New Adjustment Slider

1. **`src/utils/adjustments.tsx`** — Add to the relevant enum and to the `Adjustments` interface. Set a default in `INITIAL_ADJUSTMENTS`. Add to `ADJUSTMENT_SECTIONS` and `COPYABLE_ADJUSTMENT_KEYS` (if it should be copy-pasteable).
2. **`src/components/adjustments/<Panel>.tsx`** — Add a `<Slider>` wired to `handleAdjustmentChange(YourEnum.NewField, value)`.
3. **`src-tauri/src/image_processing.rs`** — Add the field to `AllAdjustments` (keep `bytemuck` alignment).
4. **`src-tauri/src/shaders/shader.wgsl`** — Add to `GlobalAdjustments` struct and implement the effect in the shader body.

### Adding a New Tauri Command

1. Write the `async fn` in `src-tauri/src/main.rs` (or a new module file), annotated with `#[tauri::command]`.
2. Register it in the `.invoke_handler(tauri::generate_handler![...])` call in `main()`.
3. Add the command name to the `Invokes` enum in `src/components/ui/AppProperties.tsx`.
4. Call it from the frontend: `await invoke(Invokes.YourCommand, { params })`.

### Working on the Shader

Edit `src-tauri/src/shaders/shader.wgsl`. Changes require a Rust recompile (the shader is embedded at compile time). Run `npm start` and the full app restarts.

If adding new uniform fields to `GlobalAdjustments` in the WGSL, mirror the change in the Rust `AllAdjustments` struct and ensure `bytemuck::Pod` alignment (add `_pad` fields as needed).

### Adding a New Modal

Create a new file in `src/components/modals/`. Follow the existing pattern: accept `isOpen: boolean` and `onClose: () => void` props, render conditionally, and communicate results via a callback prop. Register the modal in `App.tsx`.

---

## Known Architectural Limitations

- **Prop drilling** — `App.tsx` is large and passes many props deeply. This is a known issue (listed in the project's Current Priorities). When adding features, follow the existing pattern rather than introducing a new state management layer — a refactor is planned.
- **Coordinate transform fragmentation** — geometry transformations (crop, rotation, lens correction) are computed in multiple places. See [issue #245](https://github.com/CyberTimon/RapidRAW/issues/245) for the planned centralization.
- **X-Trans demosaicing** — the current Fuji X-Trans RAW demosaicing algorithm is a placeholder; a better algorithm is a medium-priority future task.

---

## Key Files Quick Reference

| File | What it contains |
|---|---|
| `src/App.tsx` | All app state, all `invoke()` calls, layout orchestration |
| `src/utils/adjustments.tsx` | Every adjustment type definition and default value |
| `src/components/ui/AppProperties.tsx` | `Invokes` enum, `AppSettings` interface, `ThemeProps` |
| `src/hooks/useHistoryState.tsx` | Undo/redo state management hook |
| `src/hooks/useKeyboardShortcuts.tsx` | All keyboard shortcut bindings |
| `src-tauri/src/main.rs` | Tauri command handlers, `AppState` definition |
| `src-tauri/src/image_processing.rs` | `AllAdjustments` struct, `GpuContext`, geometry math |
| `src-tauri/src/gpu_processing.rs` | wgpu pipeline, texture management |
| `src-tauri/src/shaders/shader.wgsl` | Full GPU image processing pipeline |
| `src-tauri/src/raw_processing.rs` | RAW file development (rawler integration) |
| `src-tauri/src/mask_generation.rs` | AI mask inference via ONNX Runtime |
