# Site Explorer Template

Use this template when building a site explorer playground. The NODES array and LINKS array are pre-computed by Claude from the fetched sitemap — embed them directly in the HTML.

## Layout

```
[Top bar: site title · search · section pills │ depth │ router │ layout · Fit · count]
[Full-width JointJS canvas — tree fills the space]
[Bottom status bar: hovered title — URL · "Open page" button · freshness legend]
[#ctx-menu — fixed-position right-click menu, hidden by default]
```

No sidebar. No output panel. The tree IS the output.

---

## CSS Styling

```css
html, body { margin: 0; height: 100%; background: #0d1117; color: #c9d1d9; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; overflow: hidden; display: flex; flex-direction: column; }
#top-bar { display: flex; align-items: center; gap: 10px; padding: 10px 16px; background: #161b22; border-bottom: 1px solid #30363d; flex-wrap: wrap; }
#site-title { font-weight: 700; font-size: 14px; color: #e6edf3; white-space: nowrap; }
#search { background: #0d1117; border: 1px solid #30363d; border-radius: 6px; color: #c9d1d9; padding: 5px 10px; font-size: 12px; width: 180px; outline: none; }
#search:focus { border-color: #388bfd; }
.section-pill { padding: 3px 10px; border-radius: 12px; font-size: 11px; font-weight: 600; cursor: pointer; border: 1px solid transparent; opacity: 0.5; transition: opacity 0.15s; background: transparent; }
.section-pill.active { opacity: 1; border-color: currentColor; }
.depth-btn  { padding: 3px 10px; border-radius: 6px; font-size: 11px; cursor: pointer; background: #21262d; border: 1px solid #30363d; color: #8b949e; }
.depth-btn.active  { background: #1f6feb22; border-color: #388bfd; color: #79c0ff; }
.router-btn { padding: 3px 10px; border-radius: 6px; font-size: 11px; cursor: pointer; background: #21262d; border: 1px solid #30363d; color: #8b949e; }
.router-btn.active { background: #2d1f0a; border-color: #d97706; color: #fbbf24; }
.layout-btn { padding: 3px 10px; border-radius: 6px; font-size: 11px; cursor: pointer; background: #21262d; border: 1px solid #30363d; color: #8b949e; }
.layout-btn.active { background: #0d2318; border-color: #16a34a; color: #4ade80; }
#fit-btn { padding: 4px 12px; border-radius: 6px; font-size: 11px; cursor: pointer; background: #21262d; border: 1px solid #30363d; color: #8b949e; }
#canvas-wrap { flex: 1; position: relative; overflow: hidden; }
#paper-container { width: 100%; height: 100%; }
#status-bar { display: flex; align-items: center; gap: 12px; padding: 6px 16px; background: #161b22; border-top: 1px solid #30363d; font-size: 11px; color: #8b949e; }
#hover-url { flex: 1; font-family: 'SF Mono', Consolas, monospace; font-size: 10px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
#open-btn { padding: 3px 10px; border-radius: 6px; font-size: 11px; cursor: pointer; background: #1f6feb; border: none; color: #fff; display: none; }
/* Opacity fade driven by CSS; strokeWidth exit animates via JointJS transition */
.joint-element { transition: opacity 120ms cubic-bezier(0.66, 0, 0.34, 1); }
#paper-container.hl-exit .joint-element { transition: opacity 220ms cubic-bezier(0.66, 0, 0.34, 1); }
/* Context menu */
#ctx-menu { display: none; position: fixed; background: #161b22; border: 1px solid #30363d; border-radius: 6px; padding: 4px 0; z-index: 200; min-width: 130px; }
#ctx-menu div { padding: 7px 14px; cursor: pointer; font-size: 12px; color: #c9d1d9; }
#ctx-menu div:hover { background: #21262d; }
/* Freshness legend */
.fresh-legend { display: flex; align-items: center; gap: 8px; font-size: 10px; }
.fresh-dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
```

---

## Section color palette

```js
const SECTION_COLORS = [
  { color: '#818cf8', fill: '#1e2040', border: '#4f56a8' },
  { color: '#34d399', fill: '#122820', border: '#166534' },
  { color: '#fb923c', fill: '#271a10', border: '#9a3412' },
  { color: '#f472b6', fill: '#271025', border: '#9d174d' },
  { color: '#facc15', fill: '#272010', border: '#a16207' },
  { color: '#38bdf8', fill: '#0c1f2e', border: '#0369a1' },
  { color: '#a78bfa', fill: '#1c1530', border: '#6d28d9' },
  { color: '#94a3b8', fill: '#18202e', border: '#475569' },
];

// Build section → color map from unique sections in STATIC_NODES
const sections = [...new Set(STATIC_NODES.filter(n => n.depth === 1).map(n => n.section))];
const SECTION_COLOR = { '__root__': { color: '#e6edf3', fill: '#21262d', border: '#8b949e' } };
sections.forEach((s, i) => { SECTION_COLOR[s] = SECTION_COLORS[i % SECTION_COLORS.length]; });
```

