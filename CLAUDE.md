# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Comandos

Proyecto sin dependencias, sin `package.json`, sin build ni bundler. Solo tres archivos estáticos (`index.html`, `style.css`, `game.js`).

- **Ejecutar el juego**: abrir `index.html` directamente en el navegador, o servirlo con cualquier servidor estático:
  ```bash
  python3 -m http.server 8000
  npx serve .
  ```
- **No hay tests, linter ni proceso de build** — no existen comandos de `npm test`, `npm run build`, etc.

## Arquitectura

Todo el estado y la lógica del juego viven en `game.js` (un único archivo, sin módulos), organizado en torno a variables globales mutables (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.) y funciones que operan sobre ellas. No hay clases ni un objeto de estado encapsulado.

Puntos clave para entender el flujo antes de modificar algo:

- **Tablero**: matriz `ROWS × COLS` (20×10). Cada celda es `0` (vacía) o un índice 1–7 que indexa `COLORS`/`PIECES` para identificar a qué pieza y color pertenece un bloque ya fijado.
- **Piezas**: matrices cuadradas fijas en `PIECES`. La rotación (`rotateCW`) es una transposición + reverso de filas, es decir, siempre se regenera la forma completa en vez de tener 4 rotaciones precalculadas por pieza.
- **Colisión y wall kicks**: `collide(shape, ox, oy)` es la única función de detección de choques (bordes + bloques fijados) y la usan tanto el movimiento normal como `tryRotate`, `ghostY`, `softDrop`/`hardDrop` y `spawn`. `tryRotate` prueba una serie de desplazamientos (`kicks = [0, -1, 1, -2, 2]`) contra `collide` antes de descartar la rotación — cualquier cambio a las reglas de rotación pasa por aquí.
- **Bucle de juego**: `loop(ts)` corre vía `requestAnimationFrame`, acumula delta time en `dropAccum` y cuando supera `dropInterval` baja la pieza o llama a `lockPiece()`. `dropInterval` se recalcula en `clearLines()` según el nivel (`max(100, 1000 - (level-1)*90)`).
- **Ciclo de una pieza**: `spawn()` promueve `next` a `current` y genera un nuevo `next`; si la nueva pieza colisiona de inmediato, dispara `endGame()`. `lockPiece()` encadena `merge()` → `clearLines()` → `spawn()`.
- **Renderizado**: `draw()` limpia el canvas y dibuja en orden grid → bloques fijados → ghost piece (`ghostY()`, alpha 0.2) → pieza actual. El canvas de "next" (`drawNext`) es independiente y se redibuja solo en `spawn()`.
- **Puntuación/niveles**: tabla `LINE_SCORES` multiplicada por `level`; hard drop suma 2 pts/celda, soft drop 1 pt/fila; el nivel sube cada 10 líneas acumuladas (`lines`).
- **Input**: un único listener de `keydown` en `document` hace de enrutador de todas las acciones (mover, rotar, soft/hard drop, pausa); se ignora si `paused` o `gameOver` están activos (salvo `KeyP`, que siempre puede des/pausar).

Si se cambian `COLS`, `ROWS` o `BLOCK` en `game.js`, hay que ajustar también `width`/`height` de `<canvas id="board">` en `index.html` para que coincidan (`COLS × BLOCK`, `ROWS × BLOCK`).
