# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ STALE FORK — READ BEFORE MAKING CHANGES

This is a **preserved fork** of [futurepress/epub.js](https://github.com/futurepress/epub.js), kept here in case the upstream library is abandoned or removed. **Do not treat this as active development code.**

Before modifying anything in this directory:
- Confirm the change cannot be made in the consuming project instead (e.g., Web or Mobile)
- Check whether the upstream library has already fixed the issue
- If a patch is truly necessary, document it clearly so it can be reapplied if the fork is ever refreshed
- Prefer minimal, targeted changes — avoid refactoring or modernizing

This fork is **not published** and is not consumed by any other project in this monorepo. It exists solely as a code preservation backup.

---

## What This Is

epub.js (v0.3) is a JavaScript library for rendering ePub documents in the browser. It parses EPUB3 files and renders them via iframes, supporting paginated and continuous scroll layouts, annotations, CFI-based navigation, and offline caching.

## Commands

```bash
npm install          # Install dependencies
npm start            # Start webpack-dev-server at localhost:8080 with examples
npm test             # Run tests via Karma + Mocha in ChromeHeadlessNoSandbox
npm run compile      # Transpile src/ → lib/ with Babel (CommonJS output)
npm run build        # Webpack bundle → dist/epub.js (UMD)
npm run minify       # Webpack bundle → dist/epub.min.js
npm run legacy       # Webpack bundle → dist/epub.legacy.js (wider browser targets)
npm run prepare      # Full release: compile + build + minify + legacy variants
npm run lint         # ESLint on src/
npm run docs         # Generate HTML docs from src/epub.js
```

Tests require Chrome to be installed (`ChromeHeadlessNoSandbox`). Tests run in-browser via Karma and are located in `test/*.js` with fixtures at `test/fixtures/`.

## Architecture

### Core Data Flow

```
ePub(url)                     → Book
Book.renderTo(element, opts)  → Rendition
Rendition.display(target)     → ViewManager → View(s) → Section(s)
```

### Key Classes (`src/`)

| File | Purpose |
|------|---------|
| `epub.js` | Entry point — exports `ePub()` factory and attaches classes as properties |
| `book.js` | Loads and parses the EPUB (container.xml, OPF, NCX/nav). Owns `spine`, `locations`, `navigation`, `resources`. |
| `rendition.js` | Coordinates display: picks the ViewManager, manages layout, themes, annotations, hooks. |
| `spine.js` | Ordered list of `Section` items parsed from the OPF manifest/spine. Hooks: `serialize`, `content`. |
| `section.js` | A single spine item (chapter). Handles loading, serializing, and providing document content. |
| `contents.js` | Wraps a rendered iframe's `document` — handles CFI resolution, link interception, resize events. |
| `epubcfi.js` | Full EpubCFI parser/generator for character-offset and range-based positions. |
| `layout.js` | Computes column/spread layout from OPF metadata and rendition options. |
| `locations.js` | Generates and persists CFI-based location percentages for progress tracking. |
| `navigation.js` | Parses NCX and EPUB3 nav documents into a tree of `NavItem`s. |
| `packaging.js` | Parses the OPF `.opf` file (manifest, spine, metadata, cover). |
| `annotations.js` | Manages highlights and underlines via `marks-pane`. |
| `themes.js` | Injects custom CSS/overrides into rendered views. |
| `store.js` | LocalForage-backed offline cache for book assets. |
| `archive.js` | JSZip-backed reader for `.epub` binary archives. |
| `resources.js` | Resolves and optionally replaces asset URLs (base64 or blob URLs) for archived books. |

### View Managers (`src/managers/`)

- **`default/`** — renders one section at a time in a single iframe
- **`continuous/`** — renders multiple sections to fill the viewport; enables seamless scroll/swipe
- **`views/iframe.js`** — the iframe-based view used by both managers
- **`views/inline.js`** — alternative inline view (less common)

### Hooks System

Hooks are synchronous or promise-returning functions registered on `Hook` instances (`src/utils/hook.js`). Key hook points:

```js
book.spine.hooks.serialize  // Section serialized to string
book.spine.hooks.content    // Section loaded and parsed as DOM
rendition.hooks.render      // Section rendered to screen
rendition.hooks.content     // Section contents loaded
rendition.hooks.unloaded    // Section being unloaded
```

### Build Outputs

`dist/` contains four build variants produced by `npm run prepare`:

| File | Description |
|------|-------------|
| `epub.js` | Standard UMD bundle |
| `epub.min.js` | Minified UMD bundle |
| `epub.legacy.js` | Wider browser targets (Babel `defaults`) |
| `epub.legacy.min.js` | Minified legacy bundle |

`lib/` contains the Babel-compiled CommonJS output from `npm run compile` (used as `"main"` in package.json).

JSZip is an **external** in webpack builds — consumers must include it separately. The `src/` ES module entry (`"module"` in package.json) bundles JSZip directly.

### Input Types

`Book` auto-detects input type but can be forced via `options.openAs`:

| Type | Description |
|------|-------------|
| `binary` | `.epub` ArrayBuffer via JSZip |
| `base64` | Base64-encoded epub |
| `epub` | URL to a `.epub` file |
| `opf` | URL to a `.opf` file (unzipped EPUB) |
| `json` | Manifest JSON |
| `directory` | Directory URL |

### CFI (Canonical Fragment Identifier)

EpubCFIs are the primary addressing scheme for positions within the book. The format `epubcfi(/6/4[chap01]!/4/10/2:3)` encodes spine position + DOM path + character offset. `EpubCFI` implements character-offset and simple-range CFIs; temporal/spatial offsets are not implemented.