---

## Freshness helper

```js
// Set TODAY to the date the HTML was generated (ISO string)
const TODAY = new Date('YYYY-MM-DD');
function freshnessColor(lastmod) {
  if (!lastmod) return null;
  const days = (TODAY - new Date(lastmod)) / 86400000;
  if (days < 30)  return '#22c55e'; // green  — recent
  if (days < 180) return '#eab308'; // yellow — aging
  return '#ef4444';                 // red    — stale
}
```

---

## NODES and LINKS data shape

Claude pre-populates `STATIC_NODES` and `STATIC_LINKS` from the fetched sitemap.
**Do not include summary nodes in STATIC_NODES** — they are generated dynamically.

### STATIC_NODES — one entry per real or ghost page segment

```js
const STATIC_NODES = [
  {
    id: '__root__',
    label: 'example.com',        // short display label
    title: 'Example — Home',     // full page <title> if fetchable, else null
    url: 'https://example.com',
    path: '/',
    depth: 0,
    section: '__root__',
    parentId: null,
    isGhost: false,              // true = no real page at this URL (virtual parent)
    isSummary: false,            // always false in STATIC_NODES
    lastmod: '2025-06-01',       // from sitemap <lastmod> or fetched page, else null
    status: 200,                 // HTTP status if fetchable, else null for ghost nodes
    x: 800, y: 60,              // pre-computed tree layout position (top-left of node)
  },
  // ... one entry per page/segment
];
```

### STATIC_LINKS — one entry per parent→child tree edge

```js
const STATIC_LINKS = [
  { from: '__root__', to: 'blog' },
  // Do NOT add a link to any summary node — getAllLinks() handles dynamic nodes
];
```

### CROSS_LINKS — hyperlinks between pages in different tree branches

Fetch each page and extract `<a href>` links. Add an entry when page A links to page B
**and** they are in different tree branches (not already connected by a STATIC_LINK).

```js
const CROSS_LINKS = [
  { from: 'blog', to: 'issue-42' },  // blog page lists this issue
  // ...
];
```

Cross-links are rendered as dashed blue links at 0.3 opacity.

### Dynamic generation — for sites with many paginated items (issues, posts, products)

If the site has hundreds of similar pages (e.g. /issue-NNN), embed only the most recent 5 in
STATIC_NODES and generate the rest dynamically:

```js
const INITIAL_SHOWN = 5;   // how many are in STATIC_NODES
const BATCH_SIZE    = 20;  // how many to add per expand click
const TOTAL_ITEMS   = 638; // total count of the paginated collection

function generateItemNode(num) {
  return {
    id: `issue-${num}`, label: `Issue #${num}`,
    title: `Site — Issue #${num}`,
    url: `https://example.com/issue-${num}`,
    path: `/issue-${num}`,
    depth: 2, section: 'issues', parentId: 'issues',
    isGhost: false, isSummary: false,
    lastmod: null, status: 200, x: 0, y: 0,
  };
}

function getAllNodes() {
  const nodes = STATIC_NODES.map(n => ({ ...n }));
  const existing = new Set(nodes.map(n => n.id));
  // Add dynamically generated items beyond INITIAL_SHOWN
  for (let i = TOTAL_ITEMS - INITIAL_SHOWN; i >= TOTAL_ITEMS - state.shownItemCount + 1; i--) {
    if (i < 1 || existing.has(`issue-${i}`)) continue;
    nodes.push(generateItemNode(i));
    existing.add(`issue-${i}`);
  }
  // Append summary node if items remain
  const remaining = TOTAL_ITEMS - state.shownItemCount;
  if (remaining > 0) {
    nodes.push({
      id: 'items-summary', label: `${remaining} more issues`,
      url: 'https://example.com/archives', path: '/archives',
      depth: 2, section: 'issues', parentId: 'issues',
      isGhost: true, isSummary: true,
      title: null, lastmod: null, status: null, x: 0, y: 0,
    });
  }
  return nodes;
}

