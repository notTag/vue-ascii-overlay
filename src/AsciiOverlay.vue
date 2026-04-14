<script lang="ts">
// ─── Types ────────────────────────────────────────────────────────────────────

export interface AsciiOverlayFluidConfig {
  enabled?: boolean
  resolutionScale?: number
  velocityDissipation?: number
  densityDissipation?: number
  diffusion?: number
  iterations?: number
  forceScale?: number
  hoverRadiusPx?: number
  strength?: number
  splashRangePx?: number
  splashVelocityScale?: number
  splashForceScale?: number
  splashDensity?: number
  splashRandomness?: number
  splashThicknessPx?: number
  splashTravelEasePower?: number
  splashForceDecayPower?: number
  splashDensityDecayPower?: number
  dragBoostScale?: number
  dragBoostMaxPx?: number
  dragThresholdPx?: number
  project?: boolean
  projectIterations?: number
}

export interface AsciiOverlayProps {
  fontSize?: number
  charWidth?: number
  cellPadding?: { x: number; y: number }
  fps?: number
  charset?: string
  contrast?: number
  gamma?: number
  invertLuma?: boolean
  lumaValue?: number
  lumaSmoothingMs?: number
  backgroundColor?: string
  color?: string
  fontFamily?: string
  fluid?: AsciiOverlayFluidConfig
  disableHover?: boolean
  disableClick?: boolean
  disableDrag?: boolean
}

interface FluidGrid {
  width: number
  height: number
  size: number
  density: Float32Array
  density0: Float32Array
  velX: Float32Array
  velY: Float32Array
  velX0: Float32Array
  velY0: Float32Array
  pressure: Float32Array
  divergence: Float32Array
}

interface SplashRing {
  normX: number
  normY: number
  start: number
  seed: number
  maxRadiusPx: number
  thicknessPx: number
}

interface GridLayout {
  widthCss: number
  heightCss: number
  cols: number
  rows: number
  dpr: number
  tileW: number
  tileH: number
  cellWidth: number
  cellHeight: number
  drawCellWidth: number
  drawCellHeight: number
  padX: number
  padY: number
  spaceIndex: number
}

// ─── Pure Utilities ───────────────────────────────────────────────────────────

function clamp(v: number, min: number, max: number): number {
  return Math.min(max, Math.max(min, v))
}

function clamp01(v: number): number {
  return v < 0 ? 0 : v > 1 ? 1 : v
}

function smoothstep(x: number): number {
  const t = clamp01(x)
  return t * t * (3 - 2 * t)
}

function idx(x: number, y: number, w: number): number {
  return x + y * w
}

function bilinear(
  field: Float32Array,
  x: number,
  y: number,
  w: number,
  h: number,
): number {
  const x0 = Math.floor(x)
  const y0 = Math.floor(y)
  const x1 = Math.min(x0 + 1, w - 1)
  const y1 = Math.min(y0 + 1, h - 1)
  const sx = x - x0
  const sy = y - y0
  const cx = clamp(x0, 0, w - 1)
  const cy = clamp(y0, 0, h - 1)
  return (
    field[cx + cy * w] * (1 - sx) * (1 - sy) +
    field[x1 + cy * w] * sx * (1 - sy) +
    field[cx + y1 * w] * (1 - sx) * sy +
    field[x1 + y1 * w] * sx * sy
  )
}

function pseudoRandom(n: number): number {
  const x = Math.sin(n) * 43758.5453
  return x - Math.floor(x)
}

// ─── Fluid Simulation (Stam-style Navier-Stokes) ─────────────────────────────

function createFluidGrid(w: number, h: number): FluidGrid {
  const size = w * h
  return {
    width: w,
    height: h,
    size,
    density: new Float32Array(size),
    density0: new Float32Array(size),
    velX: new Float32Array(size),
    velY: new Float32Array(size),
    velX0: new Float32Array(size),
    velY0: new Float32Array(size),
    pressure: new Float32Array(size),
    divergence: new Float32Array(size),
  }
}

function fluidDiffuse(
  x: Float32Array,
  x0: Float32Array,
  diff: number,
  iter: number,
  w: number,
  h: number,
): void {
  if (diff <= 0) {
    x.set(x0)
    return
  }
  const a = diff
  const div = 1 + 4 * a
  for (let k = 0; k < iter; k++) {
    for (let j = 1; j < h - 1; j++) {
      for (let i = 1; i < w - 1; i++) {
        const c = idx(i, j, w)
        x[c] =
          (x0[c] +
            a *
              (x[idx(i - 1, j, w)] +
                x[idx(i + 1, j, w)] +
                x[idx(i, j - 1, w)] +
                x[idx(i, j + 1, w)])) /
          div
      }
    }
  }
}

