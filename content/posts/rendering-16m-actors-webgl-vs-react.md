---
date: '2026-11-18T22:30:00-05:00'
title: 'Rendering 16.78 Million Live Actors: Why React Broke and How WebGL2 Took Over'
description: 'Why React and the DOM hit memory and main-thread limits at 16,777,216 live actors, how a 1-byte binary wire format and WebGL2 R8UI shader reached 1.9 ms (526 FPS), and when to choose React, Canvas 2D, or WebGL.'
cover:
    image: images/rendering-16m-actors-header.webp
    alt: "Rendering 16.78 Million Live Actors in the Browser"
    caption: "Scaling live fleet telemetry from 50 DOM nodes to 16,777,216 WebGL2 texels at 526 FPS"
tags: ["webgl", "react", "gke", "agents", "performance"]
categories: ["engineering"]
---

When we scaled our open-source [`agent-swarm`](https://github.com/moficodes/agent-swarm) stage dashboard on GKE from 50 actors (`swarm`) to 65,536 actors (`megaswarm`) and finally to **16,777,216 stateful actors** (`hyperswarm`, `64^4`), standard React and DOM rendering stopped being slow and became physically impossible inside a browser tab.

At 16.78 million actors, 262,144 gVisor sandboxes execute concurrently across 4,096 worker pods while over 16.5 million suspended actors hold memory checkpoints in Cloud Storage. Keeping that fleet view live meant updating millions of state transitions every second while letting an operator sweep a `10x` magnifier loupe across the `1x` cluster-wide view and zoom all the way down to an individual actor's sandbox card at `1024x`. Moving the fleet viewport from component trees to a byte-packed binary wire format and a single-triangle WebGL2 fragment shader brought full-fleet frame times down to **1.9 ms (~526 FPS)**.

{{< youtube id="YUJszz_ahq0" title="Rendering 16,777,216 Live Stateful Agents on GKE with WebGL2" >}}

*Full 4-minute live GKE recording ([watch directly on YouTube](https://youtu.be/YUJszz_ahq0)): 4,096 worker pods, 262,144 concurrent gVisor seats, 1x to 1024x WebGL2 semantic zoom, and a live two-phase mobile `/chat` approval turn.*

## Why React and the DOM hit a wall at 16.78 million actors

A `60 FPS` browser UI has `16.6 ms` per frame for JavaScript execution, React reconciliation, CSS style calculation, layout, paint, and compositing combined. Pushing 16,777,216 live actors through React breaks four independent browser budgets at once.

![Animated breakdown of how scaling from 50 to 65,536 to 16,777,216 actors breaks DOM virtualization, sub-pixel layout, the 16.6 ms frame budget, and the 4.0 GB browser tab heap limit](/images/rendering-16m-actors-bottleneck.gif)

### 1. Viewport virtualization cannot save a macro overview

The standard React answer for large datasets is windowing (`react-window` or TanStack Virtual): keep the full array in memory, mount only the 50 to 150 rows currently inside the scroll viewport, and recycle DOM nodes as the user scrolls.

Windowing works when the user reads a table 50 rows at a time. It fails on a live fleet observability wall where **all 16,777,216 actors sit inside the viewport at `1x` zoom**. The whole point of the macro view is spotting bursts of waking, thinking, awaiting-approval, executing, or crashed sandboxes across the entire population at a glance. Once all 16.78 million items are on screen simultaneously, virtualization prunes zero nodes.

### 2. Fiber and DOM memory blow past the tab heap limit

In React, every rendered actor is at least one Fiber object (often two during double-buffered `current` and `workInProgress` reconciliation), a props object, and a backing `HTMLElement` plus layout structures in Blink. Even a stripped-down `<div>` badge with a React Fiber costs roughly `400 to 1,000 bytes` across V8 and Blink heaps.

Multiplying `16,777,216` actors by even `500 bytes` requires **more than 8.3 GB of RAM** before storing a single byte of application telemetry. Chromium tabs hit V8 heap limits around `4 GB` and terminate with an out-of-memory crash long before the initial mount completes.

### 3. Single-threaded CPU reconciliation and JSON GC churn

Even if memory were infinite, updating 16.78 million actors over JSON and React runs `O(N)` work on one main-thread CPU core. Sending 16.78 million JSON actor records at ~80 bytes each ships **~1.3 GB per poll tick**, allocates tens of millions of short-lived strings during `JSON.parse()`, and stalls the main thread in V8 garbage collection. After parsing, diffing 16,777,216 Fiber nodes at an optimistic `20 nanoseconds` per node burns **~335 ms of pure JavaScript CPU time** per update, 20x the entire `16.6 ms` frame budget, before the browser touches style recalculation or rasterization.

### 4. Sub-pixel aliasing when actors outnumber screen pixels

A `1920 x 1080` display has `2,073,600` pixels total, and the fleet pane itself occupies roughly `800,000 to 3,000,000` device pixels depending on DPI. At `1x` zoom, **5 to 20 actors map onto the same physical screen pixel**.

DOM layout engines have no way to express domain-aware pixel reduction. Sub-pixel CSS boxes either round away to `0 px` or alpha-blend into muddy slate, hiding the 262,144 active sandboxes underneath the 98% majority of suspended Cloud Storage checkpoints.

![Comparison of Regular React + JSON O(N) CPU pipeline versus Binary + WebGL2 R8UI O(Pixels) GPU pipeline](/images/rendering-16m-actors-pipelines.webp)

## How WebGL2 unlocked real-time 16.78M-actor updates

Replacing React for the fleet matrix meant treating actor state as a **dense byte texture** from the Go backend all the way to the GPU fragment stage, implemented in [`board.go`](https://github.com/moficodes/agent-swarm/blob/main/cmd/conf-gateway/internal/actorstate/board.go#L1066-L1172), [`telemetry.go`](https://github.com/moficodes/agent-swarm/blob/main/cmd/conf-gateway/internal/gateway/telemetry.go#L75-L99), and [`stage.js`](https://github.com/moficodes/agent-swarm/blob/main/cmd/conf-gateway/static/stage.js#L1629-L1886).

![Cropped live capture of the 16,777,216-actor (4,096 x 4,096) Fleet Matrix sweeping the 10x loupe at 1x macro view and zooming from 1x to 1024x inspector cards](/images/rendering-16m-actors-fleet-crop.gif)

### 1. One byte per actor on the backend and over the wire

Instead of serializing JSON objects for idle actors, the gateway maintains a flat `16 MiB` byte slice (`1 byte per actor`) alongside atomic `uint32` bitsets. Each byte packs an actor's complete visual state:

* **Bits `0..3`.** Lifecycle state (`0` = never started, `1` = suspended in Cloud Storage, `2` = resuming, `3` = suspending, `4` = running, `5` = thinking, `6` = awaiting approval, `7` = executing, `8` = crashed).
* **Bit `4` (`16`).** Active live QR attendee lease.
* **Bit `5` (`32`).** Currently seated on a physical worker pod.

`GET /api/stage/fleet.bin` writes a 24-byte binary header (`"SWFL"` magic, format version, actor count, and precomputed `O(1)` active, snapshotted, and leased totals) followed by those `16,777,216` raw bytes, streamed through pooled `gzip.BestSpeed` writers. In the browser, `await resp.arrayBuffer()` and `new Uint8Array(ab, 24, actorCount)` produce a zero-copy typed array view with zero `JSON.parse()` calls, zero string allocations, and zero GC pauses.

### 2. Mapping 16,777,216 actors to one `4,096 x 4,096` integer texture

`16,777,216` (`64^4`) factors into **`4,096 x 4,096`** (`2^12 x 2^12`), which fits inside WebGL2's guaranteed `MAX_TEXTURE_SIZE` across desktop and mobile GPUs.

Rather than building 16.78 million vertex quads or instanced meshes, the client uploads the `Uint8Array` directly into a `gl.R8UI` unsigned-integer 2D texture (`1 texel = 1 actor`) via `gl.texSubImage2D` only when a new poll arrives (`fleetTextureDirty = true`). During interactive `60 to 120 Hz` pan and zoom, **zero bytes cross the CPU-to-GPU bus**: the renderer updates two `vec2`/`float` uniforms (`u_pan`, `u_zoom`) and issues a single vertex-buffer-free draw call of one oversized full-screen triangle (`gl.drawArrays(gl.TRIANGLES, 0, 3)`).

### 3. Turning `O(Actors)` CPU loops into `O(Pixels)` GPU compute

A CPU loop scales with the `16,777,216` actors in the dataset. A fragment shader scales with the `~1 to 2 million` pixels on screen and runs across thousands of GPU cores in parallel.

That GPU compute budget let us solve the sub-pixel visibility problem directly inside GLSL ([`stage.js`](https://github.com/moficodes/agent-swarm/blob/main/cmd/conf-gateway/static/stage.js#L1718-L1753)). When zoomed out (`cellPx < 1.05`), each screen pixel's fragment shader samples up to an `8 x 8` (`64`-texel) block of underlying actors via `texelFetch` and runs a priority reduction:

1. Live QR attendee leases (`priority 90`) and crashed sandboxes (`priority 85`) win unconditionally so operators never miss a human session or a fault.
2. Active sandbox states (`Resuming`, `Thinking`, `Awaiting`, `Executing`, `Running`, `Suspending`) share equal priority (`priority 70`) and break ties with one-pass deterministic integer hash reservoir sampling keyed on `gl_FragCoord.xy`. Every active state stays proportionally visible side-by-side without frame-to-frame flicker, instead of one arbitrary state masking the others.
3. Suspended Cloud Storage checkpoints (`priority 20`) and never-started slots (`priority 0`) fill the background only when no active actor maps to that pixel.

At intermediate zoom (`16x` to `256x`), the same shader procedurally computes inter-tile borders, white center pips for pod-seated actors (`bit 5`), and corner markers for QR-reservable slots (`idx % stride == 0`) with zero extra geometry.

### 4. Hybrid semantic zoom: handing back to Canvas 2D and DOM when `N` gets small

WebGL excels at millions of colored glyphs and pixels, while DOM and Canvas 2D excel at typography and rich interactive controls. Instead of building an all-WebGL application, we stacked three layers by visible cardinality:

![Three-layer hybrid semantic zoom architecture from WebGL2 R8UI shader at 1x-256x to 2D Canvas inspector cards at 512x-1024x to HTML DOM Spotlight Drawer on click](/images/rendering-16m-actors-zoom-layers.webp)

Once an operator zooms to `512x` or `1024x`, only `16 to 64` actors remain in view. At that threshold, `#fleet-webgl-canvas` hides and the stacked 2D `<canvas>` (`#fleet-canvas`) draws crisp monospace cards with actor IDs, pod seats, and `gcs://` snapshot paths. Clicking any cell maps `(clientX, clientY)` back through `(u_pan, u_zoom)` to `actorIndex = row * 4096 + col + 1` in `O(1)` math and opens a standard HTML/CSS **Spotlight Drawer** (`GET /api/actor/{id}`) with selectable logs, syntax-highlighted Python code, and keyboard navigation.

## Pipeline comparison across scale tiers

Putting Regular React, a CPU `ImageData` Canvas 2D blit, and the WebGL2 `R8UI` shader side-by-side on the same `16,777,216`-actor stream exposes both the frame-time gap and the sub-pixel sampling gap. A CPU `ImageData` fallback survives in `24 MB` of RAM without crashing the tab, yet single-sample point lookups (`floor(uv * 4096)`) skip more than 98% of active sandboxes at `1x` zoom and shimmer during pans because checking 64 texels per pixel on one CPU core would drop frame rates into single digits. Offloading that `8 x 8` reduction to the GPU keeps every active state visible at **1.9 ms per frame**.

![Side-by-side live comparison of Regular React (DOM), HTML5 Canvas 2D (CPU ImageData), and WebGL2 (R8UI integer texture and 8x8 fragment shader) on the same 16,777,216-actor stream across 1x macro pan and 48x zoom](/images/rendering-16m-actors-comparison.gif)

| Dimension | React / DOM (`<div>` per actor) | Canvas 2D (`ImageData` / `fillRect`) | WebGL2 (`R8UI` Texture + Hybrid Shell) |
| :--- | :--- | :--- | :--- |
| **Practical live limit** | ~1k to 3k mounted nodes | ~10k to 100k items (`65k` grid) | **16.78M+ items** (`4,096 x 4,096`+) |
| **Memory for 16.78M** | >8 GB (tab OOM crash) | ~16 MB data + ~8 MB RGBA buffer | **16 MB RAM + 16 MB GPU VRAM** |
| **Update scaling** | `O(N)` single-threaded CPU | `O(Pixels)` CPU (1 sample/px) | **`O(Pixels)` parallel GPU (64 samples/px)** |
| **16.78M frame time** | Fails to mount / multi-second hang | ~18 to 45 ms on CPU fallback | **1.9 ms (~526 FPS)** |
| **1x sub-pixel fidelity** | Washes out or crashes | Drops ~98% of active states & shimmers | **Preserves all active states cleanly** |
| **Typography & forms** | Native CSS, fonts, inputs | Manual `fillText()`, no inputs | Poor (needs SDF fonts or DOM overlay) |
| **Accessibility & search** | Built-in a11y tree, `Cmd+F` | Opaque bitmap (must build custom) | Opaque bitmap (must build custom) |

## Trade-offs of a custom WebGL pipeline versus React

Moving to WebGL2 gave us two orders of magnitude more headroom, but you pay an explicit engineering tax for leaving React's component model:

* **No layout engine or native text rendering.** Inside GLSL, there is no Flexbox, CSS Grid, line wrapping, or browser font hinting. Rendering legible multi-line text inside pure WebGL requires signed-distance-field (SDF) font atlases and glyph quad buffers, which is why handing off to a 2D Canvas / DOM layer at `512x` and `1024x` zoom saved thousands of lines of fragile text-rendering code.
* **Accessibility, `Cmd+F`, and browser primitives vanish.** To a screen reader or browser find-in-page, a `<canvas>` is a single empty element. Every navigation affordance that the DOM gives you for free (tab focus, text copy, tooltips, search) must be rebuilt explicitly via `#Actor` jump inputs, zoom preset buttons, ARIA status readouts, and an on-demand DOM detail drawer.
* **Manual coordinate math and hit-testing.** Component `onHover` and `onClick` handlers disappear. You own the inverse UV transform from CSS mouse coordinates through device-pixel-ratio (`devicePixelRatio`), pan offsets, and zoom factors, and off-by-one rounding bugs will select the wrong neighbor in a `4,096 x 4,096` grid.
* **GPU hardware constraints and fallback paths.** `MAX_TEXTURE_SIZE` dictates how you pack data (`4,096 x 4,096` works everywhere; a `16,777,216 x 1` 1D texture fails immediately). Browsers can also drop WebGL contexts when backgrounded or under VRAM pressure (`webglcontextlost`), and headless test environments sometimes lack hardware acceleration, requiring a software `Uint32Array` `ImageData` fallback ([`stage.js`](https://github.com/moficodes/agent-swarm/blob/main/cmd/conf-gateway/static/stage.js#L1888-L1913)).
* **Duplicated styling and steeper debugging.** CSS variables (`var(--bg)`) do not cascade into compiled GLSL shaders; Light and Dark palettes have to stay synchronized between CSS and shader lookup tables. When a badge renders the wrong color, you debug bitmasks, `UNPACK_ALIGNMENT = 1` byte packing, and `highp usampler2D` integer precision instead of opening React DevTools.

## When to choose React, Canvas 2D, or WebGL

Rendering pipelines work best as an escalation ladder matched to how many items must appear on screen at the same time:

1. **Default to React (and DOM virtualization)** whenever the number of *simultaneously mounted* elements on screen stays **under ~1,000 to 2,000**. Even if your dataset has 50 million records, stay in React if users browse it through paginated or virtualized tables, text-heavy cards, forms, or workflows where only 100 rows are visible at once. The development speed, accessibility, typography, and maintainability of React outweigh raw pixel throughput here.
2. **Step down to HTML5 Canvas 2D** when **`5,000 to ~100,000` items** must appear in the same viewport simultaneously (such as our `65,536`-actor `512 x 128` grid or the `256 x 64` minimap), updates run at moderate rates, and direct `fillRect` / `fillText` calls or a CPU `Uint32Array` `ImageData` blit stay comfortably under `8 ms` without shader complexity.
3. **Reach for WebGL2 (inside a React/DOM shell)** when **all `100,000 to 10,000,000+` items must be visible at once**, state mutates continuously (`1 to 60 Hz`), or `N` exceeds physical viewport pixels so you need programmable per-pixel aggregation on the GPU. Pair WebGL with a binary wire format (`ArrayBuffer` / `Uint8Array`), since a `2 ms` GPU shader cannot rescue a UI that still spends `800 ms` parsing JSON on the main thread, and keep your controls, navigation, and single-item inspectors in regular React/DOM where component trees shine.