function getAllLinks(allNodes) {
  const links = [...STATIC_LINKS];
  const existing = new Set(links.map(l => `${l.from}|${l.to}`));
  // Add links for dynamically generated nodes
  allNodes.filter(n => n.section === 'issues' && n.depth === 2).forEach(n => {
    const key = `issues|${n.id}`;
    if (!existing.has(key)) { links.push({ from: 'issues', to: n.id }); existing.add(key); }
  });
  return links;
}
```

If the site has no paginated collection, `getAllNodes()` just returns `STATIC_NODES.map(n => ({...n}))` and `getAllLinks()` returns `[...STATIC_LINKS]`.

---

## JointJS setup

```js
const PageNode = joint.dia.Element.define('sitemap.PageNode', {
  markup: [
    { tagName: 'rect',   selector: 'body' },
    { tagName: 'circle', selector: 'freshDot' }, // top-right: freshness indicator
    { tagName: 'circle', selector: 'pinMark' },  // top-left: pin indicator
    { tagName: 'text',   selector: 'label' },
    { tagName: 'text',   selector: 'sub' },
  ],
  size: { width: 150, height: 44 },
  attrs: {
    body:     { width: 'calc(w)', height: 'calc(h)', rx: 6, ry: 6, strokeWidth: 1.5 },
    freshDot: { r: 4, cx: 'calc(w - 7)', cy: 7, opacity: 0 },
    pinMark:  { r: 4, cx: 7, cy: 7, fill: '#facc15', opacity: 0 },
    label:    { x: 'calc(w/2)', y: 15, textAnchor: 'middle', dominantBaseline: 'middle', fontSize: 11, fontWeight: 600, textWrap: { width: -22, maxLineCount: 1, ellipsis: true } },
    sub:      { x: 'calc(w/2)', y: 32, textAnchor: 'middle', fontSize: 8, opacity: 0.45, fontFamily: "'SF Mono', Consolas, monospace", textWrap: { width: -8, maxLineCount: 1, ellipsis: true } },
  },
});

// Tree edges — solid, no arrow
const TreeLink = joint.shapes.standard.Link.define('sitemap.Link', {
  attrs: { line: { stroke: '#30363d', strokeWidth: 1, targetMarker: { type: 'none' } } },
});

// Cross-section hyperlinks — dashed blue, no arrow
const CrossLink = joint.shapes.standard.Link.define('sitemap.CrossLink', {
  attrs: { line: { stroke: '#3b82f6', strokeWidth: 1, strokeDasharray: '5 4', targetMarker: { type: 'none' }, opacity: 0.3 } },
});

const cellNamespace = { ...joint.shapes, sitemap: { PageNode, Link: TreeLink, CrossLink } };
const graph = new joint.dia.Graph({}, { cellNamespace });
const paper = new joint.dia.Paper({
  el: document.getElementById('paper-container'),
  model: graph,
  cellViewNamespace: cellNamespace,
  background: { color: '#0d1117' },
  gridSize: 1,
  width: '100%', height: '100%',  // REQUIRED — without this JointJS defaults to fixed 800×600
  // elementMove:true lets users drag nodes; links re-route automatically
  interactive: { elementMove: true, linkMove: false, labelMove: false },
});
```

---

## App state

```js
const state = {
  activeSection:  null,      // null = all sections visible
  maxDepth:       Infinity,  // depth toggle
  router:         'straight',// 'straight' | 'orthogonal' | 'metro'
  layout:         'tree',    // 'tree' | 'radial'
  collapsed:      new Set(['section-a', 'section-b']), // pre-collapse noisy sections (many pages with little hierarchy); use section IDs
  pinned:         new Map(), // nodeId → {x, y} — survives graph rebuilds
  shownItemCount: INITIAL_SHOWN, // omit if site has no paginated collection
};
```

---

## Layout — tree positions

Use a dynamic, **collapse-aware** tree layout so collapsed sections don't consume empty horizontal space. Call `computeTreeLayout(allNodes)` at the start of `buildGraph()`, then read positions from `treePos` in `getPos()`.

```js
const treePos = new Map();

// SECTION_ORDER controls left-to-right ordering of depth-1 sections
const SECTION_ORDER = ['section-a', 'section-b', /* ... all depth-1 ids in order */];

