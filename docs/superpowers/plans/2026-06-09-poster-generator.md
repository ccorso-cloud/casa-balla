# Poster Generator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-page poster generator that composes SVG tiles (tasselli + speciali) into a Paper.js grid.

**Architecture:** Single `index.html` with inline CSS and JS. Paper.js loads via CDN. All SVG files are loaded at startup via `fetch()`, imported as `paper.Symbol`, then placed on the grid by the generation algorithm.

**Tech Stack:** Paper.js (CDN), vanilla JS, no build tools.

---

## File Structure

| File | Responsibility |
|------|---------------|
| `index.html` | Entire app — UI, Paper.js rendering, placement algorithm |

No other files. Everything self-contained in one HTML page.

---

### Task 1: Create HTML shell with UI controls

**Files:**
- Create: `index.html`

- [ ] **Step 1: Write the HTML skeleton**

```html
<!DOCTYPE html>
<html lang="it">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Generatore Poster — Casa Balla</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/paper.js/0.12.17/paper-full.min.js"></script>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #f0f0f0; font-family: 'Helvetica Neue', Arial, sans-serif; overflow: hidden; height: 100vh; }
  #toolbar {
    position: fixed; top: 20px; left: 50%; transform: translateX(-50%);
    background: rgba(255,255,255,0.95); border-radius: 12px;
    padding: 16px 24px; box-shadow: 0 4px 20px rgba(0,0,0,0.15);
    display: flex; gap: 16px; align-items: center; z-index: 10;
    backdrop-filter: blur(8px);
  }
  #toolbar label { font-size: 14px; font-weight: 600; color: #333; }
  #toolbar input[type="number"] {
    width: 60px; padding: 6px 8px; border: 2px solid #ddd; border-radius: 6px;
    font-size: 14px; text-align: center;
  }
  #toolbar input[type="number"]:focus { border-color: #009fe3; outline: none; }
  #generateBtn {
    background: #009fe3; color: #fff; border: none; border-radius: 6px;
    padding: 8px 20px; font-size: 14px; font-weight: 600; cursor: pointer;
    transition: background 0.2s;
  }
  #generateBtn:hover { background: #007bb5; }
  #exportBtn {
    background: #733b8f; color: #fff; border: none; border-radius: 6px;
    padding: 8px 20px; font-size: 14px; font-weight: 600; cursor: pointer;
    transition: background 0.2s;
  }
  #exportBtn:hover { background: #5a2e72; }
  #errorMsg {
    position: fixed; bottom: 30px; left: 50%; transform: translateX(-50%);
    background: #e74c3c; color: #fff; padding: 12px 24px; border-radius: 8px;
    display: none; z-index: 10; font-size: 14px;
  }
  #canvasContainer {
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  }
  canvas { display: block; }
</style>
</head>
<body>
  <div id="toolbar">
    <label>Colonne</label>
    <input type="number" id="colsInput" value="6" min="3" max="20">
    <label>Righe</label>
    <input type="number" id="rowsInput" value="8" min="3" max="20">
    <button id="generateBtn">Genera</button>
    <button id="exportBtn">Esporta PNG</button>
  </div>
  <div id="errorMsg"></div>
  <div id="canvasContainer"></div>
  <script>
    // JS will go here in subsequent tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser to verify UI renders**

Open `index.html` in browser. Confirm toolbar appears with inputs (6, 8), buttons "Genera" and "Esporta PNG", and a blank container below.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HTML shell with toolbar UI"
```

---

### Task 2: Load all SVG files as Paper.js Symbols

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add SVG file list and loader function**

Add inside the `<script>` tag (replace placeholder comment):

```js
const TILES = [
  'tasselli/Risorsa 8.svg', 'tasselli/Risorsa 9.svg', 'tasselli/Risorsa 10.svg',
  'tasselli/Risorsa 11.svg', 'tasselli/Risorsa 12.svg', 'tasselli/Risorsa 13.svg',
  'tasselli/Risorsa 14.svg', 'tasselli/Risorsa 15.svg', 'tasselli/Risorsa 16.svg',
  'tasselli/Risorsa 17.svg', 'tasselli/Risorsa 18.svg', 'tasselli/Risorsa 19.svg',
  'tasselli/Risorsa 20.svg', 'tasselli/Risorsa 21.svg', 'tasselli/Risorsa 22.svg',
  'tasselli/Risorsa 23.svg', 'tasselli/Risorsa 24.svg', 'tasselli/Risorsa 25.svg',
  'tasselli/Risorsa 26.svg', 'tasselli/Risorsa 27.svg', 'tasselli/Risorsa 28.svg',
  'tasselli/Risorsa 29.svg'
];

const SPECIALS = [
  { file: 'speciali/titolo-2x3.svg',  w: 2, h: 3 },
  { file: 'speciali/info-2x2.svg',    w: 2, h: 2 },
  { file: 'speciali/contatti-2x1.svg', w: 2, h: 1 },
  { file: 'speciali/maxxi-1x1.svg',   w: 1, h: 1 },
  { file: 'speciali/sovrintendenza-1x1.svg', w: 1, h: 1 }
];

const UNIT = 168.38;

let tileSymbols = [];
let specialSymbols = [];

async function loadSVG(url) {
  const resp = await fetch(url);
  const svgText = await resp.text();
  return new Promise(resolve => {
    const svg = document.createElement('svg');
    svg.innerHTML = svgText;
    svg.setAttribute('style', 'display:none');
    document.body.appendChild(svg);
    paper.project.importSVG(svg, {
      onLoad: function(item) {
        item.remove();
        resolve(item);
      }
    });
  });
}

async function loadAllSVGs() {
  const tilePromises = TILES.map(url => loadSVG(url));
  const specialPromises = SPECIALS.map(s => loadSVG(s.file));
  const tiles = await Promise.all(tilePromises);
  const specials = await Promise.all(specialPromises);
  tileSymbols = tiles.map(t => new paper.Symbol(t));
  specialSymbols = specials.map((s, i) => ({
    symbol: new paper.Symbol(s),
    w: SPECIALS[i].w,
    h: SPECIALS[i].h
  }));
}
```

