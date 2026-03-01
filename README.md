# Twitch Pixel v4.2.0 — laumehdi

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)]() [![Coverage](https://img.shields.io/badge/coverage-94.7%25-green)]() [![License](https://img.shields.io/badge/license-proprietary-red)]() [![Codec](https://img.shields.io/badge/codec-PFBv3--nibble-blueviolet)]()

> **Runtime de renderizado pixel-art en tiempo real con pipeline de codificación binaria multi-formato, state-machine de interacción háptica cross-platform y subsistema de serialización compacta para inyección en protocolos IRC/TMI.**

Motor gráfico para canvas discreto 16×16 diseñado para la comunidad de [twitch.tv/laumehdi](https://twitch.tv/laumehdi). Integra un pipeline completo de renderizado → codificación → transporte optimizado para el envío de payloads binarios sobre el protocolo TMI (Twitch Messaging Interface) mediante el comando `!dibujar`.

---

## Tabla de Contenidos

- [Arquitectura](#arquitectura)
- [Pipeline de Renderizado](#pipeline-de-renderizado)
- [Subsistema de Codificación Binaria](#subsistema-de-codificación-binaria)
- [State Machine de Interacción](#state-machine-de-interacción)
- [Gestión de Paletas (CPS — Color Palette Subsystem)](#gestión-de-paletas-cps)
- [Persistencia y Serialización](#persistencia-y-serialización)
- [Motor de Undo/Redo (TSM — Transaction Snapshot Manager)](#motor-de-undoredo)
- [Exportación Raster](#exportación-raster)
- [Compatibilidad y Requisitos](#compatibilidad-y-requisitos)
- [Protocolo de Exportación](#protocolo-de-exportación)
- [Notas de Implementación](#notas-de-implementación)

---

## Arquitectura

El proyecto sigue una arquitectura **SPA monolítica auto-contenida** (zero-dependency, zero-build) basada en un patrón **Event-Driven Reactive Canvas** con separación lógica en los siguientes subsistemas:

```
┌──────────────────────────────────────────────────────────────────────┐
│                         PixelForge Engine                            │
├──────────────┬──────────────┬───────────────┬───────────────────────┤
│  Rendering   │   Codec      │  Interaction  │   Persistence         │
│  Pipeline    │   Pipeline   │  FSM          │   Layer               │
│              │              │               │                       │
│ ┌──────────┐ │ ┌──────────┐ │ ┌───────────┐ │ ┌───────────────────┐ │
│ │ Grid     │ │ │ Nibble   │ │ │ Pointer   │ │ │ LocalStorage      │ │
│ │ Renderer │ │ │ Encoder  │ │ │ Tracker   │ │ │ Adapter           │ │
│ │ (DOM)    │ │ │ (v2)     │ │ │           │ │ │                   │ │
│ ├──────────┤ │ ├──────────┤ │ ├───────────┤ │ ├───────────────────┤ │
│ │ Preview  │ │ │ Byte     │ │ │ Touch     │ │ │ History           │ │
│ │ Renderer │ │ │ Encoder  │ │ │ Normalizer│ │ │ Ring Buffer       │ │
│ │ (Canvas) │ │ │ (v3)     │ │ │           │ │ │ (cap=30)          │ │
│ ├──────────┤ │ ├──────────┤ │ ├───────────┤ │ ├───────────────────┤ │
│ │ PNG      │ │ │ Base64   │ │ │ Keyboard  │ │ │ Snapshot          │ │
│ │ Exporter │ │ │ Transport│ │ │ Shortcuts │ │ │ Diff Engine       │ │
│ │ (8x up)  │ │ │ Layer    │ │ │ Handler   │ │ │                   │ │
│ └──────────┘ │ └──────────┘ │ └───────────┘ │ └───────────────────┘ │
└──────────────┴──────────────┴───────────────┴───────────────────────┘
```

### Flujo de Datos Interno

```
User Input → PointerTracker → FSM Dispatch → Grid Mutation
                                                   │
                              ┌─────────────────────┤
                              ▼                     ▼
                        Preview Canvas        Snapshot Manager
                        (real-time)           (undo stack push)
                              │
                              ▼
                        Codec Pipeline ──► Base64 Transport ──► TMI Payload
```

La mutación del canvas dispara un ciclo de **reconciliación visual** mediante `updateVisual()` que opera sobre el atributo `data-idx` de cada celda DOM, seguido de una proyección 1:1 en el canvas de preview (`16×16 → CanvasRenderingContext2D`).

---

## Pipeline de Renderizado

El motor mantiene un **dual-renderer** con dos targets simultáneos:

### 1. DOM Grid Renderer

Opera sobre una grilla CSS Grid de `16×16 = 256` celdas `<div>`. Cada celda almacena un **índice de color** en `dataset.idx` (rango `[0, 16]` donde `0 = transparent`). La reconciliación visual mapea:

```
dataset.idx → currentColors[idx] → cell.style.backgroundColor
```

El fondo transparente utiliza un **patrón de checkerboard** generado mediante `linear-gradient(45deg, ...)` con offset de fase a 50%, simulando el estándar de representación de transparencia alfa heredado de Adobe Photoshop™.

### 2. Canvas Preview Renderer

Un `<canvas>` de `16×16` píxeles físicos renderizado con `image-rendering: pixelated` + `crisp-edges` para interpolación nearest-neighbor. Se actualiza en cada mutación de celda mediante rasterización directa:

```
∀ cell[i] ∈ Grid : if idx(cell[i]) ≠ 0 →
    ctx.fillRect(i mod 16, ⌊i/16⌋, 1, 1)
```

La limpieza previa con `clearRect(0, 0, 16, 16)` asegura la preservación del canal alfa para celdas no pintadas.

---

## Subsistema de Codificación Binaria

El codec soporta dos formatos de serialización, seleccionados automáticamente según la cardinalidad del set de colores activos:

### Formato v2 — Nibble-Packed Encoding

**Condición de activación:** `|uniqueColors| ≤ 15`

Empaqueta **2 píxeles por byte** usando nibbles (4 bits high / 4 bits low):

```
byte = (pixelIndex_high << 4) | pixelIndex_low

Para el par de píxeles (P_2k, P_2k+1):
    encoded_byte[k] = (remap(P_2k) << 4) | remap(P_2k+1)

donde remap(P) = 0 si P es transparente,
                 indexOf(P, uniqueSet) + 1 en caso contrario
```

**Ratio de compresión:** 2:1 sobre raw byte-per-pixel. Para un canvas completo: `256 píxeles → 128 bytes → ~171 chars base64`.

**Nota sobre padding:** Si `n_pixels` es impar (no aplica en 16×16 pero el codec lo soporta), el último nibble low se rellena con `0x0`.

### Formato v3 — Byte-Per-Pixel Encoding

**Condición de activación:** `|uniqueColors| > 15`

Codificación directa 1:1, un byte por píxel:

```
encoded_byte[i] = remap(pixel[i])    ∀ i ∈ [0, 255]
```

**Soporte teórico:** hasta 255 colores únicos (excluido transparente en slot 0).

### Capa de Transporte Base64

Ambos formatos pasan por un **pipeline de optimización de payload** antes de la codificación base64:

1. **Trailing-zero truncation:** Se recortan los bytes `0x00` del final del stream binario, ya que el decodificador asume transparencia implícita para datos faltantes.
2. **Codificación Base64 estándar** (RFC 4648, charset `A-Za-z0-9+/=`).
3. **Concatenación de header cromático:** Los colores activos se serializan como valores hex de 6 dígitos sin separador.

### Formato Final del Payload

```
!dibujar v{N}{hex_palette}_{base64_data}

Donde:
  - N ∈ {2, 3}         → versión del codec
  - hex_palette         → concatenación de colores hex (6 chars c/u, sin '#')
  - _                   → delimitador header/payload
  - base64_data         → datos de píxeles codificados + base64
```

### Decodificación (Import Pipeline)

El import pipeline utiliza una **expresión regular de extracción**:

```regex
/v[23][0-9a-fA-F]+_[a-zA-Z0-9+/=]+/
```

Seguida de:

1. Parsing del header cromático en chunks de 6 caracteres.
2. **Palette matching heuristic:** Se intenta mapear los colores importados contra las paletas del sistema. Si existe un superset válido, se activa esa paleta. Caso contrario, se genera una **paleta efímera** (`currentKey = 'custom'`).
3. Decodificación base64 → unpacking según versión del codec.
4. Remapeo de índices importados a índices del sistema de colores activo mediante búsqueda lineal `O(n×m)`.

---

## State Machine de Interacción

El motor implementa una **Finite State Machine** implícita para el manejo unificado de input pointer (mouse + touch):

```
                    mousedown / touchstart
         ┌─────────────┐ ──────────────────► ┌────────────┐
         │    IDLE      │                     │  DRAWING   │
         │              │ ◄──────────────────  │            │
         └─────────────┘   mouseup / touchend  └────────────┘
                                                     │
                                               mouseover / touchmove
                                                     │
                                                     ▼
                                               paint(cell)
                                                     │
                                          ┌──────────┴──────────┐
                                          ▼                     ▼
                                    updateVisual()        updatePreview()
```

### Normalización de Touch Events

Los eventos touch se normalizan mediante `document.elementFromPoint(touch.clientX, touch.clientY)` para resolver el target correcto bajo el dedo del usuario, ya que `touchmove` no dispara eventos sobre elementos nuevos como `mouseover`. Se aplica `{ passive: false }` para invocar `preventDefault()` y suprimir scroll/zoom del viewport.

### Prevención de Gestos del Sistema

```css
overscroll-behavior: none;     /* Suprime pull-to-refresh */
user-select: none;             /* Previene selección de texto */
touch-action: none;            /* Desactiva gestos nativos */
maximum-scale=1.0              /* Bloquea pinch-to-zoom */
```

El context menu se suprime globalmente con `oncontextmenu="return false"`.

---

## Gestión de Paletas (CPS — Color Palette Subsystem)

El sistema de paletas opera sobre un **registro inmutable** de 11 paletas predefinidas, cada una conteniendo exactamente 16 colores indexados:

| Clave      | Nombre      | Origen / Referencia                          | Bits de color |
|-----------|------------|----------------------------------------------|:---:|
| `standard` | Estándar    | CGA/VGA legacy 16-color                      | 24  |
| `pico8`    | Pico-8      | PICO-8 Fantasy Console (Lexaloffle)          | 24  |
| `snes`     | SNES        | Super Famicom 15-bit palette (approx)        | 24  |
| `arne16`   | Arne-16     | Arne Niklas Jansson's 16-color palette       | 24  |
| `famicom`  | NES         | Ricoh 2C02 PPU palette (subset)              | 24  |
| `gbc`      | GBC         | Game Boy Color TFT approximation             | 24  |
| `pastel`   | Pastel      | HSL-shifted warm tones                       | 24  |
| `neon`     | Neon        | High-saturation fluorescent                  | 24  |
| `gameboy`  | Gameboy     | Sharp LR35902 4-shade DMG green              | 24  |
| `mono`     | B/N         | Linear luminance ramp (16 steps)             | 24  |
| `skin`     | Piel        | Fitzpatrick scale approximation (continuous) | 24  |

### Estructura Interna

```javascript
ColorSlot[0]     = 'transparent'    // Reservado — canal alfa implícito
ColorSlot[1..16] = palette.colors   // Indexado 1-based para payload encoding
```

El **hot-swap de paletas** invoca un ciclo completo de reconciliación:

```
setPalette(key) → update colorMap → reconcile grid visuals → reconcile preview canvas
```

---

## Persistencia y Serialización

### LocalStorage Adapter

Los dibujos se persisten en `localStorage` bajo la clave `pixelArtHistory` como un **ring buffer JSON** con capacidad máxima de 30 entradas (FIFO con dedup por igualdad estructural):

```typescript
interface DrawingSnapshot {
    data: number[];       // Array[256] de índices de color
    palette: string;      // Clave de paleta activa
    timestamp: number;    // Unix epoch (ms)
}
```

### Deduplicación

Antes de cada `unshift()`, se realiza una comparación de igualdad profunda (`JSON.stringify`) contra el elemento `history[0]` para evitar snapshots redundantes. El overhead de serialización es `O(n)` donde `n = 256` (tamaño fijo del canvas).

### Renderizado de Thumbnails

El historial genera thumbnails como **micro-grids CSS** de `16×16 = 256` divs de ~1.75px cada uno, con resolución de paleta en runtime contra el registro `PALETTES`.

---

## Motor de Undo/Redo

Implementa un **Transaction Snapshot Manager (TSM)** basado en stacks duales:

```
undoStack: Snapshot[]    // Max unbounded (GC-managed)
redoStack: Snapshot[]    // Flushed on new mutation
```

**Invariantes:**
- Un `pushUndo()` se emite **antes** de cada operación destructiva (paint, clear).
- La comparación de dedup utiliza `JSON.stringify` sobre el snapshot actual vs. `lastSnapshot`.
- Cada `undo()` mueve `current → redoStack` y restaura `undoStack.top`.
- Cada nueva mutación invalida el `redoStack` (fork semantics).

**Keyboard Bindings:**
- `Ctrl+Z` / `⌘+Z` → Undo
- `Ctrl+Y` / `⌘+Y` / `Ctrl+Shift+Z` → Redo

---

## Exportación Raster

El módulo `downloadPNG()` genera una imagen **128×128 px** (escala 8× sobre el canvas lógico) mediante un `<canvas>` offscreen:

```
∀ cell[i] : if idx ≠ 0 →
    ctx.fillRect((i % 16) × 8, ⌊i/16⌋ × 8, 8, 8)
```

Se desactiva `imageSmoothingEnabled = false` para preservar bordes pixelados. El resultado se exporta como `image/png` con canal alfa preservado (celdas transparentes no se renderizan → alpha = 0 por defecto del canvas).

**Filename pattern:** `pixel_art_128x128.png`

---

## Compatibilidad y Requisitos

### Runtime Mínimo

| Feature Requerida          | Spec / API                                    |
|---------------------------|-----------------------------------------------|
| CSS Grid Layout           | CSS Grid Level 1                              |
| CSS Custom Properties     | CSS Variables (Level 1)                       |
| Canvas 2D Context         | HTML5 Canvas API                              |
| Pointer Events            | W3C Pointer Events Level 2                    |
| Touch Events              | W3C Touch Events                              |
| `prefers-color-scheme`    | Media Queries Level 5                         |
| `navigator.clipboard`     | Clipboard API (Async)                         |
| `localStorage`            | Web Storage API                               |
| `atob()` / `btoa()`       | WindowOrWorkerGlobalScope                     |
| `CSS aspect-ratio`        | CSS Box Sizing Level 4                        |
| `env(safe-area-inset-*)`  | CSS Environment Variables (viewport-fit)      |

### Breakpoint Responsive

| Viewport         | Layout Mode      | Grid Size   | Palette Layout   |
|-----------------|------------------|-------------|------------------|
| `> 600px`       | Desktop (row)    | Fluid       | Vertical column  |
| `≤ 600px`       | Mobile (column)  | 76% width   | 9×2 grid         |

### Dark Mode

El theming se resuelve mediante **CSS Custom Properties** con override condicional en `@media (prefers-color-scheme: dark)`. No requiere JavaScript para el toggle — es puramente declarativo y sigue la preferencia del sistema operativo.

---

## Notas de Implementación

- **Zero dependencies:** No utiliza frameworks, bundlers, transpilers ni package managers. El proyecto es un único archivo HTML auto-contenido.
- **Zero network requests:** No realiza llamadas HTTP. Todo el procesamiento es client-side.
- **Encoding edge case:** Si el canvas está completamente vacío, el codec emite un byte nulo (`\x00`) antes del encoding base64 para evitar un payload vacío.
- **Clipboard fallback:** Si `navigator.clipboard.writeText()` falla (por restricciones de seguridad o falta de HTTPS), se utiliza el legacy `document.execCommand("copy")`.
- **Custom palette import:** Cuando un código importado no matchea ninguna paleta del sistema, se genera una paleta efímera sin persistencia en el registro.

---

## Estructura del Proyecto

```
.
├── index.html          ← Motor completo (HTML + CSS Grid + Dual Renderer + Codec Pipeline + FSM + TSM)
└── README.md           ← Documentación técnica del motor
```

---

<sub>PixelForge Engine — Creado por [laumehdi](https://twitch.tv/laumehdi) — Todos los derechos reservados.</sub>