function computeTreeLayout(allNodes) {
  treePos.clear();
  const NW = 150, NH = 44, COLS = 4, CW = 174, CH = 74, SEC_GAP = 60, PAD = 60;
  const ROOT_Y = 60, SEC_Y = 220, CHILD_Y = 340;

  // Collect visible children per section (skip collapsed sections)
  const secChildren = {};
  SECTION_ORDER.forEach(s => { secChildren[s] = []; });
  allNodes.forEach(n => {
    if (n.depth === 2 && secChildren[n.parentId] !== undefined && !state.collapsed.has(n.parentId)) {
      secChildren[n.parentId].push(n);
    }
  });

  let curX = PAD;
  SECTION_ORDER.forEach(secId => {
    const kids = secChildren[secId] || [];
    // Collapsed sections get minimum width (1 col); expanded sections use up to COLS columns
    const cols = kids.length === 0 ? 1 : Math.min(COLS, kids.length);
    const gridW = cols * CW;
    const secCx = curX + gridW / 2;
    treePos.set(secId, { x: Math.round(secCx - NW / 2), y: SEC_Y });
    kids.forEach((n, i) => {
      treePos.set(n.id, { x: Math.round(curX + (i % cols) * CW), y: CHILD_Y + Math.floor(i / cols) * CH });
    });
    curX += gridW + SEC_GAP;
  });

  const totalW = curX - PAD - SEC_GAP;
  treePos.set('__root__', { x: Math.round(PAD + totalW / 2 - NW / 2), y: ROOT_Y });
}
```

Update `getPos()` to prefer `treePos`:

```js
function getPos(n) {
  if (state.pinned.has(n.id))          return state.pinned.get(n.id);
  if (state.layout === 'radial')        return radialPos.get(n.id) || { x: n.x, y: n.y };
  return treePos.get(n.id) || { x: n.x, y: n.y };
}
```

And in `buildGraph()`:

```js
if (state.layout === 'tree') computeTreeLayout(allNodes);
```

**Why collapse-aware matters:** with COLS=4 and 7 sections, a naïve layout allocates the full grid width per section regardless of collapse state — giving a canvas ~6000 px wide. Collapse-aware layout shrinks collapsed sections to one column each, keeping the canvas under ~2000 px.

---

## Layout — radial

```js
const radialPos = new Map();

function computeRadialLayout(allNodes) {
  const cx = 800, cy = 450, R1 = 230, R2 = 430, NW = 150, NH = 44;
  radialPos.clear();
  const root = allNodes.find(n => n.depth === 0);
  if (root) radialPos.set(root.id, { x: cx - NW/2, y: cy - NH/2 });

  const d1 = allNodes.filter(n => n.depth === 1);
  const sectorAngle = (2 * Math.PI) / Math.max(d1.length, 1);
  d1.forEach((n, i) => {
    const angle = -Math.PI/2 + i * sectorAngle;
    n._radialAngle = angle;
    n._sectorAngle = sectorAngle;
    radialPos.set(n.id, { x: Math.round(cx + R1*Math.cos(angle) - NW/2), y: Math.round(cy + R1*Math.sin(angle) - NH/2) });
  });

  const d2ByParent = {};
  allNodes.filter(n => n.depth === 2).forEach(n => {
    if (!d2ByParent[n.parentId]) d2ByParent[n.parentId] = [];
    d2ByParent[n.parentId].push(n);
  });
  d1.forEach(parent => {
    const children = d2ByParent[parent.id] || [];
    if (!children.length) return;
    const spread = Math.min(parent._sectorAngle * 0.8, Math.max(0.3, (children.length - 1) * 0.18));
    children.forEach((child, i) => {
      const offset = children.length > 1 ? -spread/2 + i/(children.length-1)*spread : 0;
      const angle = parent._radialAngle + offset;
      radialPos.set(child.id, { x: Math.round(cx + R2*Math.cos(angle) - NW/2), y: Math.round(cy + R2*Math.sin(angle) - NH/2) });
    });
  });
}
```

Position resolution — call this in `buildGraph()` before placing nodes:

```js
function getPos(n) {
  if (state.pinned.has(n.id))                           return state.pinned.get(n.id);
  if (state.layout === 'radial')                        return radialPos.get(n.id) || { x: n.x, y: n.y };
  if (clusterPos.size && clusterPos.has(n.id))          return clusterPos.get(n.id);
  return { x: n.x, y: n.y };
}
```

---

## Visibility — collapse + section + depth

```js
function getVisibleNodes(allNodes) {
  const nodeMap = new Map(allNodes.map(n => [n.id, n]));
  return allNodes.filter(n => {
    if (n.id === '__root__') return true;
    if (state.activeSection && n.section !== state.activeSection) return false;
    if (n.depth > state.maxDepth) return false;
    // Hidden if any ancestor is collapsed
    let cur = nodeMap.get(n.parentId);
    while (cur) {
      if (state.collapsed.has(cur.id)) return false;
      cur = nodeMap.get(cur.parentId);
    }
    return true;
  });
}

function countDescendants(nodeId, allNodes, nodeMap) {
  let count = 0;
  const stack = [nodeId];
  while (stack.length) {
    const id = stack.pop();
    allNodes.forEach(n => { if (n.parentId === id) { count++; stack.push(n.id); } });
  }
  return count;
}
```

---

## Building the graph

```js
// Pre-compute which nodes have children (determines collapse eligibility)
const parentIds = new Set(STATIC_LINKS.map(l => l.from));