- [ ] **Step 2: Initialize Paper.js on page load**

After the loader, add:

```js
paper.setup('myCanvas');

async function init() {
  await loadAllSVGs();
  console.log('Loaded', tileSymbols.length, 'tile symbols');
}
init();
```

And update the canvas container in the HTML:
```html
<div id="canvasContainer">
  <canvas id="myCanvas" resize></canvas>
</div>
```

Add `resize` to the canvas tag and `id="myCanvas"`.

- [ ] **Step 3: Test in browser**

Open with browser dev tools console. Confirm "Loaded 22 tile symbols" and no 404 errors for SVG files in Network tab.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: load SVG symbols via Paper.js"
```

---

### Task 3: Implement grid data model and special placement

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add grid model and special placement algorithm**

Add after the symbol arrays:

```js
let grid = [];
let gridCols = 0;
let gridRows = 0;

function initGrid(cols, rows) {
  gridCols = cols;
  gridRows = rows;
  grid = Array.from({ length: rows }, () => Array(cols).fill(null));
}

function canPlace(r, c, w, h) {
  if (r + h > gridRows || c + w > gridCols) return false;
  for (let dr = 0; dr < h; dr++)
    for (let dc = 0; dc < w; dc++)
      if (grid[r + dr][c + dc] !== null) return false;
  return true;
}

function placeSpecial(r, c, idx) {
  const special = SPECIALS[idx];
  for (let dr = 0; dr < special.h; dr++)
    for (let dc = 0; dc < special.w; dc++)
      grid[r + dr][c + dc] = { type: 'special', index: idx };
  return { row: r, col: c, w: special.w, h: special.h, special: true, index: idx };
}

function placeSpecials() {
  const placed = [];
  const indexed = SPECIALS.map((s, i) => ({ ...s, origIndex: i }));
  const order = indexed.sort((a, b) => (b.w * b.h) - (a.w * a.h));
  for (const special of order) {
    let attempts = 0;
    let success = false;
    while (attempts < 100 && !success) {
      const r = Math.floor(Math.random() * (gridRows - special.h + 1));
      const c = Math.floor(Math.random() * (gridCols - special.w + 1));
      if (canPlace(r, c, special.w, special.h)) {
        placed.push(placeSpecial(r, c, special.origIndex));
        success = true;
      }
      attempts++;
    }
    if (!success) {
      throw new Error('Griglia troppo piccola per posizionare tutti gli speciali. Aumenta righe o colonne.');
    }
  }
  return placed;
}
```

- [ ] **Step 2: Add a temporary test call at the end of `init()`:**

```js
initGrid(6, 8);
try {
  const placed = placeSpecials();
  console.log('Placed', placed.length, 'specials');
  console.log('Grid:', grid.map(r => r.map(c => c ? c.type : '·').join(' ')).join('\n'));
} catch (e) {
  console.error(e.message);
}
```

- [ ] **Step 3: Test in browser**

Open browser console. Verify grid shows 5 specials placed, no overlaps, and all cells are either occupied (type 'special') or empty.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add grid model and special placement algorithm"
```

---

### Task 4: Implement normal tile placement

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add tile placement with random sizes and pool**

Add after `placeSpecials()`:

```js
let tilePool = [];

function shuffleArray(arr) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

function refillPool() {
  tilePool = shuffleArray([...Array(tileSymbols.length).keys()]);
}

function nextTileIndex() {
  if (tilePool.length === 0) refillPool();
  return tilePool.pop();
}

function placeTiles() {
  refillPool();
  const placed = [];
  for (let r = 0; r < gridRows; r++) {
    for (let c = 0; c < gridCols; c++) {
      if (grid[r][c] !== null) continue;
      const sizes = shuffleArray([3, 2, 1]);
      for (const size of sizes) {
        if (canPlace(r, c, size, size)) {
          const idx = nextTileIndex();
          for (let dr = 0; dr < size; dr++)
            for (let dc = 0; dc < size; dc++)
              grid[r + dr][c + dc] = { type: 'tile', index: idx, size: size };
          placed.push({ row: r, col: c, size, tileIndex: idx });
          break;
        }
      }
    }
  }
  return placed;
}
```

