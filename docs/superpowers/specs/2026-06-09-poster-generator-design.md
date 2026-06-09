# Poster Generator — Design Doc

## Obiettivo

Generatore di poster digitali che compone una griglia di tasselli SVG ispirati a Giacomo Balla, usando Paper.js per il rendering.

## Griglia

- L'utente sceglie numero di righe e colonne tramite input numerici
- Ogni cella è 168.38×168.38 unità Paper.js (viewBox dei tasselli 1x1)
- Canvas dimensionata a `(cols × 168.38) × (rows × 168.38)`, centrata nella finestra
- Sfondo della canvas: bianco (`#fff`)

## Asset

**Tasselli normali** (`tasselli/`): 22 file SVG 1x1 con pattern astratti in stile futurista.

**Speciali** (`speciali/`):
| File | Taglia |
|------|--------|
| titolo-2x3.svg | 2×3 |
| info-2x2.svg | 2×2 |
| contatti-2x1.svg | 2×1 |
| maxxi-1x1.svg | 1×1 |
| sovrintendenza-1x1.svg | 1×1 |

Tutti gli SVG vengono caricati come `paper.Symbol` per istanziarli senza duplicare il DOM.

## Algoritmo di Placement

### Fase 1 — Speciali
1. Per ogni speciale (in ordine decrescente di area: 2×3, 2×2, 2×1, 1×1, 1×1):
   - Genera una posizione (riga, colonna) casuale che rientri nella griglia
   - Controlla che tutte le celle coperte siano libere
   - Se collisione, riprova (max 100 tentativi, altrimenti errore "Griglia troppo piccola")
   - Blocca le celle e posiziona il symbol

### Fase 2 — Tasselli Normali
1. Pool dei 22 tasselli mescolati casualmente
2. Scan row-by-row della griglia:
   - Per ogni cella vuota, scegli dimensione casuale tra [3, 2, 1]
   - Controlla che le celle necessarie siano tutte vuote e in-griglia
   - Se non ci sta, scala alla dimensione inferiore (3→2→1)
   - Preleva un tassello dal pool (lo ricrea e rimescola se esaurito)
   - Istanzia il symbol nella posizione con la dimensione scelta

### Fase 3 — Fallback
- Se uno speciale non trova posto in 100 tentativi, la UI mostra un messaggio d'errore e l'utente può riprovare con più colonne/righe.

## Interfaccia Utente

- Singola pagina HTML (`index.html`)
- Input: righe (default 8), colonne (default 6), pulsante "Genera"
- Canvas Paper.js a schermo intero col poster generato
- Pulsante "Esporta PNG" per salvare il risultato
- Stile minimale, coerente con l'estetica del progetto

## Strategia di ripetizione tasselli

"Consuma tutto, poi ripeti":
1. Pool = shuffle dei 22 tasselli
2. Si preleva in ordine dal pool
3. Quando il pool è vuoto, si ricrea mescolando da capo

## Tecnologia

- **Paper.js** via CDN per rendering SVG e manipolazione canvas
- **Vanilla JS** — nessun framework
- File SVG caricati con `fetch()` e importati con `paper.project.importSVG()`