function buildGraph() {
  graph.clear();
  const allNodes = getAllNodes();
  const nodeMap  = new Map(allNodes.map(n => [n.id, n]));
  const allLinks = getAllLinks(allNodes);

  // Compute layout positions
  if (state.layout === 'radial') {
    computeRadialLayout(allNodes);
  } else {
    // Compute compact cluster for any parent with >5 children (adapt section name)
    const clusterParentId = 'issues'; // or whichever section has many children
    const clusterParent = allNodes.find(n => n.id === clusterParentId);
    const clusterChildren = allNodes.filter(n => n.parentId === clusterParentId && n.depth === 2);
    if (clusterParent && clusterChildren.length > 0) computeCluster(clusterParent, clusterChildren);
  }

  const visible    = getVisibleNodes(allNodes);
  const visibleIds = new Set(visible.map(n => n.id));

  visible.forEach(n => {
    const col = SECTION_COLOR[n.section] || SECTION_COLOR['__root__'];
    const pos = getPos(n);
    const isCollapsed = state.collapsed.has(n.id);
    const isPinned    = state.pinned.has(n.id);
    const freshColor  = freshnessColor(n.lastmod);
    const childCount  = isCollapsed ? countDescendants(n.id, allNodes, nodeMap) : 0;

    const el = new PageNode({
      id: n.id,
      position: pos,
      attrs: {
        body: {
          fill:            n.isGhost ? '#0d1117' : col.fill,
          stroke:          isCollapsed ? col.color : (n.isGhost ? '#2d333b' : col.border),
          strokeDasharray: isCollapsed ? '6 3'    : (n.isGhost ? '4 3'    : null),
        },
        freshDot: { fill: freshColor || '#22c55e', opacity: freshColor ? 1 : 0 },
        pinMark:  { opacity: isPinned ? 1 : 0 },
        label: {
          text: isCollapsed ? `\u25b8 ${n.label}` : n.label,
          fill: n.isGhost ? '#484f58' : col.color,
        },
        sub: {
          text: isCollapsed ? `${childCount} pages hidden` : n.path,
          fill: '#484f58',
        },
      },
    });
    el.set('nodeData', n);
    graph.addCell(el);
  });

  const routerCfg = state.router === 'straight' ? { name: 'normal' } : { name: state.router };
  const connCfg   = state.router === 'metro'    ? { name: 'rounded', args: { radius: 8 } } : { name: 'normal' };

  // Tree links
  allLinks.forEach(({ from, to }) => {
    if (!visibleIds.has(from) || !visibleIds.has(to)) return;
    graph.addCell(new TreeLink({ source: { id: from }, target: { id: to }, router: routerCfg, connector: connCfg }));
  });

  // Cross-links (dashed blue — only between visible nodes)
  CROSS_LINKS.forEach(({ from, to }) => {
    if (!visibleIds.has(from) || !visibleIds.has(to)) return;
    graph.addCell(new CrossLink({ source: { id: from }, target: { id: to } }));
  });
}
```

---

## Collapse, expand, and summary

```js
function toggleCollapse(nodeId) {
  if (state.collapsed.has(nodeId)) state.collapsed.delete(nodeId);
  else state.collapsed.add(nodeId);
  buildGraph(); fitAll();
}

function expandSummary() {
  state.shownItemCount = Math.min(state.shownItemCount + BATCH_SIZE, TOTAL_ITEMS);
  buildGraph(); fitAll();
}

paper.on('element:pointerclick', (view) => {
  const nd = view.model.get('nodeData');
  if (!nd) return;
  if (nd.isSummary)          { expandSummary(); return; }
  if (parentIds.has(nd.id))  { toggleCollapse(nd.id); return; }
  if (!nd.isGhost)             window.open(nd.url, '_blank');
});
```

**Collapsed node visual:** dashed border in section color + label prefixed with `▸` + sub shows "N pages hidden".

---

## Pin nodes (right-click)

```js
let ctxNodeId = null;

paper.on('element:contextmenu', (view, evt) => {
  evt.preventDefault();
  const nd = view.model.get('nodeData'); if (!nd) return;
  ctxNodeId = nd.id;
  const menu = document.getElementById('ctx-menu');
  document.getElementById('ctx-pin').textContent = state.pinned.has(nd.id) ? 'Unpin node' : 'Pin node';
  const colEl = document.getElementById('ctx-collapse');
  colEl.textContent    = state.collapsed.has(nd.id) ? 'Expand' : 'Collapse';
  colEl.style.display  = parentIds.has(nd.id) ? '' : 'none';
  menu.style.display   = '';
  menu.style.left      = evt.clientX + 'px';
  menu.style.top       = evt.clientY + 'px';
});

document.addEventListener('pointerdown', e => {
  if (!e.target.closest('#ctx-menu')) document.getElementById('ctx-menu').style.display = 'none';
});

