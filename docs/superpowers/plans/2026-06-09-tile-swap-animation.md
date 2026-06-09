# Tile Swap Animation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) for syntax tracking.

**Goal:** Add periodic tile-swap animation with stretch-smear effect to the poster generator.

**Architecture:** All changes in `index.html` inline JS. New global state tracks previous layout positions. A 5s `setInterval` triggers a new random layout, then `view.onFrame` interpolates tiles from old positions to new with a stretch-smear deformation. `renderPoster` attaches `_item` references to layout objects for animation lookup.

**Tech Stack:** Paper.js (CDN), Vanilla JS

**Files modified:** `index.html` only

---

### Task 1: Add global animation state

**Modify:** `index.html:94-95`

- [ ] **Step 1: Add state variables**

Replace:

```javascript
    let tileSymbols = [];
    let specialSymbols = [];
```

With:

```javascript
    let tileSymbols = [];
    let specialSymbols = [];
    let swapTimer = null;
    let isAnimating = false;
    let animatingTiles = [];
    let prevSpecials = [];
    let prevTiles = [];
    let currentSpecials = [];
    let currentTiles = [];
    let currentCs = 0;
```

---

### Task 2: Attach item references in renderPoster

**Modify:** `index.html:255-279` — `renderPoster()`

- [ ] **Step 1: Save `_item` references during placement**

Replace the `for (const s of placedSpecials)` and `for (const t of placedTiles)` loops inside `renderPoster`:

```javascript
      for (const s of placedSpecials) {
        const inst = specialSymbols[s.index].symbol.place([0, 0]);
        placeAndFill(inst, s.col * cs, s.row * cs, s.w * cs, s.h * cs);
        s._item = inst;
      }

      for (const t of placedTiles) {
        const inst = tileSymbols[t.tileIndex].place([0, 0]);
        placeAndFill(inst, t.col * cs, t.row * cs, t.size * cs, t.size * cs);
        t._item = inst;
      }
```

---

### Task 3: Save layout and start timer after init

**Modify:** `index.html:297-304` — inside `init()`

- [ ] **Step 1: Update the try block in init()**

Replace lines:

```javascript
        const specials = placeSpecials();
        const tiles = placeTiles();
        renderPoster(specials, tiles, cs);
      } catch (e) {
        showError(e.message);
      } finally {
        generating = false;
      }
```

With:

```javascript
        const specials = placeSpecials();
        const tiles = placeTiles();
        renderPoster(specials, tiles, cs);
        currentSpecials = specials;
        currentTiles = tiles;
        currentCs = cs;
        startSwapTimer();
      } catch (e) {
        showError(e.message);
      } finally {
        generating = false;
      }
```

- [ ] **Step 2: Add startSwapTimer function**

Before `init()`, add:

```javascript
    function startSwapTimer() {
      if (swapTimer) clearInterval(swapTimer);
      swapTimer = setInterval(() => {
        if (isAnimating || generating) return;
        triggerSwap();
      }, 5000);
    }
```

---

### Task 4: Add conflict detection

**Modify:** `index.html` — add helpers before `init()`

- [ ] **Step 1: Add positionsEqual and hasSamePositionConflict**

After `startSwapTimer`, add:

```javascript
    function positionsEqual(a, b) {
      return a.row === b.row && a.col === b.col;
    }

    function hasSamePositionConflict(newSpecials, newTiles, oldSpecials, oldTiles) {
      for (const ns of newSpecials) {
        for (const os of oldSpecials) {
          if (ns.index === os.index && positionsEqual(ns, os)) return true;
        }
      }
      for (const nt of newTiles) {
        for (const ot of oldTiles) {
          if (nt.tileIndex === ot.tileIndex && positionsEqual(nt, ot)) return true;
        }
      }
      return false;
    }
```

---

### Task 5: Implement triggerSwap and onFrame

**Modify:** `index.html` — add after `hasSamePositionConflict`

- [ ] **Step 1: Add triggerSwap function**

Add before `init()`:

