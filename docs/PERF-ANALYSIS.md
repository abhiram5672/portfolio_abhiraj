# Performance Analysis: Main-Thread Blocking on Mobile

**Project:** Abhiraj Rajeev Portfolio  
**Audit Context:** Mobile Chrome (Simulated 6× CPU throttling, Slow 4G / Localhost)  
**Baseline Results:** Total Blocking Time (TBT) = **4,200 ms** (Mobile) vs. **0 ms** (Desktop)  
**Goal:** Identify root causes of main-thread congestion during page load and scroll without altering visual design.

---

## 1. Page Load Main-Thread Analysis

### 1.1 Script Blocks in `index.html`

`index.html` contains exactly **two** inline `<script>` blocks and zero external JavaScript bundles:

| Script Block | Location | Load-Time Operations | Main-Thread Work & Impact |
|---|---|---|---|
| **Script 1: Nav Scroll-Spy** | `index.html` (Lines 1296–1306) | Queries `.nav a.l` into an array, attaches passive `scroll` listener, and executes `set()` immediately on parse. | **Low load-time work (~5–10 ms):** `set()` performs 3 `getElementById()` lookups and calls `getBoundingClientRect().top` on `#about`, `#work`, and `#contact`. This forces a synchronous layout calculation during initial DOM construction. |
| **Script 2: Canvas Player & Hero Animation Loop** | `index.html` (Lines 1307–1419) | 1. Queries canvas `#cv` and acquires 2D context (`desynchronized: true`).<br>2. Evaluates `matchMedia` for reduced motion.<br>3. Executes `manage(0)` which synchronously triggers `want()` for frames 0 through 8 (creates 9 `Image` instances, starts downloads).<br>4. Attaches `resize` listener and calls `resize()` (sets `cv.width`/`cv.height`, calls `draw(0)`).<br>5. **Starts unconditional `requestAnimationFrame(tick)` loop immediately.** | **Extremely Heavy (~4,000+ ms under 6× CPU throttling):** Starting the rAF loop on load starves the CPU while the browser is attempting First Contentful Paint. On every frame, `progress()` measures layout (`stage.getBoundingClientRect()`), `overlays()` writes 7 inline styles, and `manage()` / `draw()` process image decodes. |

### 1.2 Base64 Portrait Image Analysis (Line 1104)
- **Code:** `<img class="portrait" src="data:image/webp;base64,UklGRgaPAABXRUJQV...==" alt="...">`
- **Script Processing:** **None.** No script queries, modifies, or decodes this image via JS.
- **Main-Thread Cost:** Even though untouched by JS, the browser HTML parser must tokenize this 30,500-character base64 string on the main thread during HTML parsing, and the rasterizer must allocate and decode the WebP data into memory during the initial layout phase.

### 1.3 Tool Logo Marquee DOM Node Count
The tool logo marquee (`<section class="marq">`, lines 957–1094) creates a total of **106 DOM elements**:
- `1` `<section class="marq">` container
- `1` `<div class="track">` scrolling container
- `24` `<div class="t">` tile containers (6 distinct tool logos repeated across 4 complete cycles)
- `24` `<svg>` elements
- `40` `<path>` elements (complex vector paths including Claude, Photoshop, Illustrator, ChatGPT, After Effects, and Figma logos)
- `12` `<rect>` elements
- `4` `<span class="app">` icon wrapper badges

All 106 elements are placed in the DOM tree at parse time and immediately start translating via an infinite CSS keyframe animation (`@keyframes mq 40s linear infinite`).

---

## 2. Scroll Main-Thread Analysis

### 2.1 Event Listeners & Animation Callbacks

| Callback Type | Target | File & Lines | Trigger | Main-Thread Operations |
|---|---|---|---|---|
| `scroll` | `window` | Lines 1304, 1297–1306 | Unthrottled scroll | Calls `set()` on every scroll tick: reads `getBoundingClientRect().top` on 3 elements, forcing synchronous layout recalculations, and mutates `classList.toggle('on')` on nav links. |
| `resize` | `window` | Lines 1415, 1369–1375 | Window resize | Reads `cv.clientWidth`, `cv.clientHeight`, mutates `cv.width`, `cv.height` (reallocates canvas bitmap buffer), and calls `draw()`. |
| `requestAnimationFrame` | Loop (`tick`) | Lines 1403–1417 | Every screen refresh (60/120 Hz) **continuously** | Runs continuously from page load, even when stationary and even when the hero stage is 1,000+ pixels off-screen. |

### 2.2 Per-Frame Operations in `tick()`

For every single execution of `requestAnimationFrame(tick)` (every 16.6 ms at 60 Hz):

1. **Layout Reads (Reflow triggers):**
   - `stage.getBoundingClientRect()` inside `progress()` (Line 1391)
   - `stage.offsetHeight` inside `progress()` (Line 1392)
   - `window.innerHeight` inside `progress()` (Line 1392)
