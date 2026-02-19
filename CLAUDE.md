# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PptxGenJS is a TypeScript library that generates PowerPoint (OOXML .pptx) files. It runs in Node.js, browsers, and web workers. The library creates standards-compliant Open Office XML files without requiring PowerPoint.

## Build Commands

```bash
npm run build          # Rollup build → outputs to src/bld/
npm run ship           # Full build + minify + bundle + copy to demo dirs (gulp)
npm start              # Same as ship (gulp default task)
npm run watch          # Rollup watch mode for development
npm run defs           # Copy type defs to vite-demo node_modules
```

There are no automated tests. Testing is manual — see `TESTING.md` for the full manual test matrix (browser, Node, web worker, Vite, Webpack).

## Linting

```bash
npx eslint src/        # ESLint 9 with typescript-eslint (config: eslint.config.mjs)
```

## Build Pipeline

1. **Rollup** (`rollup.config.mjs`) compiles TypeScript from `src/pptxgen.ts` (single entry point) into three formats in `src/bld/`:
   - `pptxgen.js` — IIFE (browser global `PptxGenJS`, JSZip as external)
   - `pptxgen.cjs.js` — CommonJS
   - `pptxgen.es.js` — ES module
2. **Gulp** (`gulpfile.js`) runs the `ship` pipeline: minify → bundle (IIFE + JSZip into single file) → prepend version headers → copy to `dist/` and demo directories

Final dist artifacts: `pptxgen.bundle.js`, `pptxgen.min.js`, `pptxgen.cjs.js`, `pptxgen.es.js`

## Architecture

### Source Files (`src/`)

All source is in `src/` — 10 TypeScript files, no subdirectories:

| File | Role |
|------|------|
| `pptxgen.ts` | **Main entry & public API.** The `PptxGenJS` class — presentation properties, `addSlide()`, `defineSlideMaster()`, `write()`/`writeFile()`, and the output/export pipeline using JSZip |
| `slide.ts` | `Slide` class — per-slide API (`addText`, `addImage`, `addChart`, `addTable`, `addShape`, `addMedia`, `addNotes`) that delegates to gen-objects |
| `core-enums.ts` | All constants (EMU, ONEPT, defaults), type aliases, and frozen enum-like objects (ChartType, ShapeType, SchemeColor, etc.) |
| `core-interfaces.ts` | All TypeScript interfaces and types for the public API and internal data structures |
| `gen-objects.ts` | Slide object creation — validates options and pushes typed objects onto `slide._slideObjects[]` |
| `gen-xml.ts` | XML string generation — converts slide objects into Open XML markup for slides, layouts, masters, and relationships |
| `gen-charts.ts` | Chart-specific XML generation — produces chart XML and embedded XLSX data sources |
| `gen-tables.ts` | Table layout engine — text-to-lines calculation, auto-paging for long tables across slides |
| `gen-media.ts` | Media encoding — base64 encodes images/audio/video, handles Node fs/https and browser fetch |
| `gen-utils.ts` | Shared utilities — unit conversions (inches↔EMU), color parsing, XML encoding, UUID generation |

### Key Patterns

- **Units**: PowerPoint uses EMU (914400 per inch) and DXA. The public API accepts inches; conversions happen internally via `gen-utils.ts`.
- **XML Generation**: The library builds raw XML strings (not DOM), concatenating them into the OOXML package structure. `gen-xml.ts` is the largest file.
- **JSZip**: The final .pptx is a ZIP file. `PptxGenJS.write()` assembles all XML parts into a JSZip instance and outputs as blob/buffer/base64/stream.
- **Slide Objects**: Each `slide.addX()` call pushes a typed object into `_slideObjects[]`. During `write()`, these are serialized to XML.
- **Slide Masters/Layouts**: Defined via `defineSlideMaster()`, stored as `SlideLayout` objects. Slides reference layouts by name.
- **Auto-Paging**: Tables can auto-page across multiple slides when content exceeds slide height (`gen-tables.ts`).
- **Environment Detection**: `gen-media.ts` lazy-loads Node built-ins (`node:fs`, `node:https`) only at runtime to stay browser-compatible.

### Type Definitions

`types/index.d.ts` is the hand-maintained public type definition file (published to npm). It mirrors but is separate from the internal `core-interfaces.ts`.

## TypeScript Config

- Target: ES2016, Module: ES2020
- `strictNullChecks: false` — intentionally disabled, many codepaths rely on this
- `noImplicitAny: false`

## Dependencies

Runtime: `jszip` (ZIP/OOXML packaging), `image-size` (Node-only image dimensions), `https` (Node-only remote images)