function fluidProject(
  velX: Float32Array,
  velY: Float32Array,
  pressure: Float32Array,
  divergence: Float32Array,
  iter: number,
  w: number,
  h: number,
): void {
  const n = Math.max(w, h)
  // Compute divergence
  for (let j = 1; j < h - 1; j++) {
    for (let i = 1; i < w - 1; i++) {
      const c = idx(i, j, w)
      divergence[c] =
        (-0.5 *
          (velX[idx(i + 1, j, w)] -
            velX[idx(i - 1, j, w)] +
            velY[idx(i, j + 1, w)] -
            velY[idx(i, j - 1, w)])) /
        n
    }
  }
  pressure.fill(0)
  // Gauss-Seidel pressure solve
  for (let k = 0; k < iter; k++) {
    for (let j = 1; j < h - 1; j++) {
      for (let i = 1; i < w - 1; i++) {
        const c = idx(i, j, w)
        pressure[c] =
          (divergence[c] +
            pressure[idx(i - 1, j, w)] +
            pressure[idx(i + 1, j, w)] +
            pressure[idx(i, j - 1, w)] +
            pressure[idx(i, j + 1, w)]) /
          4
      }
    }
  }
  // Subtract pressure gradient from velocity
  for (let j = 1; j < h - 1; j++) {
    for (let i = 1; i < w - 1; i++) {
      const c = idx(i, j, w)
      velX[c] -= 0.5 * n * (pressure[idx(i + 1, j, w)] - pressure[idx(i - 1, j, w)])
      velY[c] -= 0.5 * n * (pressure[idx(i, j + 1, w)] - pressure[idx(i, j - 1, w)])
    }
  }
}

function fluidAdvect(
  d: Float32Array,
  d0: Float32Array,
  velX: Float32Array,
  velY: Float32Array,
  dt: number,
  w: number,
  h: number,
): void {
  for (let j = 1; j < h - 1; j++) {
    for (let i = 1; i < w - 1; i++) {
      const c = idx(i, j, w)
      const x = clamp(i - dt * velX[c], 0.5, w - 1.5)
      const y = clamp(j - dt * velY[c], 0.5, h - 1.5)
      d[c] = bilinear(d0, x, y, w, h)
    }
  }
}

function fluidStep(
  grid: FluidGrid,
  dtSec: number,
  config: {
    diffusion: number
    iterations: number
    velocityDissipation: number
    densityDissipation: number
    project: boolean
    projectIterations: number
  },
): void {
  if (dtSec <= 0) return
  const dt = Math.min(dtSec, 0.1)
  const { width: w, height: h } = grid

  // Diffuse velocity
  grid.velX0.set(grid.velX)
  grid.velY0.set(grid.velY)
  fluidDiffuse(grid.velX, grid.velX0, config.diffusion, config.iterations, w, h)
  fluidDiffuse(grid.velY, grid.velY0, config.diffusion, config.iterations, w, h)
  if (config.project)
    fluidProject(grid.velX, grid.velY, grid.pressure, grid.divergence, config.projectIterations, w, h)

  // Advect velocity
  grid.velX0.set(grid.velX)
  grid.velY0.set(grid.velY)
  fluidAdvect(grid.velX, grid.velX0, grid.velX0, grid.velY0, dt, w, h)
  fluidAdvect(grid.velY, grid.velY0, grid.velX0, grid.velY0, dt, w, h)
  if (config.project)
    fluidProject(grid.velX, grid.velY, grid.pressure, grid.divergence, config.projectIterations, w, h)

  // Advect density
  grid.density0.set(grid.density)
  fluidAdvect(grid.density, grid.density0, grid.velX, grid.velY, dt, w, h)

  // Dissipation
  if (config.velocityDissipation > 0) {
    const factor = Math.exp(-config.velocityDissipation * dt)
    for (let i = 0; i < grid.size; i++) {
      grid.velX[i] *= factor
      grid.velY[i] *= factor
    }
  }
  if (config.densityDissipation > 0) {
    const factor = Math.exp(-config.densityDissipation * dt)
    for (let i = 0; i < grid.size; i++) {
      grid.density[i] *= factor
    }
  }
}

function injectForce(
  grid: FluidGrid,
  cx: number,
  cy: number,
  vx: number,
  vy: number,
  densityAmt: number,
  radius: number,
): void {
  if (radius <= 0 || (vx === 0 && vy === 0 && densityAmt <= 0)) return
  const { width: w, height: h, velX, velY, density } = grid
  const r2 = radius * radius
  const minX = Math.max(0, Math.floor(cx - radius))
  const maxX = Math.min(w - 1, Math.ceil(cx + radius))
  const minY = Math.max(0, Math.floor(cy - radius))
  const maxY = Math.min(h - 1, Math.ceil(cy + radius))
  for (let y = minY; y <= maxY; y++) {
    const dy = y - cy
    const dy2 = dy * dy
    for (let x = minX; x <= maxX; x++) {
      const dx = x - cx
      const d2 = dx * dx + dy2
      if (d2 > r2) continue
      const weight = smoothstep(1 - Math.sqrt(d2) / radius)
      const i = idx(x, y, w)
      velX[i] += vx * weight
      velY[i] += vy * weight
      if (densityAmt > 0) density[i] = clamp01(density[i] + densityAmt * weight)
    }
  }
}