document.getElementById('ctx-pin').addEventListener('click', () => {
  if (!ctxNodeId) return;
  if (state.pinned.has(ctxNodeId)) { state.pinned.delete(ctxNodeId); }
  else { const el = graph.getCell(ctxNodeId); if (el) state.pinned.set(ctxNodeId, { ...el.position() }); }
  document.getElementById('ctx-menu').style.display = 'none';
  buildGraph();
});

document.getElementById('ctx-collapse').addEventListener('click', () => {
  if (ctxNodeId) toggleCollapse(ctxNodeId);
  document.getElementById('ctx-menu').style.display = 'none';
});

// Persist pinned position when the user drags a pinned node
graph.on('change:position', el => {
  const nd = el.get('nodeData');
  if (nd && state.pinned.has(nd.id)) state.pinned.set(nd.id, { ...el.position() });
});
```

**Pinned node visual:** yellow dot in top-left corner (`pinMark` opacity = 1). Position survives all `buildGraph()` calls until unpinned.

---

## Hover — animated ancestor highlight

```js
// cubic-bezier(0.66, 0, 0.34, 1) — hand-rolled because
// joint.util.timing.cubicBezier does NOT exist in JointJS 3.x
const easeHL = t => t < 0.5 ? 4*t*t*t : 1 - Math.pow(-2*t+2, 3)/2;

function getAncestorIds(nodeId, allNodes) {
  const nodeMap = new Map(allNodes.map(n => [n.id, n]));
  const ids = new Set([nodeId]);
  let cur = nodeMap.get(nodeId);
  while (cur && cur.parentId) { ids.add(cur.parentId); cur = nodeMap.get(cur.parentId); }
  return ids;
}

function highlightAncestors(hoveredEl) {
  const nd = hoveredEl.get('nodeData'); if (!nd) return;
  const ancestors = getAncestorIds(nd.id, getAllNodes());
  paper.el.classList.remove('hl-exit');
  paper.el.style.cursor = 'pointer';

  graph.getElements().forEach(el => {
    const enid   = el.get('nodeData')?.id;
    const onPath = ancestors.has(enid);
    const col    = SECTION_COLOR[el.get('nodeData')?.section] || SECTION_COLOR['__root__'];
    el.stopTransitions('attrs/body/strokeWidth');
    el.attr('root/opacity',      onPath ? 1 : 0.12);
    el.attr('body/stroke',       onPath ? col.color : col.border);
    el.attr('body/strokeWidth',  enid === nd.id ? 3 : onPath ? 2 : 1.5);
  });

  graph.getLinks().forEach(link => {
    const isCross = link.get('type') === 'sitemap.CrossLink';
    const onPath  = ancestors.has(link.source().id) && ancestors.has(link.target().id);
    const touches = ancestors.has(link.source().id) || ancestors.has(link.target().id);
    link.stopTransitions('attrs/line/opacity');
    if (isCross) {
      link.attr('line/opacity', onPath ? 0.8 : touches ? 0.4 : 0.05);
    } else {
      link.attr('line/opacity', onPath ? 0.85 : 0.05);
      link.attr('line/stroke',  onPath ? '#6b7280' : '#30363d');
    }
  });

  // Status bar — show title if available
  document.getElementById('hover-url').textContent = nd.title ? `${nd.title} — ${nd.url}` : nd.url;
  const btn = document.getElementById('open-btn');
  btn.style.display = nd.isGhost ? 'none' : '';
  if (!nd.isGhost) btn.onclick = () => window.open(nd.url, '_blank');
}

function clearHighlights() {
  paper.el.classList.add('hl-exit');
  paper.el.style.cursor = '';
  const T = { duration: 180, timingFunction: easeHL, valueFunction: joint.util.interpolate.number };
  graph.getElements().forEach(el => {
    const col = SECTION_COLOR[el.get('nodeData')?.section] || SECTION_COLOR['__root__'];
    el.attr('root/opacity', 1);
    el.attr('body/stroke', col.border);
    el.transition('attrs/body/strokeWidth', 1.5, T);
  });
  graph.getLinks().forEach(link => {
    const isCross = link.get('type') === 'sitemap.CrossLink';
    link.transition('attrs/line/opacity', isCross ? 0.25 : 0.4, { duration: 220, timingFunction: easeHL, valueFunction: joint.util.interpolate.number });
    if (!isCross) link.attr('line/stroke', '#30363d');
  });
  document.getElementById('hover-url').textContent = '';
  document.getElementById('open-btn').style.display = 'none';
}

