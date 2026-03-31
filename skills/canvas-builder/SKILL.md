---
name: canvas-builder
description: "Creates any interactive visual builder playground — a drag-and-drop JointJS canvas where users compose nodes from a sidebar to build a structured output (SQL, regex, cron expression, decision tree, API spec, pipeline, etc.). Also handles diff/code-review tools with embedded JointJS call graphs. Use when the user asks to build an interactive builder, visual designer, drag-and-drop tool, canvas-based composer, or code reviewer for ANY domain. Examples: 'build a regex builder', 'build a regex explorer', 'build a JointJS builder playground', 'create a JointJS visual playground', 'create a visual API designer', 'make a data pipeline builder', 'build a SQL query builder', 'create a cron schedule builder', 'build a decision tree', 'create a rule engine', 'build a PR diff reviewer', 'create a GitHub Actions workflow builder', 'build a state machine', 'make a docker compose builder'."
---

# Canvas Builder Playground

A self-contained HTML playground built on a shared framework: a categorized sidebar of draggable items, a JointJS canvas where users compose nodes and connect them, and an output panel that generates the target artifact live. The framework is domain-agnostic — what changes between builder types is the sidebar items, node definitions, link semantics, and output generator.

## How to use this skill

1. **Identify the builder type** from the user's request — SQL query, regex, cron, decision tree, API design, data pipeline, GitHub Actions, state machine, etc.
2. **Load the template** from `templates/canvas-builder.md` — it defines the shared framework (layout, JointJS patterns, drag-drop, pan/zoom, router toggle, sidebar data pattern, graph event wiring, output tabs, live test) and full specifications for each builder type
3. **Select or adapt the matching builder-type spec** from the template — use the documented spec for known types; for novel types, derive node/link/output patterns following the same conventions
4. **Pre-populate with real content** relevant to the user's domain (real table schemas, real API routes, real regex patterns, real cron schedules, etc.)
5. **Write the single HTML file** following the template exactly — do not deviate from slot-based markup, port conventions, or the output-generation pattern
6. **Open in browser** — run `open <filename>.html` after writing

## Builder types covered

| Type | Trigger phrases | Output |
|---|---|---|
| SQL Query Builder | sql, query, joins, database tables | `SELECT … FROM … JOIN … WHERE …` |
| Regex Builder | regex, regular expression, pattern matching | `/pattern/flags` with live test |
| API Designer | api, endpoints, routes, REST, OpenAPI | endpoint list / OpenAPI snippet |
| Data Pipeline | pipeline, ETL, data flow, transform | transformation pseudocode |
| Cron Builder | cron, schedule, recurring job, crontab | `0 9 * * 1-5` + human description + next runs |
| Decision Tree / Rule Engine | decision tree, rules, conditions, branching logic, rule engine | JSON rules + JS if/else + pseudocode |
| GitHub Actions Builder | github actions, workflow, ci/cd, pipeline yaml | `.github/workflows/*.yml` |
| State Machine Builder | state machine, xstate, states, transitions, fsm | XState `createMachine({…})` config |
| Docker Compose Builder | docker compose, services, containers, networking | `docker-compose.yml` |

## Quick reference

- **Output**: single self-contained HTML file (all JS/CSS inline, JointJS from CDN)
- **CDN**: `https://cdn.jsdelivr.net/npm/@joint/core@4.2.4/dist/joint.min.js`
- **Layout**: `[Sidebar 240px] | [Canvas flex-1] | [Output panel 300px]` + prompt bar
- **Framework**: shared drag-drop, sidebar data object, slot-based nodes, port wiring, pan/zoom (pointer events), router toggle via `buildToolbar()`, graph event wiring, node editing overlay, output tabs, live test panel, `populateExample()`
- **Filename convention**: `<subject>-<type>.html` — e.g. `ecommerce-query-builder.html`, `email-regex-builder.html`, `shop-api-designer.html`, `loan-decision-tree.html`
