# Telegram-iOS — code audit

Auditoría real sobre el código oficial de Telegram-iOS.
**Commit:** `f1dd7a2d` (master, 7 Oct 2026) · **Método:** lectura del código + verificación por `grep`, sin suposiciones.
**Categoría de todos los hallazgos:** *bug fix / optimización* — la única categoría que el
`CONTRIBUTING.md` oficial acepta en PRs.

Cada hallazgo trae `archivo:línea`, el código exacto, y la prueba de que es real.

---

## A. Rendimiento — trabajo muerto en un bucle a 60 fps

### A1. `AudioWaveformComponent.draw()` construye y ordena un histograma que nunca usa
`submodules/TelegramUI/Components/AudioWaveformComponent/Sources/AudioWaveformComponent.swift:553`

`draw(_:)` corre **una vez por frame** mientras se reproduce un mensaje de voz
(`ConstantDisplayLinkAnimator` → `setNeedsDisplay()`, línea 439). En cada frame:

```swift
var bins: [UInt16: Int] = [:]
for i in 0 ..< maxReadSamples {
    ...
    if let count = bins[sample] { bins[sample] = count + 1 } else { bins[sample] = 1 }
}
var sortedSamples: [(UInt16, Int)] = []
var totalCount: Int = 0
for (sample, count) in bins { if sample > 0 { sortedSamples.append((sample, count)); totalCount += count } }
sortedSamples.sort { $0.1 > $1.1 }
```

**Prueba:** `grep -n "sortedSamples\|totalCount\|bins" AudioWaveformComponent.swift` devuelve
**solo estas líneas de construcción** — nunca se leen después. Es un resto de la función
`drawFakeSamplesIfNeeded` (que sigue viva en el nodo viejo `AudioWaveformNode.swift`): ahí el
histograma alimentaba `topCountPercent`; aquí se eliminó el comportamiento pero quedó el cálculo.
Se construye un diccionario y se ordena, 60 veces por segundo, para tirarlo.
**Fix:** borrar las líneas 553–576 (queda solo el bucle de binning que sí se usa). Borrado puro.

### A2. `AudioWaveformNode` — la propiedad `progress` no hace nada
`submodules/TelegramUI/Components/AudioWaveformNode/Sources/AudioWaveformNode.swift:194`

```swift
let diff: CGFloat
let samplePosition = CGFloat(i) / CGFloat(numSamples)
if let position = parameters.progress, abs(position - samplePosition) < 0.01  {
    diff = sampleWidth * 1.5
} else {
    diff = sampleWidth * 1.5          // ← idéntico
}
```

**Prueba:** las dos ramas son byte-idénticas; `samplePosition` y `position` se calculan y no se
usan en ningún otro lado. `progress` se pasa por todo el nodo (se setea desde
`ChatRecordingPreviewInputPanelNode:148`) y **no tiene efecto visual**.
**Fix:** colapsar a `let diff = sampleWidth * 1.5` (o restaurar el valor de playhead que se perdió).

---

## B. Corrección — ramas muertas

### B1. `ChatItemGalleryFooterContentNode` — `isLandscape` sin efecto
`submodules/GalleryUI/Sources/ChatItemGalleryFooterContentNode.swift:1301`

```swift
if isLandscape { panelHeight += 34.0 } else { panelHeight += 34.0 }
```
**Prueba:** ramas idénticas. El layout en landscape pierde su ajuste (o el condicional es basura).
**Fix:** `panelHeight += 34.0` (o restaurar la constante landscape).

### B2. `AccountStateManagementUtils` — `if let _` con ramas idénticas
`submodules/TelegramCore/Sources/State/AccountStateManagementUtils.swift:3507` (y `:3653`)

```swift
let channelPts: Int32
if let _ = updatedState.channelStates[peer.id] { channelPts = pts } else { channelPts = pts }
updatedState.updateChannelState(peer.id, pts: channelPts)
```
**Prueba:** ambas ramas idénticas y el binding `if let _` es inútil. El test de existencia no hace nada.
**Fix:** `let channelPts: Int32 = pts` (o usar el pts existente en la primera rama, si era la intención).

---

## C. Crash — force-unwrap de datos del servidor

### C1. `CallSessionManager` — `connections.first!` sin comprobar vacío
`submodules/TelegramCore/Sources/State/CallSessionManager.swift:1232` (y `:1261`, `:1658`)

```swift
context.state = .active(..., connections: parseConnectionSet(primary: connections.first!,
    alternative: Array(connections[1...])), ...)
```
**Prueba:** `connections` viene de `phoneCallData.connections` (vector del servidor). La línea
vecina **sí** protege su array hermano (`if !versions.isEmpty`, línea 1219), pero `connections`
nunca se comprueba antes de `.first!` y `[1...]`. Una respuesta con `connections` vacío provoca
un trap (`Unexpectedly found nil` / índice fuera de rango).
**Fix:** `guard let primary = connections.first else { … return }` y `Array(connections.dropFirst())`.

---

## D. Accesibilidad — animaciones infinitas que ignoran "Reducir movimiento"

El codebase consulta `UIAccessibility.isReduceMotionEnabled` en **64+ sitios** (`guard !… else { return }`).
Estos componentes no lo hacen (grep = 0 en cada archivo), pese a correr animaciones infinitas:

| # | Archivo:línea | Qué anima | Prueba |
|---|---|---|---|
| D1 | `AudioWaveformComponent.swift:344` (`animateIn`) | el waveform crece 0.8 s al enviar | sin chequeo |
| D2 | `LiveLocationPositionNode/…/ChatMessageLiveLocationPositionNode.swift:16` | pulso `repeatCount = .infinity` | grep 0 |
| D3 | `PeerOnlineMarkerNode/…/PeerOnlineMarkerNode.swift:67` | 3 barras "voz" infinitas | grep 0 |
| D4 | `LiveLocationTimerNode/…/LiveLocationWavesNode.swift:60` | olas por `ConstantDisplayLinkAnimator` | grep 0 |
| D5 | `ConfettiEffect/…/ConfettiView.swift:134` | confeti por display link continuo | grep 0 |

**Fix (patrón ya establecido en el repo):** `guard !UIAccessibility.isReduceMotionEnabled else { … }`
antes de arrancar la animación; dibujar un fotograma estático.

---

## Notas de honestidad

- **10 hallazgos, todos verificados** contra el código en `f1dd7a2d`, con línea y prueba `grep`.
- **Descartados tras verificar** (para no inflar): un `Data(base64Encoded:)!` en
  `ChatMessageInteractiveFileNode:720` — el literal está hardcodeado en el fuente, no es dato de
  atacante; y un `bitsPerSample` en `AudioWaveform.init(bitstream:bitsPerSample:)` que se usa para
  contar muestras pero con offsets hardcodeados a 5 bits — **latente**, porque los 5 llamadores
  pasan siempre `5`.
- **No compilado:** esta máquina no tiene Xcode/macOS. Telegram-iOS solo compila con Bazel + firma
  en un Mac. Todo cambio es borrado puro o un `guard` sobre una API ya usada 64 veces → riesgo bajo,
  pero **no se construyó ni ejecutó aquí**.