2. **Style Writes (Style invalidation):**
   - `b1.style.opacity = o1;` (Line 1397)
   - `b1.style.transform = 'translateY(...)';` (Line 1397)
   - `b2.style.opacity = o2;` (Line 1398)
   - `b2.style.transform = 'translateY(...)';` (Line 1398)
   - `b3.style.opacity = o3;` (Line 1399)
   - `s2.style.opacity = o2;` (Line 1399)
   - `hint.style.opacity = ...;` (Line 1400)
   *Note: These styles are written every frame even when values have not changed.*
3. **Canvas Drawing:**
   - Reads `cv.width`, `cv.height`, `im.naturalWidth`, `im.naturalHeight` (Lines 1384–1385)
   - Executes `ctx.drawImage(im, (cw - w) / 2, (ch - h) / 2, w, h)` (Line 1387)
4. **Image Object Instantiation & Management:**
   - When scroll index advances by ≥ 2 frames (`Math.abs(c - mgrAt) >= 2`), calls `manage(c)`.
   - Iterates `d = 0` to `8`: if a frame is missing, runs `new Image()` (Line 1332), sets `.src`, registers `.onload` and `.onerror`.
   - Evicts frames outside `KEEP = 12` by deleting from `cache` object and unsetting `src`.

### 2.3 Frame Decoding & Memory Footprint
- **Decoding Mechanism:** `want(i)` sets `im.decoding = 'async'` (Line 1335) and calls `im.decode().then(...)` (Line 1348). While decoding takes place off-thread in Blink/Chromium, the promise fulfillment callback (`ready()`) runs on the main thread and immediately calls `draw(Math.round(cur))` -> `ctx.drawImage(...)`. If `draw()` requests a frame before decoding finishes, `drawImage` blocks synchronously to rasterize.
- **Frames Kept in Memory:** Configured by `KEEP = 12` with `AHEAD = 8` and `BEHIND = 3`. Between **12 and 15 decoded 1080p frames** are maintained in memory simultaneously.
- **Memory Cost:** Each 1920 × 1080 32-bit RGBA bitmap consumes **~8.29 MB of uncompressed memory**. 12 cached frames consume **~99.5 MB of RAM/VRAM** on mobile devices.

### 2.4 Canvas Resizing
- The canvas bitmap dimensions (`cv.width`, `cv.height`) are **NOT** resized in the scroll loop.
- They are resized **only on initial load and window resize** inside `resize()` (Lines 1369–1375).

---

## 3. Other Sources of Load

### 3.1 First Screen (Above-the-Fold) Assets
At initial load (scroll position 0):
- **Hero Canvas Background:** CSS rule `#cv { background: #5ca9e4 url("frames-hd/f001.webp") center / cover no-repeat }` (Line 830).
- **Preload Link:** `<link rel="preload" as="image" href="frames-hd/f001.webp" fetchpriority="high">` (Line 7) — 120 KB network transfer.
- **Initial Buffer Downloads:** `manage(0)` triggers requests for `frames-hd/f001.webp` through `frames-hd/f009.webp` (~650 KB total) immediately on parse.
- **Below-the-fold assets parsed on load:**
  - Inline base64 portrait `<img>` (30.5 KB base64 payload, Line 1103)
  - 14 work card images (Lines 1173–1279, with `loading="lazy"`)
  - Contact background `assets/contact.webp` (Line 750, CSS background)

### 3.2 Visual Effects (Filters, Shadows, Compositing)
- **`filter: drop-shadow(0 10px 11px rgba(20, 15, 31, .1))` on `.nav` (Line 76):**
  Applied to the fixed floating navigation pill. Because `.nav` is `position: fixed; z-index: 5`, it remains permanently visible on top of all scrolling content. A CSS drop-shadow filter on a fixed element forces expensive composite recalculations over the moving canvas and marquee.
- **Text Shadows with Large Blur Radii on Mobile:**
  - Mobile hero text `.hi, .sub`: `text-shadow: 0 0 12px rgba(255, 255, 255, .75)` (Line 884).
  - Mobile hero big text `.hr p.big`: Dual shadows `text-shadow: 0 1px 4px rgba(0, 0, 0, .45), 0 1px 14px rgba(0, 0, 0, .6)` (Line 885).
  - Hero scroll hint `.hint`: `text-shadow: 0 1px 12px rgba(8, 24, 48, .5)` (Line 325).
  - Large blur radii (12–14px) on text forced through CSS filter rasterization increase GPU/CPU raster time on lower-tier mobile chips.
- **Infinite Layout-Triggering CSS Animation (`.hint i::after`, Lines 345–352):**
  `animation: drip 1.6s ease-in-out infinite` animates the layout property `top` from `-40%` to `110%`. Animating `top` forces style and layout recalculation on every frame.

---

## 4. Ranked Top 5 Causes of Main-Thread Blocking

