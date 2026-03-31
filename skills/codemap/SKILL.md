---
name: codemap
description: "Creates an interactive code map playground for any library or codebase architecture — an animated JointJS node graph with layer-colored nodes, typed connections, preset views, path highlighting, router toggle, and click-to-annotate notes. Use when the user asks to visualize a library's structure, explore a codebase architecture, map module relationships, or analyze a local project. Examples: 'create a code map for React', 'show me how Express.js is organized', 'build an architecture explorer for Vue 3', 'visualize the module relationships in JointJS', 'build a code map for this codebase', 'map out this project', 'explore the architecture of this repo', 'visualize this project structure', 'build a JointJS playground', 'create a JointJS code playground', 'jointjs playground'."
---

# Code Map Playground

Generates a self-contained HTML file with an animated JointJS graph showing library/codebase architecture: modules as nodes, dependencies as typed links, layer-filtered preset views, undirected BFS path highlighting, and a prompt output bar.

Two modes depending on whether the target is a **known library** (use training knowledge) or a **local codebase** (read files first).

---

## Mode 1: Known Library

Use when the user names a well-known library or framework (React, Express, Vue, JointJS, etc.) that you have training knowledge of.

1. **Load the template** from `../graph-explorer/templates/graph-explorer.md` — Shared Framework + Code Map Spec
2. **Pre-populate from knowledge** — real module names, file paths, key APIs, accurate code snippets, actual dependency connections
3. **Write the HTML file** — filename: `<library>-codemap.html`
4. **Open in browser** — `open <filename>-codemap.html`

---

## Mode 2: Local Codebase

Use when the user says "this codebase", "this project", "this repo", or gives a local path. Run a file analysis phase before writing the HTML.

### Phase 1 — Discover

```
Glob('src/**/*.{js,ts,jsx,tsx}')   # JS/TS projects
Glob('**/*.py')                     # Python
Glob('**/*.go')                     # Go
Glob('**/*.{rb,rake}')              # Ruby
```

- Identify the top-level structure: flat, `src/`, `lib/`, `packages/`, monorepo, etc.
- Count files per directory — directories with 3+ files become candidate layer groups
- Skip: `node_modules`, `dist`, `build`, `.git`, `__pycache__`, `vendor`, test files (`*.test.*`, `*.spec.*`, `*_test.*`)

### Phase 2 — Analyze

For each source file (or a representative sample if >60 files — prefer entry points, index files, and heavily-imported files):

**Extract per file:**
- Module name: last 1–2 path segments (e.g. `src/auth/middleware.ts` → `auth/middleware`)
- Exports: classes, functions, constants named in `export` statements
- Imports: what it imports and from where (internal paths only — skip `node_modules`)
- Key signatures: first exported class/function name, used as `keys[]` in the node

**Language patterns:**

| Language | Imports | Exports | Classes |
|---|---|---|---|
| JS/TS | `import X from './y'` | `export class/function/const` | `class X extends Y` |
| Python | `import x`, `from x import y` | top-level `def`/`class` | `class X(Y):` |
| Go | `import "pkg"` | capitalized identifiers | `type X struct` |
| Ruby | `require`, `require_relative` | module/class definitions | `class X < Y` |

**Build a dependency map:**
```
{ 'auth/middleware': { imports: ['auth/session', 'db/client'], exports: ['authenticate', 'requireRole'] } }
```

### Phase 3 — Reduce to 15–25 nodes

Large codebases need clustering — one node per directory, not per file:

- If total files ≤ 25: one node per file
- If total files 26–80: one node per directory (aggregate imports/exports across files in dir)
- If total files > 80: one node per top-level package/domain group

**Derive layers from directory structure:**
```
src/auth/      → layer: 'auth'
src/api/       → layer: 'api'
src/db/        → layer: 'db'
src/utils/     → layer: 'utils'
```

Assign a distinct color to each layer. Use the LAYERS palette from the Code Map Spec as a starting point — adapt key names to the actual directory names found.

**Derive connections:**
- `imports` from one module to another → `{ from, to, type: 'uses' }`
- `class X extends Y` across modules → `{ from, to, type: 'extends' }`
- Shared utility imported by many → `{ from, to, type: 'depends' }`
- De-duplicate: if A imports B three times (via three files in the same cluster), emit one link

**Pick DEFAULT_ANCHOR_ID:** the node with the highest in-degree (most things import it), or the entry point (`index`, `main`, `app`).

### Phase 4 — Layout

Arrange nodes in columns by layer — left to right roughly following dependency direction (most-depended-on layers on the left):

```js
// Assign x by layer column, y staggered within column
const LAYER_ORDER = ['db', 'models', 'services', 'api', 'utils']; // derive from topology
const COL_WIDTH = 220, ROW_HEIGHT = 80;
nodes.forEach((n, i) => {
  const col = LAYER_ORDER.indexOf(n.layer);
  const rowInCol = nodesPerLayer[n.layer].indexOf(n);
  n.x = col * COL_WIDTH + 60;
  n.y = rowInCol * ROW_HEIGHT + 60;
});
```

### Phase 5 — Generate

Write the HTML file using the graph-explorer template (Shared Framework + Code Map Spec), substituting:
- The extracted NODES array (with `desc` from file path + export list, `keys` from exported names, `snippet` from first meaningful export signature)
- The derived CONNECTIONS array
- The derived LAYERS object (keys = directory names, colors from palette)
- PRESETS: always include `full`; add 1–2 domain-specific presets (e.g. `{ auth layer only }`, `{ api + services }`)

**Filename:** `<project-name>-codemap.html` where project name = root directory name or `package.json` name field.

### Pitfalls for local codebase mode

| Problem | Fix |
|---|---|
| Too many nodes | Cluster by directory, not file. Cap at 25 nodes — merge smallest dirs into a `misc` or `shared` node |
| Circular imports | Still emit the link — they're real and worth visualizing. Cycles appear naturally in the graph |
| No clear layers | Fall back to: `core`, `services`, `utils`, `config` based on naming conventions |
| Monorepo with packages | Each `packages/<name>` is a layer; inter-package imports are the links |
| Missing imports (dynamic, aliased) | Note in the node `desc`: "dynamic imports not shown" |
| Large files | Read first 200 lines for import/export extraction, then also grep for `^export` / `^module.exports` to catch exports defined later in the file (e.g. default exports at the bottom, utility barrels) |

---

## Key features

- **ModuleNode** — single-color rect with label + file path subtitle + noteDot
- **Layers** — derived from directory structure or training knowledge, each with distinct fill/border/color
- **Connection types** — uses, extends, registers, depends — rendered with dash patterns and colors
- **Preset views** — Full + domain-specific (adapt to the actual structure found)
- **Path highlighting** — undirected BFS from hovered node to click-pinned anchor
- **Node detail modal** — description, exported names, code snippet, dependencies, note textarea

## Quick reference

- **Output**: single self-contained HTML file
- **CDN**: `https://cdn.jsdelivr.net/npm/@joint/core@4.2.4/dist/joint.min.js`
- **Template**: `../graph-explorer/templates/graph-explorer.md` → Shared Framework + Code Map Spec
- **Filename**: `<library-or-project>-codemap.html`
- **Node size**: 160×48px
- **Namespace**: `codemap.ModuleNode`, `codemap.Link`
