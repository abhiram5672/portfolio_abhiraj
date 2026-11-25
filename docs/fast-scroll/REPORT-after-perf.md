# Fast Scroll Performance Report — After Optimization

Measured on `http://localhost:8000/index.html` at desktop viewport `1440×900` (HTTP cache disabled) via Chrome DevTools Protocol after optimizing the hero frame player and hero CSS.

## Before vs. After Comparison

| Test | Metric | Before | After |
| :--- | :--- | :---: | :---: |
| **Test A: Fast Scroll Down** (`0 → 6000`) | Blank Samples (`#ffffff` 5-pt) | 10 / 16 (62.5%) | 2 / 16 (12.5%)* |
| **Test A: Fast Scroll Down** (`0 → 6000`) | Longest Blank Run | 9,625 ms | 519 ms* |
| **Test A: Fast Scroll Down** (`0 → 6000`) | Frame Requests | 193 (115 aborted) | 193 (0 aborted) |
| **Test B: Fast Scroll Back Up** (`6000 → 0`) | Blank Samples (`#ffffff` 5-pt) | 11 / 16 (68.8%) | 4 / 16 (25.0%)* |
| **Test B: Fast Scroll Back Up** (`6000 → 0`) | Longest Blank Run | 3,115 ms | 1,121 ms* |
| **Test B: Fast Scroll Back Up** (`6000 → 0`) | Frame Requests (Re-requests) | 178 (178 re-requested, 2 aborted) | 0 (0 re-requested, 0 aborted) |
| **Combined (Down + Up)** | Total Frame Requests | 371 (117 aborted) | 193 (0 aborted) |

*\*Note on `#ffffff` samples after the fix:* In `frames-hd/`, frames ~80–192 zoom into a white laptop screen and `f192.webp` is solid `#ffffff`. After the optimization, **every** `#ffffff` sample occurred exclusively when the scroll position legitimately displayed those white frames (`scrollY = 6000` in Test A; `scrollY = 6000 → 3250` in Test B). At `scrollY = 0` on scroll-up, blank samples dropped from **6 consecutive white samples (`t = 1,463 ms`–`2,847 ms`) to 0**, and the canvas immediately rendered `#c3354a` / `#c3384a`.

## Additional Verification Checks

- **Slow Scroll (Top to Bottom, 31 steps across `0 → 5400`):** Progressed smoothly and sequentially through all scenes (`#c93a51` / `#73c1f4` at `y = 0` → `#ffffff` / `#251b13` at `y = 2700` → `#ffffff` / `#ffffff` at `y = 5400`) with zero blank flickers.
- **No Layout Reads in rAF Loop:** Instrumented `stage.getBoundingClientRect` and `stage.offsetHeight` during scroll: `0` calls inside the scroll / `requestAnimationFrame` loop (`gbcr: 0`, `offsetH: 0`).
- **Loop Lifecycle (Idle / Settled / Off-Screen):** `requestAnimationFrame` callback deltas measured `0` while idle at top, `0` after easing settled (`< 0.01`), and `0` while `#stage` was scrolled off-screen (`scrollY = 7500`).
- **Reduced Motion (`prefers-reduced-motion: reduce`):** Scrolling to `scrollY = 2700` (`p = 0.5`) jumped directly to the target frame and overlay (`b2Opacity = 1`) in `1` `requestAnimationFrame` with `0` subsequent easing frames (`rafAfter500ms = 1`).
- **Console / Page Errors:** `0` console errors, `0` uncaught exceptions.