- [ ] **Step 2: Update init() test:**

```js
initGrid(6, 8);
try {
  const specials = placeSpecials();
  const tiles = placeTiles();
  console.log('Placed', specials.length, 'specials,', tiles.length, 'tile groups');
  console.log('Grid:', grid.map(r => r.map(c => c ? c.type + (c.size || '') : '·').join(' ')).join('\n'));
} catch (e) {
  console.error(e.message);
}
```

- [ ] **Step 3: Test in browser**

Verify console log shows "Placed 5 specials" and many tile groups. Grid output should show all cells filled with `tile1`, `tile2`, `tile3`, etc.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add tile placement with random sizes and pool"
```

---

### Task 5: Render the grid with Paper.js

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add render function**

Add after `placeTiles()`:

```js
function renderPoster(placedSpecials, placedTiles) {
  paper.project.activeLayer.removeChildren();
  const totalW = gridCols * UNIT;
  const totalH = gridRows * UNIT;

  // white background
  const bg = new paper.Shape.Rectangle({
    point: [0, 0],
    size: [totalW, totalH],
    fillColor: '#ffffff'
  });

  const offsetX = 0;
  const offsetY = 0;

  // Specials
  for (const s of placedSpecials) {
    const inst = specialSymbols[s.index].symbol.place(new paper.Point(
      s.col * UNIT + offsetX,
      s.row * UNIT + offsetY
    ));
    inst.scale(UNIT / 168.38);
  }

  // Tiles
  for (const t of placedTiles) {
    const inst = tileSymbols[t.tileIndex].place(new paper.Point(
      t.col * UNIT + offsetX,
      t.row * UNIT + offsetY
    ));
    inst.scale((UNIT * t.size) / 168.38);
  }

  paper.view.draw();

  // Center in viewport
  const canvas = paper.view.element;
  const scaleX = canvas.width / (totalW + 40);
  const scaleY = canvas.height / (totalH + 40);
  const scale = Math.min(scaleX, scaleY, 1);
  paper.view.zoom = scale;
  paper.view.center = new paper.Point(totalW / 2, totalH / 2);
}
```

- [ ] **Step 2: Update init() to render**

```js
async function init() {
  await loadAllSVGs();
  const cols = parseInt(document.getElementById('colsInput').value);
  const rows = parseInt(document.getElementById('rowsInput').value);
  initGrid(cols, rows);
  try {
    const specials = placeSpecials();
    const tiles = placeTiles();
    renderPoster(specials, tiles);
  } catch (e) {
    showError(e.message);
  }
}
```

- [ ] **Step 3: Add showError helper**

```js
function showError(msg) {
  const el = document.getElementById('errorMsg');
  el.textContent = msg;
  el.style.display = 'block';
  setTimeout(() => el.style.display = 'none', 5000);
}
```

- [ ] **Step 4: Test in browser**

Open page. Should see a 6×8 grid of rendered tiles with specials mixed in. All cells filled.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: render poster with Paper.js"
```

---

### Task 6: Wire "Genera" button and "Esporta PNG" button

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add event listeners**

Find `document.addEventListener('DOMContentLoaded', ...)` or equivalent, or simply add at the bottom of the script:

```js
document.addEventListener('DOMContentLoaded', () => {
  init();
});

document.getElementById('generateBtn').addEventListener('click', () => {
  init();
});

document.getElementById('exportBtn').addEventListener('click', () => {
  const canvas = document.getElementById('myCanvas');
  const link = document.createElement('a');
  link.download = 'poster-casa-balla.png';
  link.href = canvas.toDataURL('image/png');
  link.click();
});
```

- [ ] **Step 2: Also hide loading flicker — ensure Paper.js canvas matches window size**

Add at top of init():
```js
paper.project.activeLayer.removeChildren();
```

- [ ] **Step 3: Test in browser**

Press "Genera" — a new random poster should appear each time. Press "Esporta PNG" — a PNG download should start.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: wire generate and export buttons"
```

---

### Task 7: Clean up and polish

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Remove test console.log calls from init()**

Remove all `console.log` calls from `init()` and test code blocks.

- [ ] **Step 2: Ensure canvas always matches window size**

Add resize handler:

```js
function resizeCanvas() {
  const container = document.getElementById('canvasContainer');
  const canvas = document.getElementById('myCanvas');
  canvas.width = container.clientWidth;
  canvas.height = container.clientHeight;
}

window.addEventListener('resize', () => {
  resizeCanvas();
  init();
});
```

- [ ] **Step 3: Test in browser**

Confirm no console.log noise. Resize window — poster regenerates and fits.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "chore: cleanup console logs and add resize handling"
```
