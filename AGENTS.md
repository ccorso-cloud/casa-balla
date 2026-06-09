# Casa Balla — Poster Generator

## Panoramica

Generatore di poster digitali che compone tasselli SVG in stile futurista (Giacomo Balla) su una griglia, usando Paper.js.

## Avvio

```bash
npx serve .
# oppure: python -m http.server 8000
```

Necessario HTTP (file:// non funziona con fetch()).

## Asset

### Tasselli normali — `tasselli/`
22 file SVG 1x1 (`Risorsa 8.svg`…`Risorsa 29.svg`) con pattern astratti.

### Speciali — `speciali/`
5 file, ciascuno inserito **una sola volta** per poster:

| File | Taglia |
|------|--------|
| `titolo-2x3.svg` | 2×3 |
| `info-2x2.svg` | 2×2 |
| `contatti-2x1.svg` | 2×1 |
| `maxxi-1x1.svg` | 1×1 |
| `sovrintendenza-1x1.svg` | 1×1 |

## Architettura

Singola pagina `index.html` — tutto inline (HTML + CSS + JS).

### Algoritmo di placement

1. **Speciali** — ordinati per area decrescente, posizionati casualmente (100 tentativi, fallback deterministico). Errore se la griglia è troppo piccola.
2. **Tasselli normali** — scan row-by-row, celle vuote riempite con dimensioni random [3×3, 2×2, 1×1]. Se non ci sta, ritenta con la dimensione inferiore.
3. **Pool** — 22 tasselli mescolati, consumati in ordine. Quando il pool è esaurito, viene ricreato e rimescolato.

### Render

- `paper.Symbol` per riusare ogni SVG senza duplicare il DOM.
- Celle ridimensionate dinamicamente in proporzione alla viewport (`calcCellSize`).
- Zoom 1:1, nessuna scalatura via viewport.

### Pulsanti

- **Genera** — nuovo layout random.
- **Esporta PNG** — download del canvas corrente.

## Specifiche

- `docs/superpowers/specs/2026-06-09-poster-generator-design.md`
- `docs/superpowers/plans/2026-06-09-poster-generator.md`
