---
name: renaissance26skills
description: Build Shopify Editions-style immersive 3D backgrounds using Three.js + DepthFlow POM shader — scroll-driven wave transitions with metallic edge glow, mouse-driven parallax occlusion mapping, per-painting effects (video, breathing glow, mouse-triggered spark), bloom post-processing, floating particles, and glassmorphic UI. Based on DepthFlow (CC BY-SA 4.0). See LICENSE for details.
---

## Overview

Create immersive 3D painting backgrounds inspired by Shopify Editions Winter 2026. Each painting uses a **Parallax Occlusion Mapping (POM) shader** with an AI-generated depth map to produce real-time depth perception. Paintings transition via **scroll-driven wave wipes** with metallic edge glow. Mouse controls parallax depth. Bloom, particles, and glassmorphic UI complete the experience.

## Quick Start

A complete runnable template is included at `template.html`. To use it:

1. Copy `template.html` to your project
2. Prepare your images (JPG) and generate depth maps (DepthAnything V2)
3. Edit the `paintings` array and `sceneMap` in the CONFIG section
4. Customize the HTML sections (`data-scene` attributes must match `sceneMap` keys)
5. Open in browser — no build step needed

---

## Required Assets Per Painting

| File | Purpose | How to Generate |
|------|---------|-----------------|
| `{name}.jpg` | Original painting | Source image |
| `{name}-depth.png` | Grayscale depth map (white=near, black=far) | DepthAnything V2 on Hugging Face (free) |

That's it — **only 2 files per painting**. No manual layer separation needed.

### Generating Depth Maps

1. Go to `https://huggingface.co/spaces/depth-anything/Depth-Anything-V2`
2. Upload the painting JPG
3. Download the grayscale depth map PNG
4. Place both files in the project directory

---

## Architecture

```
Single HTML file (no build step)
├── Three.js r128 (CDN)
├── Post-processing: EffectComposer + UnrealBloomPass (CDN)
├── DepthFlow POM Shader (per painting)
│   ├── Parallax Occlusion Mapping via ray marching
│   ├── Mouse-driven depth parallax
│   ├── Depth-based atmospheric shading
│   ├── Mouse-following spotlight
│   └── Scroll-driven wave wipe with metallic edge
├── Bloom + Color Grading post-processing
├── 2D Canvas overlay
│   ├── 150 floating dust particles (mouse-following)
│   └── Mouse spotlight glow
├── Content layer (z-index: 5)
│   ├── Glassmorphic cards (backdrop-filter: blur)
│   ├── Scroll-reveal animations
│   └── Navigation
└── Grain overlay (SVG noise)
```

---

## Core Shader: DepthFlow Parallax Occlusion Mapping

