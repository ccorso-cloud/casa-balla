# Tile Swap Animation — Design Doc

## Obiettivo

Animazione periodica che scambia di posto tutti i tasselli del poster con effetto stretch-smear in stile futurista, usando Paper.js.

## Ciclo

Ogni 5 secondi:
1. Rigenera un nuovo layout (stessa logica di `placeSpecials` + `placeTiles`)
2. Anima ogni tassello dalla vecchia alla nuova posizione (~1.5s)
3. Pausa (~3.5s), poi ripeti

## Algoritmo di ri-placement

- **Tasselli normali**: nuova posizione + nuova dimensione casuale (3x3, 2x2, 1x1)
- **Speciali**: nuova posizione, dimensione fissa (2x3, 2x2, 2x1, 1x1, 1x1)
- Logica identica al placement iniziale (`placeSpecials` ordinati per area decrescente, `placeTiles` scan row-by-row)
- Vincolo extra: nessun tassello finisce nella stessa cella in cui era
- Fallback: riprova se un tassello resta nella stessa posizione

## Animazione stretch-smear

Ogni tassello si muove da posizione A a B, interpola linearmente.

### Deformazione

Durante il movimento, il tassello si allunga nella direzione di movimento seguendo una curva a campana:

- `t = tempo_animazione / durata` (0 → 1)
- `stretch = 1 + sin(t × π) × 2.5`
- Allungamento lungo l'asse di movimento: `scale(stretch, 1/√stretch)`
- A t=0 e t=1: forma normale
- A t=0.5: massimo allungamento (~3.5x)

### Implementazione Paper.js

Per ogni frame:
1. `item.position = lerp(A, B, t)`
2. Comporre matrice di stretch: `rotate(-angolo) → scale(stretch, 1/√stretch) → rotate(angolo)`
3. Applicare combinata alla matrice base del tassello
4. Ogni frame si riparte dalla matrice base per evitare accumulo

## Integrazione in index.html

### Nuove variabili globali

- `swapTimer`: ID del `setInterval` da 5 secondi
- `isAnimating`: flag per evitare animazioni sovrapposte
- `prevLayout[]`: array `{ row, col, w, h }` dello stato prima dello swap
- `animatingTiles[]`: array `{ item, oldPos, newPos, baseMatrix, angle }`

### Flusso dettagliato

Dopo ogni `renderPoster()`:
1. Salva lo stato corrente in `prevLayout`
2. Avvia `swapTimer` (se non già attivo)

Allo scadere del timer (5s), se `!isAnimating`:
1. Genera nuovo layout su `grid` (placeSpecials + placeTiles)
2. Chiama `renderPoster()` → layer pulito, nuovi PlacedSymbol alle nuove posizioni
3. Per ogni nuovo tassello, cerca la sua posizione precedente in `prevLayout`:
   - Calcola `oldPos` (centro della vecchia cella in pixel)
   - Calcola `newPos` (centro della nuova cella in pixel — posizione attuale)
4. Imposta `item.position = oldPos` (sposta il tassello al punto di partenza)
5. Popola `animatingTiles[]` con `{ item, oldPos, newPos, baseMatrix: item.matrix.clone(), angle }`
6. `isAnimating = true`, avvia `onFrame`

`view.onFrame`:
1. Se `!isAnimating` o `animatingTiles` vuoto → return
2. `t = min(framesSinceStart / totalFrames, 1)`
3. Per ogni tile in `animatingTiles`:
   - `item.matrix = baseMatrix.clone()`
   - `item.position = lerp(oldPos, newPos, t)`
   - `smear = 1 + sin(t × π) × 2.5`
   - Applica stretch: `rotate(-angle, center) → scale(smear, 1/√smear, center) → rotate(angle, center)`
4. Se `t >= 1`: animazione finita → `isAnimating = false`, `animatingTiles = []`

### Pulsanti e resize

- **"Genera"**: azzera `swapTimer`, `isAnimating = false`, fa reset completo (init)
- **Resize**: se animazione in corso, lascia completare; non riavvia `swapTimer`

### Mapping vecchie posizioni

`placedSpecials` e `placedTiles` vengono salvati in variabili globali dopo ogni `renderPoster()`. Per abbinare ogni tassello nuovo alla sua vecchia posizione, si usa:
- Tasselli normali: `symbolIndex` (indice nel pool dei normali)
- Speciali: `specialIndex` (indice in SPECIALS)
- Il confronto si fa subito dopo `renderPoster`, iterando i nuovi `placedSpecials`/`placedTiles` e cercando il corrispondente in `prevLayout` per stesso index/tipo

## Note tecniche

- `placeAndFill` riutilizzato per posizionare/scalare i nuovi tasselli
- Stretch applicato solo durante animazione; dopo il completamento i tasselli sono in forma normale
- L'animazione usa gli stessi PlacedSymbol di renderPoster, spostandoli prima al vecchio posto
- Nessuna modifica agli SVG esistenti
