# Fast Scroll Performance Report — Before Optimization

Measured on `http://localhost:8000/index.html` at desktop viewport `1440×900` (HTTP cache disabled) via Chrome DevTools Protocol before optimizing the hero frame player and CSS.

## Summary Metrics

| Test | Blank Samples | Longest Blank Run | Frame Requests | Re-Requested Frames | Aborted Requests (`net::ERR_ABORTED`) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Test A: Fast Scroll Down** (`0 → 6000`) | 10 / 16 (62.5%) | 9,625 ms | 193 (10 initial + 183 during scroll) | — | 115 |
| **Test B: Fast Scroll Back Up** (`6000 → 0`) | 11 / 16 (68.8%) | 3,115 ms | 178 | 178 | 2 |
| **Combined (Down + Up)** | 21 / 32 (65.6%) | 9,625 ms | 371 | 178 | 117 |

## Observations

- **Test A (Fast Scroll Down):** Scrolling quickly from `scrollY = 0` to `scrollY = 6000` triggered 183 additional frame requests (193 total including initial load), with 115 requests aborted (`net::ERR_ABORTED`) as the sliding window (`KEEP = 12`) evicted in-flight images. Main-thread decode contention stalled sampling between `t = 2,283 ms` and `t = 7,047 ms`, and samples 7–16 (`t = 7,047 ms` to `t = 11,908 ms`, `scrollY = 5000–6000`) read `#ffffff` across all 5 sample points (note: frames near the end of the sequence also zoom into the white laptop screen / `#ffffff` at `f192`).
- **Test B (Fast Scroll Back Up):** Scrolling back up from `scrollY = 6000` to `scrollY = 0` re-requested all 178 frames (`f178` down to `f001`) because `KEEP = 12` had evicted earlier frames from memory. While waiting for evicted frames to re-download and decode, the canvas stayed stuck on `#ffffff` for 11 consecutive samples (`t = 0 ms` through `t = 2,847 ms`, including `scrollY = 0` from `t = 1,463 ms` to `t = 2,847 ms`), recovering to `#c3384a` at `t = 3,115 ms` (`longestRunMs = 3,115 ms`).
- **Console / Page Errors:** `0` console errors, `0` uncaught exceptions (only `favicon.ico` 404).
