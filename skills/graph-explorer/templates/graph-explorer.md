# Graph Explorer — Unified Template

Shared framework for `codemap` and `data-explorer`. Both skills use the same JointJS layout, animation system, router toggle, path highlighting, pan/zoom, and on-load sequence. Only the node type, data schemas, layers, modal content, and prompt output differ — those are in the domain specs below.

When building either skill:
1. Use every section under **Shared Framework** verbatim
2. Slot in the matching **Domain Spec** for node type, schemas, layers, modal, and prompt
3. Pre-populate with real data from the user's domain

---

## Shared Framework

### Layout

```
+------------------+-----------------------------+
| Presets          | Canvas (graph diagram)      |
| Layer toggles    |   search bar (top-left)     |
| Conn toggles     |   "Powered by JointJS"(top-right)
| Router toggle    |   zoom controls (btm-right) |
| "Fit View"/"Reset Layout" btns |legend(btm-left)|
| Notes list       |                             |
+------------------+-----------------------------+
|  Generated prompt textarea        [Copy]       |
+------------------------------------------------+
```

Sidebar: 260px, dark background. Canvas: flex-1, JointJS paper mounted inside. Bottom bar: fixed-height prompt output.

### CSS Styling

```css
html { color-scheme: dark; }
/* corner-shape: squircle; with rounded borders */
/* Animate path highlight fade-out on hover leave */
.joint-element { transition: opacity 120ms cubic-bezier(0.66, 0, 0.34, 1); }
```

Data Explorer additionally needs field table styles — see Data Explorer spec.

### Library & graph setup

- Load `https://cdn.jsdelivr.net/npm/@joint/core@4.2.4/dist/joint.min.js`
- Define a custom node type — see each domain spec for the exact markup and attrs
- Define a custom link by extending standard (required to inherit line/wrapper markup):
  ```js
  // Code Map:
  const CodeMapLink = joint.shapes.standard.Link.define('codemap.Link', {});
  // Data Explorer:
  const DataMapLink = joint.shapes.standard.Link.define('datamap.Link', {});
  ```
- Start nodes invisible and park them above canvas so they fly in on load:
  ```js
  const el = new <NodeType>({
    id: n.id,
    position: stagePos(n),   // { x: n.x, y: -240 }
    attrs: { root: { opacity: 0 }, ... },
  });
  el.set('nodeData', n);
  ```
- Create a shared cell namespace and pass it to both graph and paper:
  ```js
  // Code Map:
  const cellNamespace = { ...joint.shapes, codemap: { ModuleNode, Link: CodeMapLink } };
  // Data Explorer:
  const cellNamespace = { ...joint.shapes, datamap: { EntityNode, Link: DataMapLink } };

  const graph = new joint.dia.Graph({}, { cellNamespace });
  const paper = new joint.dia.Paper({
    el: paperContainer,
    model: graph,
    cellViewNamespace: cellNamespace,
    width: '100%', height: '100%',
    background: { color: '#0d1117' },
    gridSize: 1,
    interactive: { linkMove: false },
  });
  ```

### State

```js
const state = {
  layers: { ...PRESETS.full.layers },
  conns:  { ...PRESETS.full.conns },
  router: 'manhattan',
  zoom: 0.7,
  comments: [],
  search: '',
  activePreset: 'full',
};
```

### Visibility helpers

Always define both. `animateToState` uses `getLayerIds` (layer only). `getVisibleIds` (layer + search) is used only by path highlighting. Search uses `applySearchFilter` (opacity only, no position changes).

```js
function getLayerIds() {
  const ids = new Set();
  NODES.forEach(n => { if (state.layers[n.layer]) ids.add(n.id); });
  return ids;
}

// Code Map — match label and sub (file path):
function getVisibleIds() {
  const ids = getLayerIds();
  if (state.search) {
    const q = state.search.toLowerCase();
    [...ids].forEach(id => {
      const n = NODES.find(x => x.id === id);
      if (!n.label.toLowerCase().includes(q) && !n.sub.toLowerCase().includes(q)) ids.delete(id);
    });
  }
  return ids;
}

// Data Explorer — also match field names and types:
function getVisibleIds() {
  const ids = getLayerIds();
  if (state.search) {
    const q = state.search.toLowerCase();
    [...ids].forEach(id => {
      const n = NODES.find(x => x.id === id);
      const fieldMatch = n.fields.some(f => f.name.toLowerCase().includes(q) || f.type.toLowerCase().includes(q));
      if (!n.label.toLowerCase().includes(q) && !fieldMatch) ids.delete(id);
    });
  }
  return ids;
}
```

### Search

**Never call `animateToState()` from search** — it snaps all nodes back to their data positions and destroys custom layouts. Use `applySearchFilter()` instead:

```js
function handleSearch(val) {
  state.search = val;
  applySearchFilter();
}

// withFit=true (default): also animate viewport to matching nodes.
// withFit=false: opacity/links only — used by animateToState after layer changes.
function applySearchFilter(withFit = true) {
  const layerIds = getLayerIds();
  const q = state.search.toLowerCase();
  graph.getElements().forEach(el => {
    const nd = el.get('nodeData'); if (!nd || !layerIds.has(nd.id)) return;
    const matches = !state.search || nd.label.toLowerCase().includes(q) || nd.sub.toLowerCase().includes(q);
    el.stopTransitions('attrs/root/opacity');
    el.attr('root/opacity', matches ? 1 : 0.1);
  });
  graph.getLinks().forEach(link => {
    const c = link.get('connData');
    link.stopTransitions('attrs/line/opacity');
    if (!c || !state.conns[c.type] || !layerIds.has(c.from) || !layerIds.has(c.to)) {
      link.attr('line/opacity', 0); return;
    }
    if (!state.search) { link.attr('line/opacity', 0.55); return; }
    const fromNode = NODES.find(n => n.id === c.from);
    const toNode   = NODES.find(n => n.id === c.to);
    const fm = fromNode && (fromNode.label.toLowerCase().includes(q) || fromNode.sub?.toLowerCase().includes(q));
    const tm = toNode   && (toNode.label.toLowerCase().includes(q)   || toNode.sub?.toLowerCase().includes(q));
    link.attr('line/opacity', fm && tm ? 0.55 : 0.06);
  });
  if (!withFit) return;
  const matchIds = state.search ? getVisibleIds() : layerIds;
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  graph.getElements().forEach(el => {
    const nd = el.get('nodeData');
    if (!nd || !matchIds.has(nd.id)) return;
    const { x, y } = el.position();
    const { width, height } = el.size();
    minX = Math.min(minX, x); minY = Math.min(minY, y);
    maxX = Math.max(maxX, x + width); maxY = Math.max(maxY, y + height);
  });
  if (minX !== Infinity) animateFit(new joint.g.Rect(minX - 40, minY - 40, maxX - minX + 80, maxY - minY + 80));
}
```

### Animated transitions

```js
const ANIM_MS  = 220;
const STAGGER  = 40;
const FIT_MS   = 320;
const easeOut  = t => { const c1 = 1.4, c3 = c1 + 1; return 1 + c3 * Math.pow(t-1,3) + c1 * Math.pow(t-1,2); };
const easeIn   = t => t * t * t;
const easeHighlight = (() => {
  const [p1x, p1y, p2x, p2y] = [0.66, 0, 0.34, 1];
  const A = (a1, a2) => 1 - 3*a2 + 3*a1, B = (a1, a2) => 3*a2 - 6*a1, C = a1 => 3*a1;
  const bez  = (t, a1, a2) => ((A(a1,a2)*t + B(a1,a2))*t + C(a1))*t;
  const dBez = (t, a1, a2) => 3*A(a1,a2)*t*t + 2*B(a1,a2)*t + C(a1);
  const tForX = x => { let t = x; for (let i = 0; i < 8; i++) { const s = dBez(t,p1x,p2x); if (Math.abs(s) < 1e-6) break; t -= (bez(t,p1x,p2x)-x)/s; } return t; };
  return x => bez(tForX(x), p1y, p2y);
})();
const stagePos = n => ({ x: n.x, y: -240 });

function animateToState() {
  selectedElementId = null;
  clearPathHighlights(false);
  const visibleIds = getLayerIds();

  graph.getLinks().forEach(link => link.attr('line/opacity', 0));

  const appearing    = [];
  const disappearing = [];
  graph.getElements().forEach(el => {
    const nd = el.get('nodeData');
    if (!nd) return;
    (visibleIds.has(nd.id) ? appearing : disappearing).push({ el, nd });
  });

  const byX = (a, b) => a.nd.x - b.nd.x;
  appearing.sort(byX);
  disappearing.sort(byX);

  const opT = { timingFunction: joint.util.timing.linear, valueFunction: joint.util.interpolate.number };

  disappearing.forEach(({ el, nd }, i) => {
    el.transition('position', stagePos(nd), {
      delay: i * STAGGER, duration: ANIM_MS, timingFunction: easeIn,
      valueFunction: joint.util.interpolate.object,
    });
    el.transition('attrs/root/opacity', 0, { delay: i * STAGGER, duration: ANIM_MS, ...opT });
  });

  const visibleNodes = NODES.filter(n => visibleIds.has(n.id));
  const appearMs = appearing.length * STAGGER + ANIM_MS;

  if (visibleNodes.length > 0) {
    animateFit(rectFromNodeData(visibleNodes), FIT_MS, () => {
      appearing.forEach(({ el, nd }, i) => {
        el.transition('position', { x: nd.x, y: nd.y }, {
          delay: i * STAGGER, duration: ANIM_MS, timingFunction: easeOut,
          valueFunction: joint.util.interpolate.object,
        });
        el.transition('attrs/root/opacity', 1, { delay: i * STAGGER, duration: ANIM_MS, ...opT });
      });
      setTimeout(() => {
        graph.getLinks().forEach(link => {
          const c = link.get('connData');
          const show = c && state.conns[c.type] && visibleIds.has(c.from) && visibleIds.has(c.to);
          link.transition('attrs/line/opacity', show ? 0.55 : 0, {
            duration: 320, timingFunction: joint.util.timing.linear,
            valueFunction: joint.util.interpolate.number,
          });
        });
        if (state.search) applySearchFilter(false);
      }, appearMs);
    });
  }

  return FIT_MS + appearMs;
}
```

