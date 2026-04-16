# vue-ascii-overlay

[![npm](https://img.shields.io/npm/v/vue-ascii-overlay)](https://www.npmjs.com/package/vue-ascii-overlay)

A Vue 3 component that renders an interactive ASCII character grid with mouse-driven Navier-Stokes fluid simulation. Hover, click, and drag to disturb the fluid field — characters shift in brightness to visualize the flow.

## Install

```bash
npm install vue-ascii-overlay
```

## Usage

```vue
<script setup>
import { AsciiOverlay } from 'vue-ascii-overlay'
</script>

<template>
  <div style="width: 100vw; height: 100vh">
    <AsciiOverlay />
  </div>
</template>
```

> The component fills its parent container. Make sure the parent has explicit dimensions.

## Props

All props are optional — the component works with zero configuration.

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `fontSize` | `number` | `14` | Base font size in pixels |
| `charWidth` | `number` | `0.6` | Character width as a ratio of font size |
| `cellPadding` | `{ x: number; y: number }` | `{ x: 0, y: 0 }` | Padding between grid cells in pixels |
| `fps` | `number` | `30` | Target frames per second |
| `charset` | `string` | `` ` .:-=+*#%@`` `` | Characters used for brightness mapping (dark to light) |
| `contrast` | `number` | `1` | Contrast multiplier for luma mapping |
| `gamma` | `number` | `1` | Gamma correction exponent |
| `invertLuma` | `boolean` | `false` | Invert the brightness mapping |
| `lumaValue` | `number` | `0` | Base luma value (0–1) |
| `lumaSmoothingMs` | `number` | `0` | Smoothing time for luma transitions in milliseconds |
| `backgroundColor` | `string` | `'transparent'` | Background color (CSS color string) |
| `color` | `string` | `'#ffffff'` | Character color (CSS color string) |
| `fontFamily` | `string` | `'monospace'` | Font family for rendered characters |
| `fluid` | `AsciiOverlayFluidConfig` | See below | Fluid simulation configuration |
| `disableHover` | `boolean` | `false` | Disable hover interaction |
| `disableClick` | `boolean` | `false` | Disable click splash effect |
| `disableDrag` | `boolean` | `false` | Disable drag interaction |

## Fluid Configuration

Pass a `fluid` prop to tune the Navier-Stokes simulation:

```vue
<AsciiOverlay
  :fluid="{
    enabled: true,
    forceScale: 50,
    hoverRadiusPx: 80,
    splashRangePx: 150,
  }"
/>
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enabled` | `boolean` | `true` | Enable/disable fluid simulation |
| `resolutionScale` | `number` | `1` | Grid resolution multiplier |
| `velocityDissipation` | `number` | `0.98` | How quickly velocity decays (0–1) |
| `densityDissipation` | `number` | `0.97` | How quickly density decays (0–1) |
| `diffusion` | `number` | `0.0001` | Diffusion rate |
| `iterations` | `number` | `4` | Solver iterations per step |
| `forceScale` | `number` | `50` | Force multiplier for hover interaction |
| `hoverRadiusPx` | `number` | `60` | Hover influence radius in pixels |
| `strength` | `number` | `1` | Global force strength multiplier |
| `splashRangePx` | `number` | `120` | Click splash radius in pixels |
| `splashVelocityScale` | `number` | `1` | Velocity scale for splash rings |
| `splashForceScale` | `number` | `1` | Force scale for splash rings |
| `splashDensity` | `number` | `1` | Density injected by splash |
| `splashRandomness` | `number` | `0.3` | Randomness in splash ring shape |
| `splashThicknessPx` | `number` | `20` | Thickness of splash ring in pixels |
| `splashTravelEasePower` | `number` | `2` | Easing power for splash travel |
| `splashForceDecayPower` | `number` | `2` | Decay power for splash force |
| `splashDensityDecayPower` | `number` | `2` | Decay power for splash density |
| `dragBoostScale` | `number` | `1` | Force boost multiplier during drag |
| `dragBoostMaxPx` | `number` | `200` | Max drag distance for boost calculation |
| `dragThresholdPx` | `number` | `3` | Minimum drag distance to trigger boost |
| `project` | `boolean` | `true` | Enable pressure projection (mass conservation) |
| `projectIterations` | `number` | `4` | Pressure solver iterations |

## TypeScript

Types are exported for use in your own code:

```ts
import type { AsciiOverlayProps, AsciiOverlayFluidConfig } from 'vue-ascii-overlay'
```

## Requirements

- Vue 3.3+
- Modern browser with Canvas 2D support

## License

MIT