```javascript
    async function triggerSwap() {
      isAnimating = true;
      try {
        const cols = gridCols;
        const rows = gridRows;
        const cs = currentCs;

        prevSpecials = currentSpecials;
        prevTiles = currentTiles;

        let specials, tiles, attempts = 0;
        do {
          initGrid(cols, rows);
          specials = placeSpecials();
          tiles = placeTiles();
          attempts++;
        } while (hasSamePositionConflict(specials, tiles, prevSpecials, prevTiles) && attempts < 20);

        renderPoster(specials, tiles, cs);

        const animItems = [];
        for (const ns of specials) {
          const os = prevSpecials.find(s => s.index === ns.index);
          if (!os) continue;
          const item = ns._item;
          const oldPos = new paper.Point((os.col + os.w / 2) * cs, (os.row + os.h / 2) * cs);
          const newPos = new paper.Point((ns.col + ns.w / 2) * cs, (ns.row + ns.h / 2) * cs);
          if (oldPos.getDistance(newPos) < 1) continue;
          item.position = oldPos;
          animItems.push({
            item,
            oldPos: oldPos.clone(),
            newPos: newPos.clone(),
            baseMatrix: item.matrix.clone(),
            angle: newPos.subtract(oldPos).angle
          });
        }

        for (const nt of tiles) {
          const ot = prevTiles.find(t => t.tileIndex === nt.tileIndex);
          if (!ot) continue;
          const item = nt._item;
          const oldPos = new paper.Point((ot.col + ot.size / 2) * cs, (ot.row + ot.size / 2) * cs);
          const newPos = new paper.Point((nt.col + nt.size / 2) * cs, (nt.row + nt.size / 2) * cs);
          if (oldPos.getDistance(newPos) < 1) continue;
          item.position = oldPos;
          animItems.push({
            item,
            oldPos: oldPos.clone(),
            newPos: newPos.clone(),
            baseMatrix: item.matrix.clone(),
            angle: newPos.subtract(oldPos).angle
          });
        }

        currentSpecials = specials;
        currentTiles = tiles;
        animatingTiles = animItems;
      } catch (e) {
        showError(e.message);
        isAnimating = false;
      }
    }
```

- [ ] **Step 2: Add onFrame animation handler**

After `renderPoster`, replace the existing `paper.setup('myCanvas');` with the full view setup (find lines around 243):

```javascript
    paper.setup('myCanvas');

    paper.view.onFrame = function(event) {
      if (!isAnimating || animatingTiles.length === 0) return;
      const duration = 1.5;
      const t = Math.min(event.time / duration, 1);

      for (const a of animatingTiles) {
        a.item.matrix = a.baseMatrix.clone();
        const pt = a.oldPos.clone().add(a.newPos.subtract(a.oldPos).multiply(t));
        a.item.position = pt;

        const smear = 1 + Math.sin(t * Math.PI) * 2.5;
        const center = a.item.bounds.center;
        a.item.rotate(-a.angle, center);
        a.item.scale(smear, 1 / Math.sqrt(smear), center);
        a.item.rotate(a.angle, center);
      }

      if (t >= 1) {
        isAnimating = false;
        animatingTiles = [];
        renderPoster(currentSpecials, currentTiles, currentCs);
      }
    };
```

---

### Task 6: Handle "Genera" button and resize

**Modify:** `index.html:315-325`

- [ ] **Step 1: Update resize handler**

Replace:

```javascript
    window.addEventListener('resize', () => {
      resizeCanvas();
      if (gridCols > 0) init();
    });
```

With:

```javascript
    window.addEventListener('resize', () => {
      resizeCanvas();
      if (gridCols > 0 && !isAnimating) init();
    });
```

- [ ] **Step 2: Update "Genera" button handler**

Replace:

```javascript
    document.getElementById('generateBtn').addEventListener('click', () => {
      init();
    });
```

With:

```javascript
    document.getElementById('generateBtn').addEventListener('click', () => {
      if (swapTimer) clearInterval(swapTimer);
      swapTimer = null;
      isAnimating = false;
      animatingTiles = [];
      init();
    });
```

---

### Task 7: Verify the animation works

- [ ] **Step 1: Start the server**

Run: `npx serve .` or `python -m http.server 8000`

- [ ] **Step 2: Test the flow**

Open browser to the server URL. Wait 5 seconds. Observe tiles animate to new positions with stretch-smear effect. Click "Genera" to reset. Verify timer restarts.

- [ ] **Step 3: Test edge cases**

- Change rows/columns and click "Genera" — timer resets, new poster renders
- Verify special tiles (2x3, 2x2, 2x1) move correctly
- Check browser console for errors
- Resize window during animation — should not restart