### Animated viewport fit

Do NOT use the live SVG bbox — it includes staged/off-screen nodes. Compute the content area from NODES data positions, then animate the viewport transform:

```js
function rectFromNodeData(nodes) {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  nodes.forEach(n => {
    minX = Math.min(minX, n.x); minY = Math.min(minY, n.y);
    maxX = Math.max(maxX, n.x + 160); maxY = Math.max(maxY, n.y + 48);
  });
  return new joint.g.Rect(minX - 40, minY - 40, maxX - minX + 80, maxY - minY + 80);
}

function animateFit(contentArea, duration = 600, onComplete = null) {
  const fromScale = paper.scale().sx;
  const fromTrans = paper.translate();

  paper.transformToFitContent({ padding: 50, minScaleX: 0.2, maxScaleX: 1.4, contentArea });
  const toScale = paper.scale().sx;
  const pw = paper.options.width;
  const ph = paper.options.height;
  const toTx = (pw - contentArea.width  * toScale) / 2 - contentArea.x * toScale;
  const toTy = (ph - contentArea.height * toScale) / 2 - contentArea.y * toScale;
  paper.translate(toTx, toTy);
  const toTrans = paper.translate();

  paper.scale(fromScale, fromScale);
  paper.translate(fromTrans.tx, fromTrans.ty);

  const start = performance.now();
  const ease  = t => 1 + (1.4 + 1) * Math.pow(t-1,3) + 1.4 * Math.pow(t-1,2);
  (function tick(now) {
    const t = Math.min((now - start) / duration, 1);
    const e = ease(t);
    paper.scale(fromScale + (toScale - fromScale) * e, fromScale + (toScale - fromScale) * e);
    paper.translate(fromTrans.tx + (toTrans.tx - fromTrans.tx) * e, fromTrans.ty + (toTrans.ty - fromTrans.ty) * e);
    if (t < 1) requestAnimationFrame(tick);
    else if (onComplete) onComplete();
  })(start);
}

function fitVisibleContent() {
  const visibleIds = getVisibleIds();
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  graph.getElements().forEach(el => {
    const nd = el.get('nodeData');
    if (!nd || !visibleIds.has(nd.id)) return;
    const { x, y } = el.position();
    const { width, height } = el.size();
    minX = Math.min(minX, x); minY = Math.min(minY, y);
    maxX = Math.max(maxX, x + width); maxY = Math.max(maxY, y + height);
  });
  if (minX === Infinity) return;
  animateFit(new joint.g.Rect(minX - 40, minY - 40, maxX - minX + 80, maxY - minY + 80));
}
```

`animateToState()` calls `animateFit` internally, so call sites just call it directly:
```js
animateToState();
```

### Fit View / Reset Layout buttons

Add two buttons to the sidebar below the router toggle:
- **"Fit View"** — calls `fitVisibleContent()`
- **"Reset Layout"** — resets state to the full preset and calls `animateToState()`

### Router toggle

