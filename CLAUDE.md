# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RapidRAW is a GPU-accelerated, non-destructive RAW image editor (Lightroom alternative) built with Tauri 2 (Rust backend + React/TypeScript frontend). All image processing happens on the GPU via wgpu/WGSL shaders. Edits are stored in `.rrdata` JSON sidecar files alongside original images.

## Commands

```bash
# Development (starts Vite dev server + Rust Tauri backend simultaneously)
npm start

# Frontend only (Vite)
npm run dev

# Build release
npm run tauri build

# Lint (ESLint)
npx eslint src/
```

`rawler` is a git submodule — clone with `--recurse-submodules` or run `git submodule update --init`.

## Architecture

### Frontend (`src/`)

- **`App.tsx`** — Monolithic root component holding all application state: current folder, selected images, adjustments, masks, history, settings, UI mode. It orchestrates all Tauri `invoke()` calls and passes state down via props. (Known issue: heavy prop drilling — see Current Priorities in README.)
- **`utils/adjustments.tsx`** — Single source of truth for all adjustment types. Defines TypeScript enums (`BasicAdjustment`, `ColorAdjustment`, `Effect`, etc.), the `Adjustments` interface, `INITIAL_ADJUSTMENTS`, and `MaskContainer`/`SubMask` types. Any new adjustment must be added here.
- **`hooks/useHistoryState.tsx`** — Custom undo/redo history hook wrapping state in an array. Used in `App.tsx` for the main adjustments history.
- **`hooks/useKeyboardShortcuts.tsx`** — All keyboard shortcut bindings.
- **`components/panel/`** — Top-level panels: `Editor.tsx` (edit view), `MainLibrary.tsx` (library grid), `FolderTree.tsx`, `BottomBar.tsx`, `SettingsPanel.tsx`, `CommunityPage.tsx`.
- **`components/panel/right/`** — Right sidebar panels: `ControlsPanel.tsx` (hosts adjustment sections), `MasksPanel.tsx`, `CropPanel.tsx`, `PresetsPanel.tsx`, `AIPanel.tsx`, `ExportPanel.tsx`, `MetadataPanel.tsx`.
- **`components/panel/editor/`** — Editor canvas area: `ImageCanvas.tsx` (renders processed image via Tauri IPC), `EditorToolbar.tsx`, `FullScreenViewer.tsx`, `Waveform.tsx`.
- **`components/adjustments/`** — Individual adjustment UI sections: `Basic.tsx`, `Color.tsx`, `Curves.tsx`, `Details.tsx`, `Effects.tsx`.
- **`components/ui/`** — Reusable primitives: `Slider.tsx`, `Switch.tsx`, `Dropdown.tsx`, `CollapsibleSection.tsx`, `ColorWheel.tsx`, `GlobalTooltip.tsx`, `Resizer.tsx`.
- **`utils/themes.tsx`** — Theme definitions (colors, CSS variables). `utils/palette.tsx` — palette extraction.
- **`window/TitleBar.tsx`** — Custom frameless window title bar.

### Backend (`src-tauri/src/`)

- **`main.rs`** — All `#[tauri::command]` handlers and `AppState` definition. `AppState` holds: `gpu_context` (lazy-init wgpu), `gpu_image_cache` (processed image cache), `thumbnail_cache`, `cancel_token` (for cancelling in-flight loads), preview worker thread channels, and settings.
- **`image_processing.rs`** — `AllAdjustments` struct (mirrors frontend JSON), `GpuContext` struct, geometry/crop math, orientation handling. This is the central data type passed from frontend to Rust on every render call.
- **`gpu_processing.rs`** — wgpu pipeline initialization (`get_or_init_gpu_context`), texture upload, shader dispatch, and readback. Processes images using `shader.wgsl`.
- **`raw_processing.rs`** — RAW file development via `rawler` (submodule). Handles demosaicing, orientation, highlight compression, linear RAW mode.
- **`image_loader.rs`** — Loads images (RAW/JPEG/etc), manages in-memory cache, generates thumbnails.
- **`mask_generation.rs`** — AI mask inference using ONNX Runtime (`ort` crate) for subject/sky/foreground segmentation.
- **`shaders/shader.wgsl`** — Main WGSL compute shader implementing the full adjustment pipeline (exposure, tone mapping, curves, HSL, color grading, noise reduction, effects, etc.).
- **`shaders/blur.wgsl`**, **`shaders/flare.wgsl`** — Auxiliary shaders for blur and lens flare effects.
- **`lens_correction.rs`** — Lensfun DB integration for distortion/TCA/vignette correction.
- **`tagging.rs`** / **`tagging_utils.rs`** — CLIP-based auto image tagging.
- **`exif_processing.rs`** — EXIF read/write via `kamadak-exif` and `little_exif`.
- **`ai_connector.rs`** — ComfyUI WebSocket integration for generative AI features.
- **`panorama_utils/`** — Panorama stitching math (feature detection, homography, blending).

### Key Data Flow

1. User adjusts a slider → `App.tsx` updates `adjustments` state → debounced call to `invoke('process_image', { adjustments, ... })`
2. Rust deserializes into `AllAdjustments`, runs GPU pipeline → returns image bytes via Tauri IPC `Response`
3. `ImageCanvas.tsx` renders the returned bytes as an `<img>` or canvas element
4. On save, adjustments are serialized to a `.rrdata` sidecar JSON file next to the original image

### Tauri IPC Pattern

All frontend↔backend communication uses `invoke()` from `@tauri-apps/api/core`. Images are transferred via Tauri's binary IPC (`Response` type), not Base64. The preview worker thread in `main.rs` handles live preview rendering separately from export-quality renders.