function applySplashForces(
  grid: FluidGrid,
  ring: SplashRing,
  currentRadius: number,
  forceMag: number,
  densityMag: number,
  thickness: number,
  randomness: number,
  scaleX: number,
  scaleY: number,
): void {
  const { width: w, height: h, velX, velY, density } = grid
  const cx = ring.normX * (w - 1)
  const cy = ring.normY * (h - 1)
  const gridRadius = currentRadius * Math.min(scaleX, scaleY)
  const gridThickness = thickness * Math.min(scaleX, scaleY)
  const halfThick = gridThickness / 2
  const innerR = Math.max(0, gridRadius - halfThick)
  const outerR = gridRadius + halfThick

  const minX = Math.max(0, Math.floor(cx - outerR))
  const maxX = Math.min(w - 1, Math.ceil(cx + outerR))
  const minY = Math.max(0, Math.floor(cy - outerR))
  const maxY = Math.min(h - 1, Math.ceil(cy + outerR))

  for (let y = minY; y <= maxY; y++) {
    for (let x = minX; x <= maxX; x++) {
      const dx = x - cx
      const dy = y - cy
      const dist = Math.sqrt(dx * dx + dy * dy)
      if (dist < innerR || dist > outerR || dist < 0.001) continue

      const ringDist = halfThick > 0 ? Math.abs(dist - gridRadius) / halfThick : 0
      let weight = smoothstep(1 - ringDist)
      if (randomness > 0) {
        const noise = pseudoRandom(ring.seed + x * 31 + y * 97)
        weight *= 1 - randomness + randomness * noise * 2
      }
      weight = Math.max(0, weight)

      const i = idx(x, y, w)
      const nx = dx / dist
      const ny = dy / dist
      if (forceMag > 0) {
        velX[i] += nx * forceMag * weight
        velY[i] += ny * forceMag * weight
      }
      if (densityMag > 0) {
        density[i] = clamp01(density[i] + densityMag * weight)
      }
    }
  }
}

// ─── Atlas Builder ────────────────────────────────────────────────────────────

function buildAtlas(
  canvas: HTMLCanvasElement,
  ctx: CanvasRenderingContext2D,
  charset: string,
  tileW: number,
  tileH: number,
  fillColor: string,
  fontFamily: string,
): boolean {
  if (!tileW || !tileH || charset.length === 0) return false

  canvas.width = tileW * charset.length
  canvas.height = tileH
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  ctx.imageSmoothingEnabled = false

  // Start at tile height, measure, then scale to fit
  let fontSize = tileH
  ctx.font = `${fontSize}px ${fontFamily}`

  const pad = Math.min(1, Math.floor(Math.min(tileW, tileH) / 2))
  const innerW = Math.max(1, tileW - 2 * pad)
  const innerH = Math.max(1, tileH - 2 * pad)

  // Measure all characters to find max bounds
  let maxCharW = 0
  let maxAscent = 0
  let maxDescent = 0
  const metrics: { width: number; ascent: number; descent: number; left: number }[] = []
  for (let i = 0; i < charset.length; i++) {
    const m = ctx.measureText(charset[i])
    const ascent = Number.isFinite(m.actualBoundingBoxAscent)
      ? m.actualBoundingBoxAscent
      : fontSize * 0.8
    const descent = Number.isFinite(m.actualBoundingBoxDescent)
      ? m.actualBoundingBoxDescent
      : fontSize * 0.2
    const width = m.width || fontSize * 0.6
    const left = Number.isFinite(m.actualBoundingBoxLeft) ? -m.actualBoundingBoxLeft : 0
    metrics.push({ width, ascent, descent, left })
    if (width > maxCharW) maxCharW = width
    if (ascent > maxAscent) maxAscent = ascent
    if (descent > maxDescent) maxDescent = descent
  }

  // Scale to fit uniformly
  const totalH = maxAscent + maxDescent
  const scaleX = maxCharW > 0 ? (0.92 * innerW) / maxCharW : 1
  const scaleY = totalH > 0 ? innerH / totalH : 1
  const scale = Math.min(scaleX, scaleY)
  const scaledFontSize =
    Number.isFinite(scale) && scale > 0 ? fontSize * scale : fontSize

  ctx.font = `${scaledFontSize}px ${fontFamily}`
  ctx.fillStyle = fillColor
  ctx.textAlign = 'left'
  ctx.textBaseline = 'alphabetic'

  // Re-measure at final font size for precise placement
  let finalAscent = 0
  let finalDescent = 0
  const finalMetrics: typeof metrics = []
  for (let i = 0; i < charset.length; i++) {
    const m = ctx.measureText(charset[i])
    const ascent = Number.isFinite(m.actualBoundingBoxAscent)
      ? m.actualBoundingBoxAscent
      : scaledFontSize * 0.8
    const descent = Number.isFinite(m.actualBoundingBoxDescent)
      ? m.actualBoundingBoxDescent
      : scaledFontSize * 0.2
    const width = m.width || scaledFontSize * 0.6
    const left = Number.isFinite(m.actualBoundingBoxLeft) ? -m.actualBoundingBoxLeft : 0
    finalMetrics.push({ width, ascent, descent, left })
    if (ascent > finalAscent) finalAscent = ascent
    if (descent > finalDescent) finalDescent = descent
  }

  const baseline = Math.max(pad, Math.floor(pad + innerH - finalDescent))
  for (let i = 0; i < charset.length; i++) {
    const fm = finalMetrics[i]
    const x = Math.round(i * tileW + pad + (innerW - fm.width) / 2 + fm.left)
    ctx.fillText(charset[i], x, baseline)
  }

  return true
}