paper.on('element:mouseenter', view => highlightAncestors(view.model));
paper.on('element:mouseleave', clearHighlights);
```

---

## Search

```js
function applySearch(q) {
  const term = q.toLowerCase();
  graph.getElements().forEach(el => {
    const nd = el.get('nodeData'); if (!nd) return;
    const matches = !term
      || nd.label.toLowerCase().includes(term)
      || nd.path.toLowerCase().includes(term)
      || (nd.url   && nd.url.toLowerCase().includes(term))
      || (nd.title && nd.title.toLowerCase().includes(term));
    el.attr('root/opacity', matches ? 1 : 0.07);
  });
}
```

---

## Top bar HTML

```html
<div id="top-bar">
  <span id="site-title"><!-- domain --></span>
  <input id="search" type="text" placeholder="Search pages…" oninput="applySearch(this.value)">
  <div id="section-pills" style="display:flex;gap:6px;flex-wrap:wrap"></div>
  <span style="color:#30363d;font-size:11px">│</span>
  <button class="depth-btn active"  onclick="setDepth(Infinity, this)">All</button>
  <button class="depth-btn"         onclick="setDepth(1, this)">Top</button>
  <button class="depth-btn"         onclick="setDepth(2, this)">2L</button>
  <span style="color:#30363d;font-size:11px">│</span>
  <button class="router-btn active" onclick="setRouter('straight', this)">straight</button>
  <button class="router-btn"        onclick="setRouter('orthogonal', this)">orthogonal</button>
  <button class="router-btn"        onclick="setRouter('metro', this)">metro</button>
  <span style="color:#30363d;font-size:11px">│</span>
  <button class="layout-btn active" onclick="setLayout('tree', this)">tree</button>
  <button class="layout-btn"        onclick="setLayout('radial', this)">radial</button>
  <button id="fit-btn" onclick="fitAll()">Fit</button>
  <span id="page-count" style="font-size:11px;color:#8b949e"></span>
</div>
```

Context menu and status bar:

```html
<div id="ctx-menu">
  <div id="ctx-pin">Pin node</div>
  <div id="ctx-collapse">Collapse</div>
</div>

<div id="status-bar">
  <span id="hover-url"></span>
  <button id="open-btn">Open page ↗</button>
  <span class="fresh-legend">
    <span class="fresh-dot" style="background:#22c55e"></span>recent
    <span class="fresh-dot" style="background:#eab308"></span>aging
    <span class="fresh-dot" style="background:#ef4444"></span>stale
  </span>
</div>
```

---

## Controls

```js
function buildSectionPills() {
  const container = document.getElementById('section-pills');
  const sections = [...new Set(STATIC_NODES.filter(n => n.depth === 1).map(n => n.section))];
  sections.forEach(s => {
    const col = SECTION_COLOR[s];
    const pill = document.createElement('button');
    pill.className = 'section-pill active';
    pill.textContent = s; pill.style.color = col.color; pill.dataset.section = s;
    pill.onclick = () => {
      state.activeSection = state.activeSection === s ? null : s;
      document.querySelectorAll('.section-pill').forEach(p =>
        p.classList.toggle('active', state.activeSection === null || p.dataset.section === state.activeSection));
      buildGraph(); fitAll();
    };
    container.appendChild(pill);
  });
}

function setDepth(d, btn) {
  state.maxDepth = d;
  document.querySelectorAll('.depth-btn').forEach(b => b.classList.remove('active'));
  if (btn) btn.classList.add('active');
  buildGraph(); fitAll();
}

function setRouter(name, btn) {
  state.router = name;
  document.querySelectorAll('.router-btn').forEach(b => b.classList.remove('active'));
  if (btn) btn.classList.add('active');
  // Apply to existing links without rebuilding — skip CrossLinks (always dashed)
  graph.getLinks().forEach(link => {
    if (link.get('type') === 'sitemap.CrossLink') return;
    link.set('router',    name === 'straight' ? { name: 'normal' } : { name });
    link.set('connector', name === 'metro'    ? { name: 'rounded', args: { radius: 8 } } : { name: 'normal' });
  });
}

function setLayout(name, btn) {
  state.layout = name;
  document.querySelectorAll('.layout-btn').forEach(b => b.classList.remove('active'));
  if (btn) btn.classList.add('active');
  buildGraph(); fitAll();
}

function fitAll() {
  paper.transformToFitContent({ padding: 60, minScaleX: 0.1, maxScaleX: 1.5 });
}
```

---

## Pan and zoom

```js
// Use evt.originalEvent to get native clientX/Y — the JointJS wrapper may not expose them directly
let _panOrigin = null;
paper.on('blank:pointerdown', evt => {
  const oe = evt.originalEvent;
  _panOrigin = { cx: oe.clientX, cy: oe.clientY, tx: paper.translate().tx, ty: paper.translate().ty };
});
// pointermove on document (screen coords) — constant speed regardless of zoom level
document.addEventListener('pointermove', e => {
  if (!_panOrigin) return;
  paper.translate(_panOrigin.tx + e.clientX - _panOrigin.cx, _panOrigin.ty + e.clientY - _panOrigin.cy);
});
document.addEventListener('pointerup', () => { _panOrigin = null; });

