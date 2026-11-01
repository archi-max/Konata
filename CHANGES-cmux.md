# Changes on the `cmux` branch

This branch is based on shioyadan/Konata (via archi-max/Konata, 866f516). It adds a horizontal
scale that does not depend on the vertical zoom, so the pipeline canvas can line up column for
column with a waveform time axis. cmux uses this for its pipeline view next to waveforms.

## Why

Upstream Konata derives both axes from `zoomLevel`: an op is `32 * 2^-zoomLevel` px wide and
`24 * 2^-zoomLevel` px tall. A waveform viewer needs any pixels-per-cycle value (for example
0.0005 px/cycle when zoomed far out, up to 400 px/cycle), while the rows stay readable (24 px).

## API

All additions are in `src/core/konata_renderer.ts` unless noted.

- `KonataRenderSpec.cycleWidth?: number`: CSS px per cycle. When it is set (finite and > 0),
  `opWidth` and `laneWidth` equal it. Row height, margins, the `canDraw*` thresholds, fonts,
  `drawingStep` and `opResolution` still follow `zoomLevel`. It is not part of
  `DEFAULT_KONATA_RENDER_SPEC`.
- `isValidKonataCycleWidth(cycleWidth: number | undefined): cycleWidth is number`
- `getKonataCycleWidth(spec: Pick<KonataRenderSpec, "zoomLevel" | "cycleWidth">): number`:
  the effective horizontal px per cycle.
- `KonataRenderMetrics.withCycleWidth(cycleWidth: number | undefined, anchorPixelX = 0)`: returns
  a spec with the new width (`undefined` removes it). The cycle under `anchorPixelX` stays fixed.
- `KonataRenderMetrics.horizontallyCompressed: boolean`: true when `opWidth` is smaller than
  the zoomLevel default.
- `KonataView` may carry `cycleWidth`. `getKonataView` copies it only when it is present.
- `getNamespaceKey(contentKey, spec)` in `tiled_pipeline_renderer.ts`: the tile cache key.
  It is unchanged when `cycleWidth` is unset and gets `|cycle:<w>` appended when it is set.

## Behaviour

- **Unset `cycleWidth`.** Behaviour is the same as upstream. Metrics, tile keys, navigator scales
  and drawing commands do not change. A test checks that a spec with `cycleWidth` equal to the
  zoom width records exactly the same canvas commands as a spec without it.
- **`withZoomLevel`.** The horizontal pointer compensation uses the `opWidth` ratio. With a fixed
  `cycleWidth`, zooming changes only the row height, and `position[0]` stays put. Without
  `cycleWidth` the ratio is the same as before.
- **View controller.** It interpolates `cycleWidth` log-linearly when both ends of a transition
  have one. `zoomAt` keeps the cycle under the centre fixed. Otherwise it uses the target's value.
- **Cycle navigator.** The thumb and the cycle scale use the effective width.
- **Store.** `KONATA_SET_VIEW` keeps `cycleWidth` when the view carries one.
- **Compressed drawing.** Applies when `opWidth` is smaller than the zoom default. The per-cycle
  elapsed-count numbers are drawn only while one fits in a cycle, and the loop stops at the first
  count that does not fit. This fixes unbounded work when about 1e5 cycles of one stage are
  visible. A stage name is drawn only when it fits in the stage's width (at least one cycle).
  Stage rectangles, the overview path and dependency arrows are bounded by the number of visible
  rows, so they are unchanged.

No UI was added. Keyboard horizontal steps in `app.tsx` are still measured in zoomLevel units.