// ─── Luma Mapper Factory ──────────────────────────────────────────────────────

function createLumaMapper(
  charsetLen: number,
  contrast: number,
  gamma: number,
  invert: boolean,
): (luma: number) => number {
  const c = Number.isFinite(contrast) ? contrast : 1
  const g = Number.isFinite(gamma) ? gamma : 1
  const maxIdx = Math.max(0, charsetLen - 1)
  return (luma: number): number => {
    let v = luma
    if (c !== 1) v = (v - 0.5) * c + 0.5
    v = clamp01(v)
    if (g !== 1) v = Math.pow(v, g)
    v = clamp01(v)
    if (invert) v = 1 - v
    const index = Math.floor(v * (maxIdx + 1))
    return clamp(index, 0, maxIdx)
  }
}
</script>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'

const props = withDefaults(defineProps<AsciiOverlayProps>(), {
  fontSize: 14,
  charWidth: 14,
  cellPadding: () => ({ x: 1, y: 2 }),
  fps: 30,
  charset: ' .:-=+*#%@',
  contrast: 1,
  gamma: 1,
  invertLuma: true,
  lumaValue: 0.7,
  lumaSmoothingMs: 0,
  backgroundColor: '#000000',
  color: '#ffffff',
  fontFamily: "'Courier New', Courier, monospace",
  fluid: () => ({}),
  disableHover: false,
  disableClick: false,
  disableDrag: false,
})

const containerRef = ref<HTMLDivElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)

let cleanupFn: (() => void) | null = null