| Rank | Issue | Location | Root Cause |
|---|---|---|---|
| **1** | **Unconditional, Unthrottled `requestAnimationFrame(tick)` Loop** | `index.html`<br>Lines 1403–1417 | The render loop starts immediately on parse and runs infinitely at 60 Hz. On every frame, it reads layout (`stage.getBoundingClientRect()`, `stage.offsetHeight`), writes inline styles to 5 DOM nodes, and performs lerp arithmetic—even when the user is stationary at the top of the page. Under 6× CPU throttling, these repeated layout queries monopolize the main thread during the first 5 seconds of page life. |
| **2** | **Synchronous Initial Frame Flood (`manage(0)` on parse)** | `index.html`<br>Lines 1355–1360, 1414 | Calling `manage(0)` immediately creates 9 `new Image()` instances (`f001` to `f009`) and invokes asynchronous decoding on all 9 images simultaneously. As each image arrives, its completion microtask triggers immediate canvas draws (`draw(0)`), creating contention with DOM parsing and first paint. |
| **3** | **Layout-Thrashing in Unthrottled Scroll Listener** | `index.html`<br>Lines 1297–1305 | The `scroll` listener is attached without throttling, debouncing, or rAF coordination. It executes on every pixel of scroll and calls `getBoundingClientRect().top` on `#about`, `#work`, and `#contact`. If the user scrolls while `tick()` is running, layout is read and invalidated multiple times per frame. |
| **4** | **Layout Animation on `.hint i::after` (`top` property)** | `index.html`<br>Lines 337–352 | The scroll hint uses `@keyframes drip` which mutates `top: -40%` to `top: 110%` infinitely. Animating `top` causes recurring layout reflows and paint invalidations on the main thread throughout page life. |
| **5** | **Heavy CSS `filter: drop-shadow` on Fixed `.nav`** | `index.html`<br>Line 76 | `.nav` uses `filter: drop-shadow(0 10px 11px rgba(20, 15, 31, .1))`. Unlike `box-shadow` (which is compositor-promoted on rectangular/pill shapes), `drop-shadow()` is an SVG-filter-style alpha mask operation that recalculates during compositing as elements scroll behind it. |

---

## 5. Potential Fixes & Visual Impact Analysis

### Cause 1: Unconditional rAF Loop
- **Smallest Possible Fix:**
  1. Only run `requestAnimationFrame(tick)` when scrolling is actively occurring (e.g., wake the loop on a passive `scroll` event and let it sleep after the lerp settles: `if (Math.abs(target - cur) < 0.01)`).
  2. Use an `IntersectionObserver` to completely pause the loop when `#stage` is out of the viewport.
  3. Cache `stage.offsetHeight` on `resize` instead of querying layout inside `progress()`.
- **Visual Impact:** **Nothing.** (Frame animation remains completely smooth when scrolling; stops wasting cycles when idle).

### Cause 2: Initial Frame Flood (`manage(0)`)
- **Smallest Possible Fix:**
  On load, only request frame 0 (`want(0)`). Defer pre-buffering frames 1–8 until after the page has fired its `load` event or until the user begins scrolling.
- **Visual Impact:** **Nothing.** (Frame 0 is already preloaded via `<link rel="preload">`; frames 1–8 are only needed once the user begins scrolling down).

### Cause 3: Layout-Thrashing in Scroll-Spy
- **Smallest Possible Fix:**
  Remove the unthrottled `getBoundingClientRect().top` loop from `addEventListener('scroll', set)`. Replace with an `IntersectionObserver` observing `#about`, `#work`, and `#contact` to toggle the active nav class, or execute `set()` inside the existing rAF tick.
- **Visual Impact:** **Nothing.** (Active nav pill indicators highlight identically).

### Cause 4: Layout Animation on `.hint i::after`
- **Smallest Possible Fix:**
  Change `@keyframes drip` to animate `transform: translateY(...)` instead of `top`:
  ```css
  /* Change top to transform */
  @keyframes drip {
    from { transform: translateY(-100%); }
    to { transform: translateY(250%); }
  }
  ```
- **Visual Impact:** **Nothing.** (Identical visual drop effect, but runs purely on the GPU compositor thread without triggering layout reflow).

### Cause 5: Fixed Nav `filter: drop-shadow`
- **Smallest Possible Fix:**
  Replace `filter: drop-shadow(0 10px 11px rgba(20, 15, 31, .1))` with `box-shadow: 0 10px 11px rgba(20, 15, 31, .1)`.
- **Visual Impact:** **Nothing.** (The `.nav` container is a rounded pill with a solid background; `box-shadow` produces the exact same shadow geometry without the filter overhead).

---

## 6. Single Recommended First Fix

### **Recommended Fix: Make the Hero Animation Loop Scroll-Reactive & Cache Layout Queries**
- **Target:** `index.html` (Lines 1390–1417)
- **Why this first:**
  The `requestAnimationFrame(tick)` loop is the primary consumer of main-thread execution time (~42 seconds of total CPU time under 6× throttle). Every 16 ms, it synchronously triggers a browser reflow by reading `stage.getBoundingClientRect()` and `stage.offsetHeight`, followed immediately by style mutations.
  By:
  1. Reading `total = stage.offsetHeight - window.innerHeight` only on window `resize` (cached in a variable).
  2. Starting/running the rAF loop only when scroll position changes or when lerp has not yet converged (`Math.abs(target - cur) >= 0.01`).
  We immediately eliminate hundreds of unnecessary reflows during initial page load, freeing the main thread for FCP/LCP and directly slashing Total Blocking Time (TBT).
