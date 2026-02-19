<!-- quikgraph-injected -->
# Quikgraph Integration

Quikgraph provides semantic code search and analysis. **Prefer CLI commands over MCP** — they're faster and provide richer output.

## Quick Reference

Use `qg` (short alias) instead of `quikgraph` for all commands.

### Smart Query (Auto Intent Detection)

| Need | Command |
|------|---------|
| Let Quikgraph decide query type | `qg query "your question" --agent` |

### Semantic Search (Use Instead of Grep/Glob)

| Need | Command |
|------|---------|
| Search code by meaning | `qg search "query" --code --agent` |
| Search docs/markdown | `qg search "query" --docs --agent` |
| Search by language | `qg search "query" --lang rust --agent` |

### Graph Analysis (Code Structure)

| Need | Command |
|------|---------|
| Who calls this function | `qg graph callers <symbol>` |
| What does this call | `qg graph callees <symbol>` |
| All references to symbol | `qg graph refs <symbol>` |
| Dependencies of symbol | `qg graph deps <symbol>` |
| What depends on this | `qg graph dependents <symbol>` |
| Path between symbols | `qg graph path <from> <to>` |

### Impact & Risk Analysis

| Need | Command |
|------|---------|
| Impact of uncommitted changes | `qg impact` |
| Full PR impact analysis | `qg impact-analyze --base main` |
| Which tests to run | `qg suggest-tests --base main` |

### Index Management

| Need | Command |
|------|---------|
| Check index status | `qg status` |
| Start/attach daemon | `qg watch .` |
| Health check | `qg doctor` |

## When to Use Quikgraph vs Built-in Tools

**Use `qg search --agent`** for: natural language queries, concept searches, finding code by behavior
**Use Grep** for: literal strings, exact regex patterns, known symbol names
**Use `qg graph`** for: callers, callees, dependencies, paths between symbols

## Important Notes

- **Always use `--agent` flag** for search/query — returns JSON optimized for LLMs
- **If Quikgraph not indexed**: Fall back to Grep/Glob

---

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