Ported from [DepthFlow](https://github.com/BrokenSource/DepthFlow) (CC BY-SA 4.0, Tremeschin).

### How It Works

Unlike simple UV offset parallax, POM uses **ray marching** to find the exact surface intersection:

```
Camera ──ray──→ ╲
                  ╲  (forward pass: coarse steps)
                   ╲
        ──────────╳── depth surface
                 ╱
                ╱  (backward pass: fine refinement)
               ╱
          hit point → sample color texture here
```

1. **Forward pass** (200 iterations, coarse): march ray from camera toward surface, stop when ray enters the depth surface
2. **Backward pass** (100 iterations, fine): step back with tiny increments to find precise intersection
3. **Sample color** at the hit UV coordinate — this creates proper occlusion (foreground blocks background)

### Key Uniforms

| Uniform | Type | Purpose |
|---------|------|---------|
| `image` | sampler2D | Color texture (painting) |
| `depth` | sampler2D | Depth map (grayscale) |
| `uCameraPos` | vec3 | Mouse-driven XY offset (parallax direction) |
| `uHeight` | float | Depth intensity (0.3–0.5 recommended) |
| `uQuality` | float | Ray march quality (0.85 = good balance) |
| `uZoom` | float | Zoom level (scroll-driven) |
| `uWave` | float | Wave wipe progress (0=hidden, 1=fully revealed) |
| `uMouse` | vec2 | Mouse position for spotlight |
| `uTime` | float | Animation time |

### Shader Effects Stack

```glsl
// 1. POM ray march → find hit UV
// 2. Sample color at hit UV
// 3. Spotlight (mouse-following warm light)
// 4. Depth-based shading (near=bright, far=dark+blue)
// 5. Atmosphere tint (per-painting color mood)
// 6. Color grading (warm shift + contrast)
// 7. Vignette
// 8. Wave wipe with metallic edge glow
```

---

## Scroll-Driven Wave Transitions

### Concept

Each painting (except the first) has a **wave mask** that reveals it from bottom to top as the user scrolls into its section. The wave position is directly tied to scroll progress — stop scrolling, wave stops.

### Wave Edge

```glsl
// Ocean wave shape: layered sine waves
float wave = waveY
  + sin(vUv.x * 4.0 + uTime * 1.2) * 0.035
  + sin(vUv.x * 7.0 - uTime * 0.8) * 0.02
  + sin(vUv.x * 2.0 + uTime * 0.5) * 0.05;

// Soft mask (no hard line)
float mask = smoothstep(wave + 0.06, wave - 0.06, vUv.y);
```

### Metallic Edge Glow

Along the wave front, scattered metallic highlights using pseudo-random hash:

```glsl
float n1 = fract(sin(vUv.x * 93.7 + vUv.y * 47.3) * 43758.5);
float scatter = smoothstep(0.6, 0.9, n1) * glowZone;
// + mouse-reactive specular shimmer
```

### Render Order

Paintings stack: `painting 0 (bottom) → 1 → 2 → 3 (top)`. Each painting's wave determines how much it covers the one below. Scrolling up reverses the wave, re-revealing the painting underneath.

---

## Post-Processing

### Bloom (UnrealBloomPass)

```javascript
const bloomPass = new THREE.UnrealBloomPass(
  new THREE.Vector2(width, height),
  0.35,  // strength
  0.8,   // radius
  0.85   // threshold
);
```

Bloom strength varies per painting (e.g., Starry Night gets stronger bloom).

### Particles (2D Canvas Overlay)

150 floating dust particles with:
- Mouse-following behavior (each particle has random follow strength 0–40%)
- Particles near mouse glow brighter
- Bright particles have radial gradient glow halos
- Canvas spotlight glow at mouse position

---

## Glassmorphic UI

All content cards use dark transparent glass:

```css
.card {
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  color: #fff;
}
```

---

## Full-Screen Project Showcase

Instead of traditional cards, each project occupies a full viewport section. Text floats directly on the 3D background — no card container to block the painting.

### Layout

```
┌──────────────────────────────────────────────┐
│                                              │
│  ┃  AI PLATFORM ————————                     │
│  ●                                           │
│  ┃  Intelligent                              │
│  ┃  Assistant                                │
│  ┃                                           │
│  ┃  Description text...                      │
│  ┃                                           │
│  ┃  98.5%  ACCURACY                          │
│  ┃  ─────────────────                        │
│  ┃  [ View Project → ]                       │
│                                              │
│        (3D painting background visible)      │
└──────────────────────────────────────────────┘
```

### Decorative Elements

- **Vertical line** — CSS `::before` pseudo-element, gradient from transparent to 15% white
- **Gold label with extending line** — `::after` on label creates a fading gold line
- **Gradient stat numbers** — `background: linear-gradient(135deg, #fff 60%, var(--gold))` with `background-clip: text`
- **CTA button** — glass background + gold border on hover + outer glow

### Sliding Gold Dot

A `position: fixed` gold dot that **tracks which project is in view**:

```javascript
// JS: detect which project-screen is at viewport center
const viewMid = scrollY + innerHeight * 0.5;
projectScreens.forEach(screen => {
  const top = screen.offsetTop;
  const bottom = top + screen.offsetHeight;
  if (viewMid >= top && viewMid < bottom) activeProject = screen;
});

// Move dot to active project's decorative line midpoint
dot.style.transform = `translate(${dotX}px, ${dotY}px)`;
```

- Dot has `transition: transform .8s cubic-bezier(0.16,1,0.3,1)` for smooth sliding
- Glowing trail via `::after` radial gradient (30px)
- `box-shadow: 0 0 16px rgba(gold), 0 0 40px rgba(gold)` for double glow
- Fades out when scrolling away from Works section

### Staggered Entrance Animation

Each element animates in with increasing delay when scrolled into view:

```css
.project-label  { transition-delay: 0.1s; transform: translateX(-20px); }
.project-title  { transition-delay: 0.2s; transform: translateY(30px);  }
.project-desc   { transition-delay: 0.35s; transform: translateY(20px); }
.project-stat   { transition-delay: 0.45s; transform: translateX(-30px); }
.project-link   { transition-delay: 0.55s; transform: translateY(15px); }
/* All reset to opacity:1 + transform:none when .vis is added */
```

---

## Per-Painting Special Effects

Each painting can have a unique shader effect via uniforms, activated only when that painting is visible.

### Painting 1: School of Athens — POM Depth Parallax
- Full DepthFlow POM with AI depth map
- Mouse-driven 3D parallax (strength 0.35)
- The showcase effect of the entire system

### Painting 2: Starry Night — Video Background
- MP4 video texture replaces static image (`THREE.VideoTexture`)
- Auto-loop, muted, playsInline
- POM depth disabled (`uHeight=0`, `uCameraPos=0`) to avoid distortion on moving content
- All other effects (Bloom, spotlight, particles, wave transition) still active
- Video compressed from 67MB → 6.5MB via ffmpeg (CRF 28, no audio)

### Painting 3: Botticelli — Breathing Light Band
- Horizontal light band across the shell/sea area (`uWater` uniform)
- Slow brightness pulse (sin-based, organic rhythm)
- Warm shimmer along X axis (metallic sheen on shell)
- Position and width tunable via `bandCenter` and `smoothstep` range

```glsl
float band = smoothstep(0.2, 0.0, abs(vUv.y - 0.15));
float breath = sin(uTime * 0.4) * 0.1;
float shimmer = sin(vUv.x * 8.0 + uTime * 0.6) * 0.3 + 0.7;
color.rgb += warmColor * band * shimmer * 0.45;
```

### Painting 4: Creation of Adam — Mouse-Triggered Spark
- `uSpark` uniform controls electric arc at finger contact point
- **Only activates when mouse approaches the touch point** — no constant glow
- Mouse distance to touch point (0.38, 0.56) drives intensity via `smoothstep(0.2, 0.05, dist)`
- Core glow (warm white) + outer glow (golden shimmer) + radiance spreading across painting
- When mouse moves away, spark smoothly fades to zero

```glsl
float mouseToTouch = distance(mousePos, touchPoint);
float intensity = smoothstep(0.2, 0.05, mouseToTouch);
// Core + shimmer + radiance only when intensity > 0
```

### Adding Custom Effects to New Paintings

1. Add a new uniform (e.g., `uMyEffect`) to the shader and quad creation
2. Set it to `1.0` for the target painting index in the animation loop
3. Add the effect code in the shader between depth shading and atmosphere tint
4. Use `vUv` for position, `uTime` for animation, `uMouse` for interaction

---

## Implementation Steps

### Step 1: Setup HTML

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<!-- Post-processing scripts (CopyShader, LuminosityHighPassShader, EffectComposer, RenderPass, ShaderPass, UnrealBloomPass) -->
```

### Step 2: Create POM Shader

Copy the `POM_FRAG` and `POM_VERT` shader strings from the implementation. The shader handles:
- Ray marching parallax
- Wave wipe mask
- Metallic edge glow
- Spotlight + depth shading + vignette

### Step 3: Create Quads Per Painting

```javascript
paintings.forEach((p, idx) => {
  const mat = new THREE.ShaderMaterial({
    uniforms: { image, depth, uCameraPos, uHeight, uWave, ... },
    vertexShader: POM_VERT,
    fragmentShader: POM_FRAG,
    transparent: true,
  });
  const quad = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), mat);
  quad.renderOrder = idx; // stacking order
  scene.add(quad);
});
```

### Step 4: Scroll-Driven Wave

```javascript
// Calculate wave progress per section
const progress = (scrollY + innerHeight * 0.4 - sectionStart) / (scrollRange * 0.4);
waveTarget = clamp(progress, 0, 1);
// Smooth interpolation
wave += (waveTarget - wave) * 0.04;
```

### Step 5: Mouse Parallax

```javascript
smx += (mx - smx) * 0.025; // slow, smooth follow
smy += (my - smy) * 0.025;
mat.uniforms.uCameraPos.value.set(smx * 0.55, smy * 0.55, 0);
```

### Step 6: Add Bloom + Particles

EffectComposer with UnrealBloomPass. 2D canvas overlay for floating dust particles with mouse interaction.

---

## Shopify Editions Comparison

| Aspect | Shopify | This Skill |
|--------|---------|-----------|
| Depth method | Real 3D models (TripoAI + Blender) | POM shader + AI depth map |
| Inpainting | AI-generated occluded areas | None (POM handles basic occlusion) |
| Animation | Theatre.js keyframed timeline | Scroll-driven interpolation |
| Transitions | 3D camera path through scenes | Wave wipe with metallic edge |
| Post-processing | Bloom + color overlay + DoF | Bloom + vignette + color grade |
| Particles | 3D butterflies + Rive animations | 2D floating dust with glow |
| Per-scene effects | Custom shaders per scene | Unique effect per painting (POM/video/breathing/spark) |
| Assets needed | 3D models + compressed textures | JPG + depth PNG, or MP4 video |
| Build system | React + Vite + SSR | Single HTML file, zero build |
| Cost | Paid tools + team | Free (DepthAnything V2 + Three.js) |
| Result | ~100% | ~75% of Shopify's visual quality |

---

## Tips & Gotchas

1. **Depth map quality matters** — DepthAnything V2 works well for most paintings. The better the depth map, the better the parallax.
2. **Mouse parallax range** — Keep `uCameraPos` multiplier at 0.4–0.7. Higher = more depth but shows edge artifacts.
3. **Wave speed** — `0.04` interpolation speed feels natural. Faster feels mechanical.
4. **Bloom threshold** — 0.85 prevents bloom on dark areas. Lower = more glow but washes out.
5. **Performance** — POM with 200+100 ray steps is expensive. On mobile, reduce `uQuality` to 0.3.
6. **Edge artifacts** — POM stretches pixels at depth boundaries. This is the fundamental limit without inpainting.