5 options in a 2-column grid. Use `'normal'` for direct lines — NOT `'straight'` (that's a connector):

```js
const ROUTERS = [
  { name: 'manhattan',  label: 'Manhattan',  desc: 'Right-angles, avoids obstacles', args: { padding: 18 } },
  { name: 'orthogonal', label: 'Orthogonal', desc: 'Right-angles, no avoidance',     args: { padding: 10 } },
  { name: 'metro',      label: 'Metro',      desc: '45° diagonals at bends',         args: { padding: 10 } },
  { name: 'normal',     label: 'Normal',     desc: 'Direct lines, no bends',         args: {} },
  { name: 'oneSide',    label: 'One Side',   desc: 'All links exit one side',        args: { padding: 20 } },
];

function applyRouter(name) {
  state.router = name;
  const r = ROUTERS.find(r => r.name === name);
  const def = Object.keys(r.args).length ? { name, args: r.args } : { name };
  graph.getLinks().forEach(link => link.router(def));
}
// Call applyRouter(state.router) at the end of initGraph() so all links get
// the persisted router on initial render and after each animateToState() rebuild.
```

### Path highlighting (undirected BFS)

`selectedElementId` — click-pinned anchor. Pre-select a meaningful root node on load.

```js
let selectedElementId = null;
const DEFAULT_ANCHOR_ID = 'root-node-id'; // set to the most meaningful root/central node

// Use undirected BFS — architecture/data diagrams have inconsistent edge directions.
// Directed BFS misses valid paths through leaf nodes (nodes that only appear as edge sources).
function findPathIds(fromEl, toEl) {
  if (!toEl || fromEl.id === toEl.id) return new Set([fromEl.id]);
  const prev = new Map(), queue = [fromEl], visited = new Set([fromEl.id]);
  let found = false;
  outer: while (queue.length > 0) {
    const cur = queue.shift();
    for (const nb of graph.getNeighbors(cur)) {
      if (visited.has(nb.id)) continue;
      visited.add(nb.id);
      prev.set(nb.id, cur);
      if (nb.id === toEl.id) { found = true; break outer; }
      queue.push(nb);
    }
  }
  if (!found) return null;
  const pathSet = new Set();
  let node = toEl;
  while (node) { pathSet.add(node.id); node = prev.get(node.id); }
  return pathSet;
}

// Enter state is instant: stopTransitions() + el.attr() synchronously.
// Exit is animated via clearPathHighlights().
function highlightPath(element) {
  const visibleIds = getVisibleIds();
  const nodeData = element.get('nodeData');
  if (!nodeData || !visibleIds.has(nodeData.id)) return;
  const selectedEl = graph.getCell(selectedElementId || DEFAULT_ANCHOR_ID);
  const pathSet = findPathIds(element, selectedEl);

  graph.getElements().forEach(el => {
    const nd = el.get('nodeData');
    if (!nd || !visibleIds.has(nd.id)) return;
    const elLay = LAYERS[nd.layer] || {};
    const onPath = pathSet && pathSet.has(el.id);
    const isHovered = el.id === element.id;
    const isAnchor = selectedEl && el.id === selectedEl.id && el.id !== element.id;

    el.stopTransitions('attrs/body/strokeWidth');
    el.attr('root/opacity', onPath ? 1 : 0.55);
    el.attr('body/stroke', onPath ? elLay.color : elLay.border);
    el.attr('body/strokeWidth', isHovered ? 4 : isAnchor ? 3 : onPath ? 2 : 1.5);
  });

  const layerIds = getLayerIds();
  graph.getLinks().forEach(link => {
    const c = link.get('connData'); if (!c) return;
    link.stopTransitions('attrs/line/opacity');
    const layerVisible = layerIds.has(c.from) && layerIds.has(c.to) && state.conns[c.type];
    const onPath = layerVisible && pathSet && pathSet.has(c.from) && pathSet.has(c.to);
    link.attr('line/opacity', !layerVisible ? 0 : onPath ? 0.75 : 0.12);
  });
}

function clearPathHighlights(restoreLinks = true) {
  const layerIds = getLayerIds();
  const T = { duration: 120, timingFunction: easeHighlight, valueFunction: joint.util.interpolate.number };
  graph.getElements().forEach(el => {
    const nd = el.get('nodeData'); if (!nd || !layerIds.has(nd.id)) return;
    const elLay = LAYERS[nd.layer] || {};
    el.attr('root/opacity', 1);
    el.attr('body/stroke', elLay.border);
    el.transition('attrs/body/strokeWidth', 1.5, T);
  });
  if (restoreLinks) {
    const visibleIds = getVisibleIds();
    graph.getLinks().forEach(link => {
      const c = link.get('connData');
      const show = c && state.conns[c.type] && visibleIds.has(c.from) && visibleIds.has(c.to);
      link.transition('attrs/line/opacity', show ? 0.55 : 0, {
        duration: 220, timingFunction: easeHighlight, valueFunction: joint.util.interpolate.number,
      });
    });
  }
}
```

Paper events:
```js
paper.on('element:mouseenter', (v) => {
  paper.el.classList.remove('hl-exit');
  paper.el.style.cursor = 'pointer';
  highlightPath(v.model);
});
paper.on('element:mouseleave', () => {
  paper.el.classList.add('hl-exit');
  paper.el.style.cursor = '';
  const anchorEl = graph.getCell(selectedElementId || DEFAULT_ANCHOR_ID);
  anchorEl ? highlightPath(anchorEl) : clearPathHighlights();
});
paper.on('element:pointerclick', (v) => {
  const el = v.model;
  selectedElementId = selectedElementId === el.id ? null : el.id;
  highlightPath(el);
  openModal(el.get('nodeData'));
});
paper.on('blank:pointerclick', () => {
  paper.el.classList.add('hl-exit');
  selectedElementId = null;
  const defaultEl = graph.getCell(DEFAULT_ANCHOR_ID);
  defaultEl ? highlightPath(defaultEl) : clearPathHighlights();
});
```

### Pan and zoom

```js
let _panOrigin = null;
paper.on('blank:pointerdown', evt => {
  _panOrigin = { cx: evt.clientX, cy: evt.clientY, tx: paper.translate().tx, ty: paper.translate().ty };
});
document.addEventListener('pointermove', e => {
  if (!_panOrigin) return;
  paper.translate(_panOrigin.tx + e.clientX - _panOrigin.cx, _panOrigin.ty + e.clientY - _panOrigin.cy);
});
document.addEventListener('pointerup', () => { _panOrigin = null; });

wrap.addEventListener('wheel', e => {
  e.preventDefault();
  const s = paper.scale().sx, f = e.deltaY < 0 ? 1.12 : 1/1.12;
  const ns = Math.max(0.2, Math.min(2, s * f));
  const r = paperContainer.getBoundingClientRect();
  const ox = e.clientX - r.left, oy = e.clientY - r.top;
  const t = paper.translate();
  paper.translate(ox - (ox - t.tx) * ns/s, oy - (oy - t.ty) * ns/s);
  paper.scale(ns, ns);
}, { passive: false });
```

### On load

```js
window.addEventListener('load', () => {
  paper.setDimensions(wrap.clientWidth, wrap.clientHeight);
  initGraph();
  renderControls();
  paper.scale(state.zoom, state.zoom);
  const totalMs = animateToState();
  setTimeout(() => {
    const rootEl = graph.getCell(DEFAULT_ANCHOR_ID);
    if (rootEl) highlightPath(rootEl);
  }, totalMs + 60);
});
```

### Common pitfalls

| Problem | Fix |
|---|---|
| `dia.Graph: cell type must be a string` | Never use `new joint.dia.Link()` — always use a defined concrete type. |
| `dia.LinkView: markup required` | Extend `joint.shapes.standard.Link.define()`, NOT `joint.dia.Link.define()` — standard inherits the line/wrapper markup. |
| Straight router not working | `'straight'` is a **connector** name. For direct-line routing use `{ name: 'normal' }`. |
| Hidden nodes distorting fit | Compute `contentArea` from NODES data positions, not live SVG bbox. |
| Search distorts layout | Never call `animateToState()` from `handleSearch` — it resets node positions. Use `applySearchFilter()` instead. |
| Directed BFS misses leaf nodes | Use `graph.getNeighbors(cur)` with no options (undirected). |
| Animations break on rapid toggle | Do NOT call `el.stopTransitions()` before `el.transition()` — it cancels the queued animation. |
| `</script>` in snippet strings | The HTML parser closes the `<script>` block on the raw token — escape every `</script>` as `<\/script>` inside snippet strings. |
| `joint.util.timing.cubicBezier` is not a function | This API does not exist in JointJS 3.x. Use the hand-rolled `easeHighlight` already defined in this template, or inline `t => t < 0.5 ? 4*t*t*t : 1 - Math.pow(-2*t+2,3)/2` for ease-in-out cubic. |
| Data Explorer: `header` rect bleeds past rounded corners | Accept the slight bleed — it looks fine at this scale, or clip with `overflow: hidden` on the SVG group if needed. |
| Data Explorer: link labels overlap nodes | Use `position: { distance: 0.45 }` (slightly off-center toward source). |
| Data Explorer: field search not working | Ensure `getVisibleIds()` checks `n.fields.some(f => f.name.includes(q))` in addition to the label. |

---

## Code Map Spec

Use this spec for: "create a code map for X", "visualize the architecture of Y", "show me how Z is structured".

### Node type

```js
const ModuleNode = joint.dia.Element.define('codemap.ModuleNode', {
  markup: [
    { tagName: 'rect',   selector: 'body' },
    { tagName: 'text',   selector: 'label' },
    { tagName: 'text',   selector: 'sublabel' },
    { tagName: 'circle', selector: 'noteDot' },
  ],
  size: { width: 160, height: 48 },
  attrs: {
    body:     { width: 'calc(w)', height: 'calc(h)', rx: 8, ry: 8 },
    label:    { x: 'calc(w/2)', y: 18, textAnchor: 'middle', fontSize: 12, fontWeight: 600 },
    sublabel: { x: 'calc(w/2)', y: 34, textAnchor: 'middle', fontSize: 9, opacity: 0.55 },
    noteDot:  { r: 4, cx: 'calc(w - 8)', cy: 8, opacity: 0 },
  }
});
```

Instantiate with:
```js
const el = new ModuleNode({
  id: n.id,
  position: stagePos(n),
  attrs: {
    root:     { opacity: 0 },
    body:     { fill: lay.fill, stroke: lay.border, strokeWidth: 1.5 },
    label:    { text: n.label, fill: lay.color },
    sublabel: { text: n.sub.split('/').pop(), fill: '#8b949e' },
    noteDot:  { fill: lay.color, opacity: 0 },
  },
});
el.set('nodeData', n);
```

### Node data schema

```js
const NODES = [
  {
    id: 'module-id',         // unique, used as JointJS element id
    label: 'Display Name',   // shown in the node
    sub: 'path/to/file.js',  // file path, last segment shown as subtitle
    layer: 'layerKey',       // must match a key in LAYERS
    x: 200, y: 100,          // target position
    desc: 'What this module does...',
    keys: ['method()', 'Class', 'option'],
    snippet: `code example — escape <\/script> if needed`,
    deps: ['other-id'],
  },
];
```

### Connection data schema

```js
const CONNECTIONS = [
  { from: 'source-id', to: 'target-id', type: 'uses' },
  // types: uses | extends | registers | depends
];
```

### Layers (6–8, adapt to target library)

```js
const LAYERS = {
  core:       { label: 'Core',         color: '#818cf8', fill: '#1e2040', border: '#4f56a8' },
  shapes:     { label: 'Shapes',       color: '#34d399', fill: '#122820', border: '#166534' },
  routing:    { label: 'Routing',      color: '#fb923c', fill: '#271a10', border: '#9a3412' },
  connectors: { label: 'Connectors',   color: '#f472b6', fill: '#271025', border: '#9d174d' },
  highlight:  { label: 'Highlighters', color: '#facc15', fill: '#272010', border: '#a16207' },
  tools:      { label: 'Tools',        color: '#38bdf8', fill: '#0c1f2e', border: '#0369a1' },
  geometry:   { label: 'Geometry',     color: '#a78bfa', fill: '#1c1530', border: '#6d28d9' },
  plugins:    { label: 'Plugins',      color: '#94a3b8', fill: '#18202e', border: '#475569' },
};
```

### Connection types

```js
const CONN_TYPES = {
  uses:      { label: 'Uses / Calls',   color: '#5b7cf6', dash: null },
  extends:   { label: 'Extends',        color: '#34d399', dash: '6 3' },
  registers: { label: 'Registers with', color: '#fb923c', dash: '4 4' },
  depends:   { label: 'Depends on',     color: '#94a3b8', dash: '3 5' },
};
```

### Presets (4 named)

```js
const PRESETS = {
  full:      { layers: /* all true */, conns: /* all true */ },
  core:      { layers: { core:true, /* rest false */ }, conns: { uses:true, extends:true, /* rest false */ } },
  rendering: { layers: { core:true, shapes:true, routing:true, connectors:true, /* rest false */ }, conns: { uses:true, extends:true, registers:true, depends:false } },
  geometry:  { layers: { geometry:true, /* rest false */ }, conns: { depends:true, /* rest false */ } },
};
```

### Node detail modal

Click a node → modal showing:
- Name (large heading) + file path (monospace subtitle)
- Description paragraph (dark background box)
- Key APIs as inline monospace badge chips
- Code snippet (`<pre>` green monospace on dark)
- Dependencies list
- Note textarea → save adds to sidebar notes list + marks node with noteDot

### Prompt output

```js
function updatePrompt() {
  const visLayers = Object.entries(state.layers).filter(([,v]) => v).map(([k]) => LAYERS[k]?.label || k);
  let text = `Code map — showing layers: ${visLayers.join(', ')}.`;
  if (state.comments.length > 0) {
    text += '\n\nNotes:\n' + state.comments.map(c => `**${c.label}** (${c.sub}): ${c.note}`).join('\n');
  }
  document.getElementById('prompt-output').value = text;
}
```

### Pre-populate with real data

Use 15–25 nodes with:
- Real module names and actual file paths
- Key classes/functions from the public API
- Accurate code snippets showing typical usage — **escape any `</script>` as `<\/script>`**
- Connections reflecting actual import/dependency relationships
- Layer groupings matching the library's conceptual architecture

---

## Data Explorer Spec

Use this spec for: "visualize my database schema", "explore this Postgres schema", "show entity relationships", "create a data map for our API models".

### Additional CSS (field table)

```css
.field-table {
  width: 100%; border-collapse: collapse; font-size: 12px;
  font-family: 'SF Mono', Consolas, monospace;
}
.field-table th {
  text-align: left; padding: 5px 8px; font-size: 10px; font-weight: 600;
  color: #8b949e; text-transform: uppercase; letter-spacing: 0.08em;
  border-bottom: 1px solid #30363d;
}
.field-table td { padding: 5px 8px; border-bottom: 1px solid #21262d; color: #c9d1d9; }
.field-table tr:last-child td { border-bottom: none; }
.field-badge {
  display: inline-block; padding: 1px 5px; border-radius: 4px;
  font-size: 9px; font-weight: 700; letter-spacing: 0.05em; margin-right: 3px;
}
.badge-pk { background: #2d3f1e; color: #7ee787; border: 1px solid #3fb950; }
.badge-fk { background: #1c2d40; color: #79c0ff; border: 1px solid #388bfd; }
.badge-uq { background: #2e1d40; color: #d2a8ff; border: 1px solid #8957e5; }
.badge-nn { background: #302010; color: #ffa657; border: 1px solid #d29922; }
.field-type-tag { color: #8b949e; font-size: 11px; }
```

### Node type

```js
const EntityNode = joint.dia.Element.define('datamap.EntityNode', {
  markup: [
    { tagName: 'rect',   selector: 'body' },
    { tagName: 'rect',   selector: 'header' },
    { tagName: 'text',   selector: 'label' },
    { tagName: 'text',   selector: 'sublabel' },
    { tagName: 'circle', selector: 'noteDot' },
  ],
  size: { width: 170, height: 50 },
  attrs: {
    body:     { width: 'calc(w)', height: 'calc(h)', rx: 8, ry: 8 },
    header:   { width: 'calc(w)', height: 22, rx: 8, ry: 8 },
    label:    { x: 'calc(w/2)', y: 15, textAnchor: 'middle', fontSize: 12, fontWeight: 700 },
    sublabel: { x: 'calc(w/2)', y: 37, textAnchor: 'middle', fontSize: 9, opacity: 0.65 },
    noteDot:  { r: 4, cx: 'calc(w - 8)', cy: 8, opacity: 0 },
  }
});
```

The `header` rect creates the two-tone look. Set `headerFill` in each layer as a low-alpha version of `color`.

```js
function buildSublabel(n) {
  const pk = n.fields.find(f => f.pk);
  return [pk ? `PK: ${pk.name}` : '', `${n.fields.length} fields`].filter(Boolean).join(' · ');
}

const el = new EntityNode({
  id: n.id,
  position: stagePos(n),
  attrs: {
    root:     { opacity: 0 },
    body:     { fill: lay.fill, stroke: lay.border, strokeWidth: 1.5 },
    header:   { fill: lay.headerFill, stroke: 'none' },
    label:    { text: n.label, fill: lay.color },
    sublabel: { text: buildSublabel(n), fill: '#8b949e' },
    noteDot:  { fill: lay.color, opacity: 0 },
  },
});
el.set('nodeData', n);
```

### Node data schema

```js
const NODES = [
  {
    id: 'users',
    label: 'users',
    sub: 'auth',                   // schema / domain — used in sublabel context and modal
    layer: 'auth',
    x: 100, y: 120,
    desc: 'Core user accounts. Stores credentials, profile data, and account status.',
    fields: [
      { name: 'id',         type: 'uuid',        pk: true,  nullable: false },
      { name: 'email',      type: 'varchar(255)', nullable: false, unique: true },
      { name: 'name',       type: 'varchar(100)', nullable: true },
      { name: 'role',       type: 'user_role',    nullable: false },
      { name: 'created_at', type: 'timestamptz',  nullable: false },
    ],
    record_hint: '~2M rows',
    snippet: `-- Find active users by role\nSELECT id, email, name\nFROM users\nWHERE role = $1\n  AND deleted_at IS NULL;`,
    deps: ['sessions', 'profiles'],
  },
];
```

### Relation data schema

```js
const CONNECTIONS = [
  { from: 'orders', to: 'users',    type: 'foreignKey', label: 'user_id'   },
  { from: 'orders', to: 'products', type: 'references', label: 'via items' },
  { from: 'admins', to: 'users',    type: 'extends',    label: ''          },
  // types: foreignKey | references | extends | contains | depends
];
```

Render the label as a small annotation on the link — use `position: { distance: 0.45 }`:
```js
labels: c.label ? [{
  attrs: {
    text: { text: c.label, fontSize: 9, fill: '#8b949e', fontFamily: "'SF Mono', Consolas, monospace" },
    rect: { fill: '#0d1117', stroke: 'none', rx: 3 },
  },
  position: { distance: 0.45 },
}] : [],
```

### Layers (4–6, adapt to target domain)

```js
const LAYERS = {
  auth:      { label: 'Auth',      color: '#818cf8', fill: '#1a1e3a', headerFill: 'rgba(129,140,248,0.12)', border: '#4f56a8' },
  product:   { label: 'Product',   color: '#34d399', fill: '#0f2219', headerFill: 'rgba(52,211,153,0.12)',  border: '#166534' },
  order:     { label: 'Order',     color: '#fb923c', fill: '#261910', headerFill: 'rgba(251,146,60,0.12)',  border: '#9a3412' },
  payment:   { label: 'Payment',   color: '#f472b6', fill: '#261020', headerFill: 'rgba(244,114,182,0.12)', border: '#9d174d' },
  analytics: { label: 'Analytics', color: '#facc15', fill: '#261f0e', headerFill: 'rgba(250,204,21,0.12)',  border: '#a16207' },
  config:    { label: 'Config',    color: '#94a3b8', fill: '#16202e', headerFill: 'rgba(148,163,184,0.12)', border: '#475569' },
};
```

### Relationship types

```js
const CONN_TYPES = {
  foreignKey: { label: 'Foreign Key', color: '#5b7cf6', dash: null  },
  references: { label: 'References',  color: '#34d399', dash: '6 3' },
  extends:    { label: 'Extends',     color: '#fb923c', dash: '4 4' },
  contains:   { label: 'Contains',    color: '#f472b6', dash: '3 6' },
  depends:    { label: 'Depends on',  color: '#94a3b8', dash: '2 5' },
};
```

### Presets (4 named)

```js
const PRESETS = {
  full:         { layers: /* all true */, conns: /* all true */ },
  core:         { layers: { auth:true, product:true, order:true, /* rest false */ }, conns: { foreignKey:true, references:true, extends:false, contains:false, depends:false } },
  auth:         { layers: { auth:true, /* rest false */ }, conns: { foreignKey:true, extends:true, /* rest false */ } },
  transactions: { layers: { order:true, payment:true, product:true, /* rest false */ }, conns: { foreignKey:true, references:true, contains:true, depends:false, extends:false } },
};
```

### Entity detail modal

Click an entity → modal showing:
- **Entity name** (large heading) + schema path (monospace subtitle)
- **Record hint** badge (e.g. "~2M rows") in the header if `record_hint` is set
- **Description** paragraph (dark background box)
- **Fields table** — Field | Type | Constraints columns with PK/FK/UQ/NN badges
- **Sample query** (`<pre>` green monospace on dark)
- **Related entities** list (from `deps` array)
- **Note textarea** → save adds to sidebar notes list + marks node with noteDot

```js
function openModal(nd) {
  // ... header/desc setup ...

  // Record hint
  const hintEl = document.getElementById('modal-hint');
  hintEl.textContent = nd.record_hint || '';
  hintEl.style.display = nd.record_hint ? '' : 'none';

  // Field table
  const tbody = document.getElementById('field-table-body');
  tbody.innerHTML = '';
  nd.fields.forEach(f => {
    const badges = [];
    if (f.pk)     badges.push('<span class="field-badge badge-pk">PK</span>');
    if (f.fk)     badges.push('<span class="field-badge badge-fk">FK</span>');
    if (f.unique) badges.push('<span class="field-badge badge-uq">UQ</span>');
    if (!f.nullable && !f.pk) badges.push('<span class="field-badge badge-nn">NN</span>');
    const tr = document.createElement('tr');
    tr.innerHTML = `<td>${badges.join('')}${f.name}</td><td class="field-type-tag">${f.type}</td><td></td>`;
    tbody.appendChild(tr);
  });

  document.getElementById('modal-snippet').textContent = nd.snippet || '';
}
```

Modal HTML (inside `.modal-body`):
```html
<div class="modal-desc" id="modal-desc"></div>
<div>
  <div class="modal-section-title">Fields</div>
  <table class="field-table">
    <thead><tr><th>Field</th><th>Type</th><th>Constraints</th></tr></thead>
    <tbody id="field-table-body"></tbody>
  </table>
</div>
<div>
  <div class="modal-section-title">Sample Query</div>
  <pre class="code-snippet" id="modal-snippet"></pre>
</div>
<div id="modal-deps-section">
  <div class="modal-section-title">Related Entities</div>
  <div class="deps-list" id="modal-deps"></div>
</div>
<div>
  <div class="modal-section-title">Note</div>
  <textarea class="note-area" id="note-textarea"></textarea>
  <button class="save-note-btn" id="save-note-btn">Save Note</button>
</div>
```

### Prompt output

```js
function updatePrompt() {
  const visLayers = Object.entries(state.layers).filter(([,v]) => v).map(([k]) => LAYERS[k]?.label || k);
  const visIds = getLayerIds();
  const entitySummary = NODES.filter(n => visIds.has(n.id))
    .map(n => `${n.label} (${n.fields.length} fields)`).join(', ');
  let text = `Data model — showing domains: ${visLayers.join(', ')}.\nEntities: ${entitySummary}.`;
  if (state.comments.length > 0) {
    text += '\n\nNotes:\n' + state.comments.map(c => `**${c.label}**: ${c.note}`).join('\n');
  }
  document.getElementById('prompt-output').value = text;
}
```

### Pre-populate with real data

Use 12–20 entities with:
- Real table/collection names matching the domain (`orders`, `order_items`, `products`, `users`)
- Real field names with accurate SQL types (`uuid`, `varchar(255)`, `timestamptz`, `numeric(10,2)`, `jsonb`)
- Correct constraint flags: PK on primary keys, FK on foreign key columns, UNIQUE on natural keys, NOT NULL where appropriate
- Connections reflecting actual FK relationships, with the FK column name in `label`
- Layers matching natural domain groupings (auth, product, order, payment, analytics, config)
- Sample queries showing typical access patterns — **escape any `</script>` as `<\/script>`**
- `record_hint` for entities with meaningful size differences

---

## Powered by JointJS badge

**Always include** this badge in every generated HTML file — no exceptions.

**CSS** — add inside `<style>`:
```css
#powered-by {
  position: absolute; top: 12px; right: 12px; z-index: 10;
  display: flex; align-items: center; gap: 7px;
  background: #161b22cc; border: 1px solid #30363d; border-radius: 8px;
  padding: 10px 20px 10px 18px; text-decoration: none;
  color: #8b949e; font-size: 11px; font-weight: 500;
  backdrop-filter: blur(6px);
  transition: border-color 160ms, color 160ms, background 160ms;
}
#powered-by:hover { border-color: #58a6ff; color: #c9d1d9; background: #1c2d40cc; }
#powered-by img { height: 18px; width: auto; opacity: 0.85; transition: opacity 160ms; }
#powered-by:hover img { opacity: 1; }
#powered-by .pb-sep { width: 1px; margin-inline: 5px; height: 14px; background: #30363d; }
```

**HTML** — place as first child inside `#canvas-wrap`:
```html
<a id="powered-by" href="https://jointjs.com?utm_source=jointjs-claude-playground&utm_medium=graph-explorer&utm_campaign=claude-code" target="_blank" rel="noopener">
  <span style="color:#6e7681;font-size:10px;letter-spacing:0.04em;text-transform:uppercase">Powered by</span>
  <div class="pb-sep"></div>
  <img src="https://cdn.prod.website-files.com/63061d4ee85b5a18644f221c/633045c1d726c7116dcbe582_JJS_logo.svg" alt="JointJS" />
</a>
```

**Position rules** — default is `top: 12px; right: 12px`. Adjust if something overlaps:

| Conflict | Fix |
|---|---|
| Search bar (`#search-bar`) extends past the horizontal midpoint | `top: 52px; right: 12px` — drop below the search input |
| Zoom controls (`#zoom-controls`) are repositioned to the top-right | `top: 12px; right: 120px` — shift left of the zoom cluster |
| A legend, tooltip, or note panel floats in the top-right | `bottom: 52px; right: 12px` — sits above the prompt bar |
| Both top-right and bottom-right are occupied | `top: 12px; left: 272px` — just right of the 260px sidebar |

Always verify visually: the badge must not obscure any interactive control (buttons, inputs, pills). When in doubt, prefer moving it lower rather than to a different corner.
