# Image Converter Pro

A fast, fully offline, cross-platform desktop app for batch-converting images to WebP. Built with Tauri v2, React, TypeScript, and Rust.

![platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS-lightgrey)
![license](https://img.shields.io/badge/license-MIT-blue)

## Features

- **Batch conversion** of 10,000+ images, processed in parallel across every CPU core
- **Drag & drop** files or whole folders (recursive), or use the native file/folder pickers
- **Input formats:** JPG, JPEG, PNG, WebP, BMP, TIFF, GIF
- **WebP encoding** via Google's libwebp — Lossy, Lossless, and Near-lossless modes, quality 1–100
- **Image options:** resize (fit-within or exact dimensions), auto-rotate via EXIF, strip metadata, keep original timestamps
- **Folder structure** can be preserved in the output directory or flattened
- **Skip or replace** existing output files
- Live progress: current file(s), remaining count, elapsed time, ETA, throughput, converted/skipped/failed counts
- **Pause, resume, and cancel** mid-batch
- A plain-text conversion log is written to the output folder after each run
- Dark and light themes (or follow the OS)
- 100% offline — no network calls, no telemetry, no bundled fonts/CDNs

## Tech stack

| Layer | Technology |
|---|---|
| Shell | Tauri v2 |
| UI | React 18 + TypeScript + Vite |
| State | Zustand |
| Backend | Rust (rayon for parallelism, `image` + `webp` for the conversion pipeline, `kamadak-exif` for orientation) |

## Project structure

```
image-converter-pro/
├── src/                      # React frontend
│   ├── components/           # DropZone, ConversionQueue, Settings, ProgressPanel, Layout
│   ├── hooks/                # theme, settings persistence, drag-drop, conversion lifecycle
│   ├── store/                # Zustand app store
│   ├── types/                # TS types mirroring the Rust IPC models
│   └── utils/tauri.ts        # typed wrapper around Tauri commands/events
├── src-tauri/                # Rust backend
│   ├── src/
│   │   ├── models.rs         # shared IPC data types
│   │   ├── scanner.rs        # file/folder discovery
│   │   ├── converter.rs      # decode → orient → resize → encode pipeline
│   │   ├── engine.rs         # job orchestration, worker pool, pause/cancel, progress events
│   │   └── commands.rs       # Tauri command handlers
│   ├── icons/                # app icon source + generated icon set
│   └── tauri.conf.json
└── .github/workflows/        # CI and release automation
```

## Development

### Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [Rust](https://www.rust-lang.org/tools/install) (stable) with the MSVC toolchain on Windows
- Platform build tools — see [Tauri's prerequisites guide](https://v2.tauri.app/start/prerequisites/) for your OS

### Run locally

```bash
npm install
npm run tauri dev
```

This starts the Vite dev server and opens the app in a native window with hot reload.

### Type-check / lint

```bash
npm run build      # tsc -b && vite build
npm run lint
```

### Rust checks

```bash
cd src-tauri
cargo check
cargo clippy --all-targets -- -D warnings
cargo test
```

## Building installers locally

```bash
npm run tauri build
```

Produces a native installer for your current OS/architecture under `src-tauri/target/release/bundle/`. To target a specific architecture:

```bash
npm run tauri build -- --target aarch64-apple-darwin
npm run tauri build -- --target x86_64-pc-windows-msvc
npm run tauri build -- --target aarch64-pc-windows-msvc
```

Cross-compiling Windows/macOS targets from a different host OS is unreliable for this kind of app (native webview + libwebp linking); use the CI workflow below or build natively on each platform.

## CI/CD

- **`.github/workflows/ci.yml`** runs on every push/PR: type-checks and builds the frontend, then runs `cargo clippy`, `cargo check`, and `cargo test` for the backend on Linux, Windows, and macOS.
- **`.github/workflows/release.yml`** runs whenever a tag matching `v*` is pushed (e.g. `v1.0.0`), or can be triggered manually from the Actions tab for a test build. It builds and attaches to the matching GitHub Release:
  - Windows x64 installer (NSIS `.exe`) + portable `.zip`
  - Windows ARM64 installer (NSIS `.exe`) + portable `.zip`
  - macOS Intel `.dmg` + `.app.zip`
  - macOS Apple Silicon `.dmg` + `.app.zip`

See [`DISTRIBUTION.md`](./DISTRIBUTION.md) for how to cut a release and for optional code-signing setup.

## License

MIT — see [LICENSE](./LICENSE).
