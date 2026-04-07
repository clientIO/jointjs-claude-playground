# JointJS Code Playground — Claude Code Plugin

A Claude Code plugin to create custom JointJS-powered interactive playgrounds.

It enables you to build code maps for library/codebase architectures, data explorers for database schemas, canvas builders for drag-and-drop visual composers, diff reviewers with inline commenting and call graphs, and site explorers for interactive website/docs sitemaps.

## Demos
- [Codemap built with an interactive diagram](https://changelog.jointjs.com/gallery/jointjs-claude-playground/index.html)
- [SQL Query Builder](https://changelog.jointjs.com/gallery/jointjs-claude-playground/sql-query-builder.html)
- [Decision Tree Builder](https://changelog.jointjs.com/gallery/jointjs-claude-playground/decision-tree-builder.html)
- [Regex Builder](https://changelog.jointjs.com/gallery/jointjs-claude-playground/regex-builder.html)
- [Cron Builder](https://changelog.jointjs.com/gallery/jointjs-claude-playground/cron-builder.html)
- [Diff Review Tool](https://changelog.jointjs.com/gallery/jointjs-claude-playground/diff-review.html)
- [Visual Site Explorer](https://changelog.jointjs.com/gallery/jointjs-claude-playground/jointjs-site-explorer.html)


## Skills
Five JointJS-powered interactive playground skills: a **code map** for library/codebase architecture, a **data explorer** for database schemas, a **canvas builder** for drag-and-drop visual composers, a **diff reviewer** with inline commenting and call graphs, and a **site explorer** for interactive website/docs sitemaps.

### 1. `codemap` — Code Map Playground

When you ask Claude to visualize a library's structure — e.g. "create a code map for React" or "show me how Express.js is organized" — this skill generates a self-contained HTML file with:

- **Animated node graph** using JointJS 4.2 with layer-colored nodes and typed connections
- **Preset views** that animate between Full, Core, Rendering, and Geometry perspectives
- **Router toggle** to switch between Manhattan, Metro, Direct, and One Side live
- **Path highlighting** — hover any node to trace the shortest undirected path to the pinned anchor node
- **Click-to-comment** — click nodes to read docs, see code snippets, and add notes
- **Generated prompt** at the bottom — summarizes your active view and notes, ready to copy back into Claude

**Trigger phrases:**
- "Create a code map for JointJS 4.2"
- "Show me how the React codebase is structured as an interactive diagram"
- "Build an architecture explorer for Express.js"
- "Visualize the module relationships in Vue 3"

**Template:** `skills/graph-explorer/templates/graph-explorer.md` → Shared Framework + Code Map Spec

---

### 2. `data-explorer` — Data Explorer Playground

When you ask Claude to visualize a database schema or data model — e.g. "explore my Postgres schema" or "show entity relationships for our API" — this skill generates a self-contained HTML file with:

- **Entity nodes** with a two-tone header (domain color strip + body) and `PK: <field> · N fields` sublabel
- **Field-level search** — typing a field name or type highlights matching entities
- **Full field table in modal** — click any entity to see all fields with PK / FK / UQ / NN constraint badges, sample query, and note field
- **Relationship links** with FK column name labels (foreignKey, references, extends, contains, depends)
- Same animated transitions, router toggle, preset views, BFS path highlighting, and prompt output as the code map

**Trigger phrases:**
- "Visualize my database schema"
- "Create a data map for our API models"
- "Show me the entity relationships in our app"
- "Explore this Postgres schema as an interactive diagram"

**Template:** `skills/graph-explorer/templates/graph-explorer.md` → Shared Framework + Data Explorer Spec

---

### 3. `canvas-builder` — Interactive Visual Builder Playground

When you ask Claude to build any drag-and-drop visual composer — e.g. "build a regex builder", "create a SQL query builder", "make a decision tree" — this skill generates a self-contained HTML file with:

- **Categorized sidebar** of draggable items with accent-colored section headers
- **JointJS canvas** where users drag nodes, connect them with typed links, and edit properties via double-click overlay
- **Live output panel** with multiple format tabs (JSON, code, pseudocode) that regenerates on every graph change
- **Live test panel** for applicable builder types (regex, decision tree, cron)
- **Router toggle** (Direct / Orthogonal / Manhattan / Metro / One Side) + Fit + Clear toolbar
- **Pan & zoom** with pointer events and mouse wheel

**Builder types:**

| Type | Trigger phrases | Output |
|---|---|---|
| SQL Query Builder | sql, query, joins, database tables | `SELECT … FROM … JOIN … WHERE …` |
| Regex Builder | regex, regular expression, pattern matching | `/pattern/flags` with live test |
| API Designer | api, endpoints, routes, REST, OpenAPI | endpoint list / OpenAPI snippet |
| Data Pipeline | pipeline, ETL, data flow, transform | transformation pseudocode |
| Cron Builder | cron, schedule, recurring job, crontab | `0 9 * * 1-5` + human description + next runs |
| Decision Tree / Rule Engine | decision tree, rules, conditions, branching logic | JSON rules + JS if/else + pseudocode |
| GitHub Actions Builder | github actions, workflow, ci/cd, pipeline yaml | `.github/workflows/*.yml` |
| State Machine Builder | state machine, xstate, states, transitions, fsm | XState `createMachine({…})` config |
| Docker Compose Builder | docker compose, services, containers, networking | `docker-compose.yml` |

**Template:** `skills/canvas-builder/templates/canvas-builder.md`

---

### 4. `diff-review` — Diff Review with Call Graph

When you ask Claude to review a diff or PR — e.g. "review this PR", "show me what changed with a call graph" — this skill generates a self-contained HTML file with:

- **Line-by-line diff viewer** with syntax-highlighted add/remove/context lines
- **Inline commenting** — click any line to add a review comment
- **Hunk approve/reject** — review each change hunk independently with status tracking
- **JointJS call graph** alongside the diff — shows which functions changed and what calls them
- **Bidirectional linking** — click a graph node to scroll to its hunk; click a hunk's function label to highlight it in the graph
- **Filter pills** — All / Changed only / Callers views of the call graph
- **Router toggle** for the call graph

**Trigger phrases:**
- "Review this PR"
- "Show me what changed in this diff"
- "Create a code review tool with call graph"
- "Build a PR diff reviewer"
- "Visualize what functions changed and their callers"

**Template:** `skills/diff-review/SKILL.md`

---

### 5. `site-explorer` — Site Explorer Playground

When you ask Claude to map a website's structure — e.g. "create a site map for docs.jointjs.com" or "show me the page hierarchy of react.dev" — this skill fetches the sitemap and generates a self-contained HTML file with:

- **Zoomable JointJS tree** — every page as a node, URL hierarchy as parent/child links
- **Section colors** — top-level URL segments get distinct colors; the whole subtree inherits the section color
- **Ancestor path highlight** — hover a page node to trace its path back to the root
- **Click to open** — click any node to open the real page in a new tab
- **Search** — filter by page title or URL path segment
- **Section filter pills** — show/hide entire sections of the site
- **Depth toggle** — collapse to top-level only, 2 levels, or full tree

**Trigger phrases:**
- "Create a site map for docs.jointjs.com"
- "Build a website explorer for react.dev"
- "Show me the structure of the Next.js docs"
- "Map out this documentation site"
- "Visualize the page hierarchy of stripe.com/docs"

**Template:** `skills/site-explorer/templates/site-explorer.md`

---

## Installation

You can install the plugin through JointJS marketplace. First you need to add the marketplace, then you can install the plugin.

### Add JointJS Marketplace

```
/plugin marketplace add https://github.com/clientIO/jointjs-claude-marketplace
```

### Install JointJS Claude Playground plugin
```
/plugin install jointjs-claude-playground@jointjs
```

After installing the plugin, run `/reload-plugins` to enable the newly installed plugin.

Sometimes, the `/reload-plugins` command is unreliable, so you might need to restart Claude Code in your terminal—exit it and launch it again using `claude` if you don’t see plugin skills in the autocomplete list under `/jointjs` after installing and reloading the plugins.

#### A potential problem with installing Claude Code plugins

Note that there’s a bug in Claude Code where installing a Claude Code Plugin fails if you don’t have SSH keys configured on your system for GitHub.

When installing a plugin, Claude Code clones the repository via SSH (`git@github.com:…`), so if your GitHub SSH keys aren’t configured properly, the installation will fail.

Ideally, you should fix your GitHub SSH config, but as a workaround, before Claude Code resolves the problem, you can bypass SSH in your git config—run the following in your terminal:
```bash
git config --global url."https://github.com/".insteadOf git@github.com:
```

If you want to learn more about the bug in Claude Code, check the open issue on GitHub:
[Marketplace plugin cloning should default to HTTPS instead of SSH ](https://github.com/anthropics/claude-code/issues/26588)

## Output

A single `.html` file opened directly in your browser. No server, no build step.

## Filename conventions

- Code map: `<library>-codemap.html` — e.g. `react-codemap.html`
- Data explorer: `<schema>-data-explorer.html` — e.g. `ecommerce-data-explorer.html`
- Canvas builder: `<subject>-<type>.html` — e.g. `loan-decision-tree.html`, `email-regex-builder.html`
- Diff review: `<subject>-diff-review.html` — e.g. `auth-diff-review.html`
- Site explorer: `<domain>-site-explorer.html` — e.g. `docs-jointjs-site-explorer.html`
