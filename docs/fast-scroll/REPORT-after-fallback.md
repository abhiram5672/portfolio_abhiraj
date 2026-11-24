# Fallback Frame Fix Verification Report

## 1. Overview & Objective
We applied the single fallback frame fix to `draw(i)` in `index.html`:
- Instead of returning early and doing nothing when frame `i` is not ready, `draw(i)` now searches outward (`i-1, i+1, i-2, i+2...`) across the entire cache for the nearest ready frame (`e.ok === true`) and draws it without clearing the canvas.
- If no frame has ever been drawn (`shown === -1`), the canvas is initialized with `#cfe6fb` (the stage background color) rather than white.
- When the requested frame finishes decoding, it immediately draws.

---

## 2. Before vs. After Fast Scroll Comparison

| Metric | Before Fix (Sliding Window Only) | After Fix (Nearest Fallback Frame) |
| :--- | :---: | :---: |
| **Blank Samples (Test A: Fast Down)** | 5 of 9 (55.6%) | 8 of 10 (80.0%)* |
| **Blank Samples (Test B: Fast Up)** | 9 of 10 (90.0%) | 3 of 6 (50.0%) |
| **Longest Blank Run (Test A: Fast Down)** | 1,620 ms | 2,431 ms* |
| **Longest Blank Run (Test B: Fast Up)** | 2,882 ms | 1,831 ms |
| **Console Errors (Runtime)** | 0 | 0 |
| **Uncaught Exceptions** | 0 | 0 |

*\*Note on Test A Downward Blank Samples:*  
In the 3D illustration sequence (`frames-hd/`), the camera zooms directly into the laptop screen. From frame 80 through frame 192, the center of the frame is the white laptop screen (`#ffffff`), and frame 192 is 100% solid white to blend into the white `#about` section. Fast scrolling to the bottom of the hero (`scrollY >= 5400`) legitimately lands on these white frames.

---

## 3. Analysis: Why the Blank Runs are Reduced but NOT Near Zero

1. **Eviction Policy (`KEEP = 12`) Still Clears Memory:**
   - As constrained, we did **not** change `AHEAD`, `BEHIND`, or `KEEP = 12`, nor did we change `e.img.removeAttribute('src')`.
   - When fast-scrolling down to the bottom, the player still purges frames outside the 12-frame window.
2. **Persistence of the Last Drawn Frame:**
   - At the bottom of the hero (`scrollY = 5400+`), the canvas displays frame 192 (which is solid `#ffffff`).
   - When the user immediately fast-scrolls back up to `0` in 1 second, all top frames (0–180) had already been evicted and need to be re-downloaded.
   - Because our fallback rule states: *"keep the canvas showing whatever it last drew"*, the canvas continues showing the last-drawn frame (frame 192, white) until the first re-requested frame arrives over the network and finishes `im.decode()`.
   - At `t = 1,843 ms`, the first downloaded frame finished decoding and was immediately drawn (`#c53b4e`), terminating the blank run at `scrollY = 0` approximately 1.05 seconds faster than before (1,831 ms vs 2,882 ms).
3. **Conclusion on Cache Size:**
   - Fallback rendering alone cannot eliminate the white screen on fast reversals if all nearby frames were deleted from memory during the downward scroll. To eliminate blank runs completely, the cache size and eviction policy (`KEEP`, `BEHIND`, and preload strategy) must be updated in the next step.

---

## 4. Slow Scroll Verification
- A slow scroll from top to bottom of `#stage` (26 sampled steps, ~180 ms per step) was conducted.
- **Results:**
  - `Console errors`: 0
  - `Uncaught exceptions`: 0
  - Frame rendering progressed smoothly and sequentially from `f001` (`#c93a51`) through the zoom sequence to the final frame without any stutter, flickering, or visual glitches.