function setup() {
  const container = containerRef.value!
  const canvas = canvasRef.value!
  if (!container || !canvas) return () => {}

  const mainCtx = canvas.getContext('2d')!
  if (!mainCtx) return () => {}
  mainCtx.imageSmoothingEnabled = false

  // ── Resolve fluid config with defaults ──────────────────────────────────
  const fl = props.fluid ?? {}
  const fluidEnabled = fl.enabled !== false
  const fluidConfig = {
    resolutionScale: clamp(fl.resolutionScale ?? 0.6, 0.2, 1),
    velocityDissipation: Math.max(0, fl.velocityDissipation ?? 1.2),
    densityDissipation: Math.max(0, fl.densityDissipation ?? 0.8),
    diffusion: Math.max(0, fl.diffusion ?? 0.2),
    iterations: Math.max(0, Math.round(fl.iterations ?? 3)),
    forceScale: Math.max(0, fl.forceScale ?? 0.5),
    hoverRadiusPx: Math.max(0, fl.hoverRadiusPx ?? 28),
    strength: clamp(fl.strength ?? 0.6, 0, 1),
    splashRangePx: Math.max(0, fl.splashRangePx ?? 184),
    splashVelocityScale: Math.max(0, fl.splashVelocityScale ?? 3),
    splashForceScale: Math.max(0, fl.splashForceScale ?? 2),
    splashDensity: clamp(fl.splashDensity ?? 0.12, 0, 1),
    splashRandomness: clamp(fl.splashRandomness ?? 0.14, 0, 1),
    splashThicknessPx: Math.max(0, fl.splashThicknessPx ?? 100),
    splashTravelEasePower: Math.max(0.1, fl.splashTravelEasePower ?? 2.5),
    splashForceDecayPower: Math.max(0.1, fl.splashForceDecayPower ?? 1.2),
    splashDensityDecayPower: Math.max(0.1, fl.splashDensityDecayPower ?? 2),
    dragBoostScale: Math.max(0, fl.dragBoostScale ?? 0.03),
    dragBoostMaxPx: Math.max(0, fl.dragBoostMaxPx ?? 60),
    dragThresholdPx: Math.max(0, fl.dragThresholdPx ?? 6),
    project: fl.project !== false,
    projectIterations: Math.max(0, Math.round(fl.projectIterations ?? 8)),
  }

  // ── Measure font aspect ratio ───────────────────────────────────────────
  const savedFont = mainCtx.font
  mainCtx.font = `100px ${props.fontFamily}`
  const mMetrics = mainCtx.measureText('M')
  mainCtx.font = savedFont
  const mAscent = Number.isFinite(mMetrics.actualBoundingBoxAscent)
    ? mMetrics.actualBoundingBoxAscent
    : 80
  const mDescent = Number.isFinite(mMetrics.actualBoundingBoxDescent)
    ? mMetrics.actualBoundingBoxDescent
    : 20
  const mWidth = mMetrics.width || 60
  const aspectRatio = (mAscent + mDescent) / mWidth
  const DEFAULT_ASPECT = 1.6
  const cellAspectRatio =
    Number.isFinite(aspectRatio) && aspectRatio > 0 ? aspectRatio : DEFAULT_ASPECT

  // ── Cell dimensions ─────────────────────────────────────────────────────
  const drawCellWidth = props.charWidth
  const drawCellHeight = drawCellWidth * cellAspectRatio
  const padX = props.cellPadding.x
  const padY = props.cellPadding.y
  const cellWidth = drawCellWidth + 2 * padX
  const cellHeight = drawCellHeight + 2 * padY

  // ── Create offscreen atlas canvas ───────────────────────────────────────
  const atlasCanvas = document.createElement('canvas')
  const atlasCtx = atlasCanvas.getContext('2d')!
  const charset = props.charset.length > 0 ? props.charset : ' '
  const spaceIndex = charset.indexOf(' ')

  // ── Luma mapper ─────────────────────────────────────────────────────────
  // invertLuma only controls mask modulation (reveal vs disturb), not the mapper.
  // Passing it to the mapper would double-invert: mask pushes luma→0, mapper flips 0→1→'@'.
  const lumaToChar = createLumaMapper(charset.length, props.contrast, props.gamma, false)
  const fpsInterval = props.fps > 0 ? 1000 / props.fps : 0

  // ── Mutable state ───────────────────────────────────────────────────────
  let layout: GridLayout = {
    widthCss: 0,
    heightCss: 0,
    cols: 0,
    rows: 0,
    dpr: 1,
    tileW: 0,
    tileH: 0,
    cellWidth,
    cellHeight,
    drawCellWidth,
    drawCellHeight,
    padX,
    padY,
    spaceIndex,
  }
  let fluidGrid: FluidGrid | null = null
  let maskArray: Float32Array | null = null
  let smoothedLuma: Float32Array | null = null
  let lastRenderTime: number | null = null
  let lastFrameTime = 0
  let animFrameId: number | null = null
  let destroyed = false
  let reducedMotion = false
  let atlasColor: string | null = null

  // Mouse/pointer state
  let lastPointer: { x: number; y: number; time: number } | null = null
  let activePointerId: number | null = null
  let pointerDownPos: { x: number; y: number; time: number } | null = null
  let hasDragged = false
  const splashRings: SplashRing[] = []

  // ── Rebuild atlas ───────────────────────────────────────────────────────
  function rebuildAtlas(color?: string) {
    const c = color ?? props.color
    if (
      buildAtlas(atlasCanvas, atlasCtx, charset, layout.tileW, layout.tileH, c, props.fontFamily)
    ) {
      atlasColor = c
    }
  }

  // ── Resize handler ──────────────────────────────────────────────────────
  function handleResize() {
    const rect = container.getBoundingClientRect()
    const w = Math.max(0, rect.width)
    const h = Math.max(0, rect.height)
    if (w === 0 || h === 0) {
      layout = { ...layout, widthCss: 0, heightCss: 0, cols: 0, rows: 0 }
      return
    }

    const dpr = Math.min(window.devicePixelRatio || 1, 2)
    const cols = Math.max(1, Math.floor(w / cellWidth))
    const rows = Math.max(1, Math.floor(h / cellHeight))
    const tileW = Math.max(1, Math.round(drawCellWidth * dpr))
    const tileH = Math.max(1, Math.round(drawCellHeight * dpr))

    const changed =
      layout.widthCss !== w ||
      layout.heightCss !== h ||
      layout.dpr !== dpr ||
      layout.cols !== cols ||
      layout.rows !== rows

    layout = {
      widthCss: w,
      heightCss: h,
      cols,
      rows,
      dpr,
      tileW,
      tileH,
      cellWidth,
      cellHeight,
      drawCellWidth,
      drawCellHeight,
      padX,
      padY,
      spaceIndex,
    }

    if (!changed) return

    // Resize canvas
    canvas.width = Math.max(1, Math.round(w * dpr))
    canvas.height = Math.max(1, Math.round(h * dpr))
    canvas.style.width = `${w}px`
    canvas.style.height = `${h}px`
    mainCtx.setTransform(canvas.width / w, 0, 0, canvas.height / h, 0, 0)
    mainCtx.imageSmoothingEnabled = false

    // Resize buffers
    const size = cols * rows
    maskArray = new Float32Array(size)
    smoothedLuma = new Float32Array(size)

    // Rebuild atlas at new tile size
    rebuildAtlas()

    // Resize or create fluid grid
    if (fluidEnabled) {
      const fw = Math.max(1, Math.round(cols * fluidConfig.resolutionScale))
      const fh = Math.max(1, Math.round(rows * fluidConfig.resolutionScale))
      if (!fluidGrid || fluidGrid.width !== fw || fluidGrid.height !== fh) {
        fluidGrid = createFluidGrid(fw, fh)
      }
    }
  }

  // ── Ensure fluid grid exists ────────────────────────────────────────────
  function ensureFluidGrid(): FluidGrid | null {
    if (!fluidEnabled || !layout.cols || !layout.rows) return null
    if (fluidGrid) return fluidGrid
    const fw = Math.max(1, Math.round(layout.cols * fluidConfig.resolutionScale))
    const fh = Math.max(1, Math.round(layout.rows * fluidConfig.resolutionScale))
    fluidGrid = createFluidGrid(fw, fh)
    return fluidGrid
  }

  // ── Project fluid density → mask array ──────────────────────────────────
  function projectFluidToMask() {
    if (!fluidGrid || !maskArray) return
    const { width: fw, height: fh, density } = fluidGrid
    const { cols, rows } = layout
    if (!cols || !rows) return
    const sx = cols > 1 ? (fw - 1) / (cols - 1) : 0
    const sy = rows > 1 ? (fh - 1) / (rows - 1) : 0
    for (let r = 0; r < rows; r++) {
      const fy = r * sy
      const off = r * cols
      for (let c = 0; c < cols; c++) {
        maskArray[off + c] = clamp01(bilinear(density, c * sx, fy, fw, fh))
      }
    }
  }

  // ── Process splash rings ────────────────────────────────────────────────
  function processSplashRings(now: number, dtMs: number) {
    const grid = ensureFluidGrid()
    if (!grid || !layout.cols || !layout.rows) return
    const scaleX = grid.width > 1 ? grid.width / layout.widthCss : 0
    const scaleY = grid.height > 1 ? grid.height / layout.heightCss : 0
    if (scaleX <= 0 || scaleY <= 0) return
    const dtScale = clamp(dtMs / 16.67, 0.5, 2)

    let writeIdx = 0
    for (let i = 0; i < splashRings.length; i++) {
      const ring = splashRings[i]
      const elapsed = now - ring.start
      if (elapsed < 0) {
        splashRings[writeIdx++] = ring
        continue
      }
      const maxR = Math.max(1, ring.maxRadiusPx)
      const speed = 240 * fluidConfig.splashVelocityScale
      if (speed <= 0) continue
      const duration = (maxR / speed) * 1000
      if (elapsed > duration) continue

      const t = Math.min(1, elapsed / duration)
      const eased = 1 - Math.pow(1 - t, fluidConfig.splashTravelEasePower)
      const currentRadius = Math.max(0, eased * maxR)
      const decay = Math.max(0, 1 - t)
      const forceMag =
        fluidConfig.forceScale *
        fluidConfig.splashForceScale *
        fluidConfig.splashVelocityScale *
        Math.pow(decay, fluidConfig.splashForceDecayPower) *
        20
      const densityMag =
        fluidConfig.splashDensity * Math.pow(decay, fluidConfig.splashDensityDecayPower) * dtScale

      if (forceMag > 0 || densityMag > 0) {
        applySplashForces(
          grid,
          ring,
          currentRadius,
          forceMag,
          densityMag,
          ring.thicknessPx,
          fluidConfig.splashRandomness,
          scaleX,
          scaleY,
        )
      }
      splashRings[writeIdx++] = ring
    }
    splashRings.length = writeIdx
  }

  // ── Main render ─────────────────────────────────────────────────────────
  function render(timestamp: number) {
    if (destroyed) return
    const { cols, rows } = layout
    if (!cols || !rows) return

    const dtMs = lastRenderTime != null ? timestamp - lastRenderTime : 0
    const dtSec = dtMs / 1000

    // Step fluid sim
    if (fluidEnabled) {
      const grid = ensureFluidGrid()
      if (grid) {
        processSplashRings(timestamp, dtMs)
        fluidStep(grid, dtSec, fluidConfig)
        projectFluidToMask()
      }
    }

    // Render characters
    mainCtx.clearRect(0, 0, layout.widthCss, layout.heightCss)
    mainCtx.globalAlpha = 1

    const baseLuma = props.lumaValue
    const smooth = props.lumaSmoothingMs
    const useSmoothing = smooth > 0 && smoothedLuma != null
    const smoothFactor =
      useSmoothing && dtMs > 0 && smooth > 0 ? clamp01(1 - Math.exp(-dtMs / smooth)) : 0
    const firstFrame = lastRenderTime == null

    for (let row = 0; row < rows; row++) {
      for (let col = 0; col < cols; col++) {
        const i = row * cols + col
        let luma = baseLuma

        // Apply luma smoothing
        if (useSmoothing && smoothedLuma) {
          if (firstFrame) {
            smoothedLuma[i] = luma
          } else {
            smoothedLuma[i] += (luma - smoothedLuma[i]) * smoothFactor
          }
          luma = smoothedLuma[i]
        }

        // Apply fluid mask
        const mask = fluidEnabled && maskArray ? maskArray[i] : 1
        if (mask <= 0 && props.invertLuma) continue
        if (props.invertLuma) {
          luma *= mask
        } else {
          luma = luma * mask + (1 - mask)
        }

        const charIdx = lumaToChar(luma)
        if (charIdx === spaceIndex && spaceIndex >= 0) continue

        mainCtx.drawImage(
          atlasCanvas,
          charIdx * layout.tileW,
          0,
          layout.tileW,
          layout.tileH,
          col * cellWidth + padX,
          row * cellHeight + padY,
          drawCellWidth,
          drawCellHeight,
        )
      }
    }

    lastRenderTime = timestamp
  }

  // ── Animation loop ──────────────────────────────────────────────────────
  function tick(timestamp: number) {
    if (destroyed) return
    if (fpsInterval > 0 && timestamp - lastFrameTime < fpsInterval) {
      animFrameId = requestAnimationFrame(tick)
      return
    }
    lastFrameTime = timestamp
    render(timestamp)
    animFrameId = requestAnimationFrame(tick)
  }

  function startLoop() {
    if (animFrameId != null) return
    animFrameId = requestAnimationFrame(tick)
  }

  function stopLoop() {
    if (animFrameId != null) {
      cancelAnimationFrame(animFrameId)
      animFrameId = null
    }
  }

  // ── Pointer handlers ────────────────────────────────────────────────────
  function onPointerMove(e: PointerEvent) {
    if (!fluidEnabled || reducedMotion) return
    if (e.pointerType !== 'mouse' && activePointerId !== e.pointerId) return

    const isDragging = activePointerId === e.pointerId
    const isHover = !isDragging && !props.disableHover
    const isDragMove = isDragging && !props.disableDrag
    if (!isHover && !isDragMove) {
      lastPointer = { x: e.clientX, y: e.clientY, time: e.timeStamp }
      return
    }

    const rect = container.getBoundingClientRect()
    const lx = e.clientX - rect.left
    const ly = e.clientY - rect.top
    if (lx < 0 || ly < 0 || lx > rect.width || ly > rect.height) {
      lastPointer = null
      return
    }

    const grid = ensureFluidGrid()
    if (!grid) return
    if (!layout.widthCss || !layout.heightCss) return

    const prev = lastPointer
    const now = e.timeStamp
    const dx = prev ? e.clientX - prev.x : e.movementX || 0
    const dy = prev ? e.clientY - prev.y : e.movementY || 0
    const dist = Math.hypot(dx, dy)
    const dt = prev ? clamp((now - prev.time) / 1000, 0.001, 0.05) : 0.016

    // Check drag threshold
    if (isDragging && pointerDownPos && !hasDragged) {
      if (Math.hypot(e.clientX - pointerDownPos.x, e.clientY - pointerDownPos.y) >= fluidConfig.dragThresholdPx) {
        hasDragged = true
      }
    }

    if (dist > 0) {
      const gsx = grid.width / layout.widthCss
      const gsy = grid.height / layout.heightCss
      const gx = (lx / layout.widthCss) * (grid.width - 1)
      const gy = (ly / layout.heightCss) * (grid.height - 1)

      let dragBoost = 0
      if (isDragging && hasDragged && fluidConfig.dragBoostScale > 0 && fluidConfig.dragBoostMaxPx > 0) {
        dragBoost = Math.min(fluidConfig.dragBoostMaxPx, (dist / dt) * fluidConfig.dragBoostScale)
      }

      const radius = (fluidConfig.hoverRadiusPx + dragBoost) * Math.min(gsx, gsy)
      const vx = (dx / dt) * gsx * fluidConfig.forceScale
      const vy = (dy / dt) * gsy * fluidConfig.forceScale

      injectForce(grid, gx, gy, vx, vy, fluidConfig.strength, radius)
    }

    lastPointer = { x: e.clientX, y: e.clientY, time: now }
  }

  function onPointerDown(e: PointerEvent) {
    if (props.disableClick && props.disableDrag) return
    if (!e.isPrimary || !fluidEnabled || reducedMotion) return

    const rect = container.getBoundingClientRect()
    const lx = e.clientX - rect.left
    const ly = e.clientY - rect.top
    if (lx < 0 || ly < 0 || lx > rect.width || ly > rect.height) return

    activePointerId = e.pointerId
    pointerDownPos = { x: e.clientX, y: e.clientY, time: e.timeStamp }
    hasDragged = false
    lastPointer = { x: e.clientX, y: e.clientY, time: e.timeStamp }
  }

  function onPointerUp(e: PointerEvent) {
    if (!e.isPrimary || activePointerId !== e.pointerId) return
    const wasClick = !hasDragged && !props.disableClick
    activePointerId = null
    pointerDownPos = null
    hasDragged = false

    if (!wasClick || !fluidEnabled || reducedMotion) return
    if (fluidConfig.splashRangePx <= 0) return

    const rect = container.getBoundingClientRect()
    const lx = e.clientX - rect.left
    const ly = e.clientY - rect.top
    if (lx < 0 || ly < 0 || lx > rect.width || ly > rect.height) return
    if (!layout.widthCss || !layout.heightCss) return

    // Max radius = distance to farthest corner
    const maxCornerDist = Math.max(
      Math.hypot(lx, ly),
      Math.hypot(lx, layout.heightCss - ly),
      Math.hypot(layout.widthCss - lx, ly),
      Math.hypot(layout.widthCss - lx, layout.heightCss - ly),
    )
    const maxRadius = Math.min(fluidConfig.splashRangePx, maxCornerDist)
    if (maxRadius <= 0) return

    const thickness =
      fluidConfig.splashThicknessPx > 0
        ? fluidConfig.splashThicknessPx
        : Math.max(8, Math.min(80, 0.08 * maxRadius))

    splashRings.push({
      normX: lx / layout.widthCss,
      normY: ly / layout.heightCss,
      start: e.timeStamp,
      seed: Math.random() * 1000,
      maxRadiusPx: maxRadius,
      thicknessPx: thickness,
    })
  }

  // ── ResizeObserver ──────────────────────────────────────────────────────
  const resizeObserver = new ResizeObserver(() => handleResize())
  resizeObserver.observe(container)

  // ── Reduced motion ──────────────────────────────────────────────────────
  let motionQuery: MediaQueryList | null = null
  if (window.matchMedia) {
    motionQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
    reducedMotion = motionQuery.matches
    const onMotionChange = () => {
      reducedMotion = motionQuery!.matches
      if (reducedMotion) stopLoop()
      else startLoop()
    }
    motionQuery.addEventListener?.('change', onMotionChange)
  }

  // ── Initialize ──────────────────────────────────────────────────────────
  handleResize()
  rebuildAtlas()
  if (!reducedMotion) startLoop()

  // ── Attach events ───────────────────────────────────────────────────────
  if (fluidEnabled) {
    window.addEventListener('pointermove', onPointerMove, { passive: true })
    window.addEventListener('pointerdown', onPointerDown, { passive: true })
    window.addEventListener('pointerup', onPointerUp, { passive: true })
    window.addEventListener('pointercancel', onPointerUp, { passive: true })
  }

  // ── Cleanup ─────────────────────────────────────────────────────────────
  return () => {
    destroyed = true
    stopLoop()
    resizeObserver.disconnect()
    if (fluidEnabled) {
      window.removeEventListener('pointermove', onPointerMove)
      window.removeEventListener('pointerdown', onPointerDown)
      window.removeEventListener('pointerup', onPointerUp)
      window.removeEventListener('pointercancel', onPointerUp)
    }
    if (motionQuery) {
      motionQuery.removeEventListener?.('change', () => {})
    }
  }
}

// Re-setup when key props change
watch(
  () => [
    props.fontSize,
    props.charWidth,
    props.cellPadding.x,
    props.cellPadding.y,
    props.fps,
    props.charset,
    props.contrast,
    props.gamma,
    props.invertLuma,
    props.color,
    props.fontFamily,
    props.fluid?.enabled,
    props.disableHover,
    props.disableClick,
    props.disableDrag,
  ],
  () => {
    cleanupFn?.()
    cleanupFn = setup()
  },
)

onMounted(() => {
  cleanupFn = setup()
})

onUnmounted(() => {
  cleanupFn?.()
  cleanupFn = null
})
</script>

<template>
  <div
    ref="containerRef"
    class="ascii-overlay"
    :style="{ backgroundColor }"
  >
    <canvas
      ref="canvasRef"
      aria-hidden="true"
      class="ascii-overlay__canvas"
    />
  </div>
</template>

<style scoped>
.ascii-overlay {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.ascii-overlay__canvas {
  position: absolute;
  inset: 0;
  pointer-events: none;
}
</style>