// Zoom at cursor point — attach to paper-container (not canvas-wrap) so the target matches paper.el
document.getElementById('paper-container').addEventListener('wheel', evt => {
  evt.preventDefault();
  const s = paper.scale().sx;
  const ns = Math.max(0.05, Math.min(3, s * (evt.deltaY < 0 ? 1.1 : 0.9)));
  const r = paper.el.getBoundingClientRect();
  const ox = evt.clientX - r.left, oy = evt.clientY - r.top;
  const { tx, ty } = paper.translate();
  paper.translate(ox - (ox - tx) * ns/s, oy - (oy - ty) * ns/s);
  paper.scale(ns, ns);
}, { passive: false });
```

---

## On load

```js
window.addEventListener('load', () => {
  const wrap = document.getElementById('canvas-wrap');
  paper.setDimensions(wrap.clientWidth, wrap.clientHeight);
  document.getElementById('site-title').textContent = BASE_URL.replace(/https?:\/\//, '');
  buildSectionPills();
  buildGraph();
  fitAll();
});
window.addEventListener('resize', () => {
  const wrap = document.getElementById('canvas-wrap');
  paper.setDimensions(wrap.clientWidth, wrap.clientHeight);
  fitAll();
});
```

---

## Common pitfalls

| Problem | Fix |
|---|---|
| All nodes overlap at origin | Pre-compute x/y in STATIC_NODES before writing the HTML — never leave them as 0 |
| Ghost nodes clickable | Check `nd.isGhost` in `element:pointerclick` and skip `window.open` |
| Summary node needs URL | Point it at the archives/index page so the "Open page" button is suppressed (isGhost:true) |
| Tree too wide to fit | Use collapse-aware `computeTreeLayout` (see Layout section) — collapsed sections get 1 column instead of their full grid allocation. Also pre-collapse the noisiest sections in `state.collapsed`. |
| `joint.util.timing.cubicBezier` is not a function | Does not exist in JointJS 3.x — use the hand-rolled `easeHL` above |
| Nodes not draggable | `interactive: false` disables everything — use `{ elementMove: true, linkMove: false }` |
| Hover flicker on stroke change | Call `el.stopTransitions('attrs/body/strokeWidth')` before setting attrs on enter |
| Router toggle rebuilds graph | Call `link.set('router', ...)` directly on existing links — no graph rebuild needed (skip CrossLinks) |
| CrossLink opacity not restored | In `clearHighlights`, check `link.get('type') === 'sitemap.CrossLink'` to restore to 0.25, not 0.4 |
| Collapse orphans root | Always return root from `getVisibleNodes` regardless of section/collapse state |
| Dynamic nodes lose links | Add dynamic links in `getAllLinks()` by checking against existing link set — never modify STATIC_LINKS |
| Pinned node resets on rebuild | Pass pinned position from `state.pinned.get(n.id)` in `getPos()` before checking layout |
| Radial layout crowds many children | The spread formula caps at `sectorAngle * 0.8` — with >10 children per parent, switch to tree layout |
| `TreeLink` needs markup | Extend `joint.shapes.standard.Link.define()`, NOT `joint.dia.Link.define()` |
| Paper canvas not full-size | Always pass `width: '100%', height: '100%'` to `new joint.dia.Paper({...})` — without these JointJS defaults to a fixed 800×600 SVG |
| `paper.scaleUniform` is not a function | This method does not exist in JointJS 3.x. Zoom at a point manually: `const { tx, ty } = paper.translate(); paper.translate(ox - (ox - tx) * ns / s, oy - (oy - ty) * ns / s); paper.scale(ns, ns);` |
| `joint.util.timing.cubicBezier` is not a function | This API does not exist in JointJS 3.x. Use a hand-rolled easing function: `t => t < 0.5 ? 4*t*t*t : 1 - Math.pow(-2*t+2,3)/2` (ease-in-out cubic). |

---

## Pre-populate checklist

Before writing the HTML, Claude must have:
- [ ] Fetched sitemap or root HTML — all URLs extracted and deduplicated
- [ ] `STATIC_NODES` with real x/y positions, `title`, `lastmod`, `status` on every node
- [ ] `STATIC_LINKS` — one entry per parent→child edge, **no summary link**
- [ ] `CROSS_LINKS` — fetched key pages and identified cross-section hyperlinks
- [ ] `SECTION_COLOR` map built from unique section values
- [ ] `TODAY` constant set to the generation date (for freshness calculation)
- [ ] `BASE_URL` constant set to the site root
- [ ] Node count check — aim for 20–150 visible nodes; use compact cluster or summary for more
- [ ] If paginated collection exists: `INITIAL_SHOWN`, `BATCH_SIZE`, `TOTAL_ITEMS` constants set
