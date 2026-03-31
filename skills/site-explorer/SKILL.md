---
name: site-explorer
description: "Creates an interactive website map — a zoomable JointJS tree showing every page of a documentation or marketing site, grouped by section, with click-to-open and search. Use when the user wants to visualize the structure of a website, explore a docs site, create a sitemap tree, or map out page hierarchy. Examples: 'create a site map for docs.jointjs.com', 'build a website explorer for react.dev', 'show me the structure of the Next.js docs', 'map out this documentation site', 'visualize the page hierarchy of stripe.com/docs', 'build a sitemap tree for our website'."
---

# Site Explorer Playground

Generates a self-contained HTML file with a zoomable JointJS tree showing every page of a website — pages as nodes colored by section, parent/child URL hierarchy as links, click-to-open, search, and section filter pills.

**Key difference from other skills:** Claude fetches the data (sitemap.xml or root page) during skill execution, then embeds it hardcoded in the HTML. The HTML itself makes no network requests.

## How to use this skill

1. **Fetch the sitemap** — see Phase 1 below
2. **Parse URLs into a tree** — Phase 2
3. **Compute layout** — Phase 3
4. **Load the template** from `templates/site-explorer.md` and write the HTML file
5. **Open in browser** — `open <filename>-site-explorer.html`

**Filename:** `<domain>-site-explorer.html` — e.g. `docs-jointjs-site-explorer.html`, `react-dev-site-explorer.html`

---

## Phase 1 — Fetch

Try in order, stop when you get data:

```
1. WebFetch('<baseUrl>/sitemap.xml')
2. WebFetch('<baseUrl>/sitemap_index.xml')  → lists sub-sitemaps, fetch each
3. WebFetch('<baseUrl>/sitemap/')
4. WebFetch(baseUrl)  → parse <a href> links from the HTML as fallback
```

Extract all `<loc>` values from XML, or all internal `href` values from HTML.
Filter to URLs that start with `baseUrl` — discard external links, anchors (`#`), and query strings.
Deduplicate. Aim for 20–200 URLs — if more, keep only paths with depth ≤ 3.

---

## Phase 2 — Build tree

```js
function buildTree(urls, baseUrl) {
  const nodes = new Map();
  const root = { id: '__root__', label: baseUrl.replace(/https?:\/\//, ''), url: baseUrl, depth: 0, children: [] };
  nodes.set('__root__', root);

  // Normalize: strip baseUrl, leading/trailing slashes
  const paths = urls
    .map(u => u.replace(baseUrl, '').replace(/^\/|\/$/g, ''))
    .filter(p => p.length > 0)
    .sort(); // sort so parents are always created before children

  paths.forEach(path => {
    const segments = path.split('/');
    segments.forEach((seg, i) => {
      const id = segments.slice(0, i + 1).join('/');
      if (nodes.has(id)) return;
      const parentId = i === 0 ? '__root__' : segments.slice(0, i).join('/');
      const node = {
        id,
        label: decodeURIComponent(seg).replace(/-/g, ' '),
        url: baseUrl + '/' + id,
        path: '/' + id,
        segment: seg,
        depth: i + 1,
        section: segments[0],  // top-level segment = section
        children: [],
      };
      nodes.set(id, node);
      const parent = nodes.get(parentId);
      if (parent) parent.children.push(node);
    });
  });

  return { root, nodes };
}
```

**Trim ghost nodes:** if a URL segment appears only as a parent (no actual page at that URL), mark it `isGhost: true` — render it smaller/dimmer.

---

## Phase 3 — Layout

Compute x/y positions with a simple level-order layout. Center each level horizontally:

```js
const NW = 150, NH = 44, H_GAP = 24, V_GAP = 72;

function layoutTree(root) {
  // BFS to collect levels
  const levels = [];
  const queue = [root];
  while (queue.length) {
    const node = queue.shift();
    if (!levels[node.depth]) levels[node.depth] = [];
    levels[node.depth].push(node);
    node.children.forEach(c => queue.push(c));
  }

  // Compute tree width from the widest level, then center all levels within it
  const maxLevelW = Math.max(...levels.map(lvl => lvl.length * NW + Math.max(0, lvl.length - 1) * H_GAP));
  const TREE_W = maxLevelW + 160; // padding on each side

  levels.forEach((level, depth) => {
    const totalW = level.length * NW + Math.max(0, level.length - 1) * H_GAP;
    const offsetX = (TREE_W - totalW) / 2;
    level.forEach((node, i) => {
      node.x = offsetX + i * (NW + H_GAP);
      node.y = depth * (NH + V_GAP) + 60;
    });
  });
}
```

For large trees (>80 nodes), collapse leaf-heavy subtrees: represent them as a single summary node ("14 pages") that expands on click.

---

## Template

See `templates/site-explorer.md` for the full HTML spec including:
- `PageNode` JointJS element definition
- Section color palette
- Search + section filter + depth toggle
- Ancestor path highlight on hover
- Click to open URL
- Pan/zoom
- Status bar

---

## Pitfalls

| Problem | Fix |
|---|---|
| Sitemap returns 404 | Fall back to fetching root page and parsing `<a href>` links |
| Sitemap index lists sub-sitemaps | Fetch each sub-sitemap and merge all `<loc>` values |
| Hundreds of URLs | Keep depth ≤ 3, or summarize leaf clusters into count nodes |
| URL segments are UUIDs/IDs | Label them as the parent + "#N" — they're likely dynamic routes |
| No clear section grouping | Use depth-1 segments as sections; if only one, use depth-2 |
| Ghost parent paths (no real page) | Render smaller, dimmer, no click handler |
