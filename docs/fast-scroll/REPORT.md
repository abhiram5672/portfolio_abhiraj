# Fast Scroll Performance & White Screen Audit Report

## Context & Objective
The user reported that when scrolling quickly through the hero animation (`frames-hd/`, 192 frames), the animation lags, and when scrolling back up the canvas screen can turn white.
The reported hypothesis is that the frame player's sliding window cache (`AHEAD=8, BEHIND=3, KEEP=12`) aggressively evicts frames outside the immediate window, leading to severe cache thrashing, aborted network requests, and blank canvas states.

This test was performed using Chrome DevTools Protocol at desktop viewport (1440×900) on `http://localhost:8000/`. No code was altered.

---

## 1. Test Summary Table

| Test | Frame Requests | Frame Numbers Requested | Time to Finish Loading (after last wheel) | Blank Samples | Longest Blank Run |
| :--- | :---: | :--- | :---: | :---: | :---: |
| **Test A: Fast Scroll Down** | 171 | f010–f043, f051–f062, f067–f078, f080–f091, f092–f192 | 357 ms | 5 / 9 samples (55.6%) | 1,620 ms |
| **Test B: Fast Scroll Back Up** | 178 | f178–f170, f169–f160, f159–f104, f103–f078, f077–f001 | 352 ms | 9 / 10 samples (90.0%) | 2,882 ms |

---

## 2. Memory Eviction & Frame Re-Requests
- **Were frames requested in Test A requested again in Test B?**
  **YES.**
- **Count:** **157 frames** previously downloaded in Test A were evicted from memory and re-requested from scratch in Test B.
- **Root Cause Evidence:** In `index.html` (lines 1361–1367):
  ```javascript
  Object.keys(cache).forEach(function (k) {
    if (Math.abs(Number(k) - c) > KEEP) { // KEEP = 12
      var e = cache[k]; delete cache[k];
      e.img.onload = e.img.onerror = null;
      e.img.removeAttribute('src');
    }
  });
  ```
  Because `KEEP = 12`, scrolling through the 192-frame sequence continuously purges frames that are more than 12 indices behind the scroll head. When scrolling back up, virtually the entire sequence must be re-fetched and re-decoded.

---

## 3. White Screen Timing: In Progress vs. After Finished
- **Observations:** The white screen occurred **both while requests were in progress and persisted for over 1.2 to 2.2 seconds after network requests finished**.
- **Timestamps:**
  - **Wheel input completed:** `t = 1,050 ms`
  - **Last network request finished:** `t = 1,402 ms` (`+352 ms` after last wheel event)
  - **Viewport reached scroll position 0:** `t = 1,339 ms` (Sample `up-04.png`)
  - **White screen observed (`centerHex: #ffffff`):**
    - `t = 0 ms` (`up-01.png`, scrollY: 6000)
    - `t = 255 ms` (`up-02.png`, scrollY: 4500)
    - `t = 995 ms` (`up-03.png`, scrollY: 300)
    - `t = 1,339 ms` (`up-04.png`, scrollY: 0) — *requests still in progress*
    - `t = 1,597 ms` (`up-05.png`, scrollY: 0) — *requests finished, decoding pending*
    - `t = 1,862 ms` (`up-06.png`, scrollY: 0) — *requests finished, decoding pending*
    - `t = 2,114 ms` (`up-07.png`, scrollY: 0) — *requests finished, decoding pending*
    - `t = 2,366 ms` (`up-08.png`, scrollY: 0) — *requests finished, decoding pending*
    - `t = 2,632 ms` (`up-09.png`, scrollY: 0) — *requests finished, decoding pending*
  - **Frame recovery (first visible frame restored, `#c93a51`):** `t = 3,687 ms` (`up-10.png`, scrollY: 0)
- **Explanation:** Although the network completed downloading frame images by `t = 1,402 ms`, the browser's asynchronous decode queue (`im.decode()`) was overwhelmed by 178 simultaneous image decoding tasks. In `index.html` line 1348, `e.ok = true` is only assigned once `im.decode()` resolves. Because no frame within the immediate draw window (`d <= 8`) had resolved its decode promise, `got(i)` returned `null` and `draw()` rendered nothing, leaving the canvas blank white for an additional 2,285 ms after network completion.

---

## 4. Text Overlay Behavior During Blank Periods
During the entire blank/white screen duration on scroll-up (`t = 1,339 ms` to `t = 2,632 ms`), the DOM text overlays remained visible and fully styled over the white background:
- **`#b1` (Hero Headline & Role):** Opacity = **1.0**
- **`#hint` ("Scroll to play"):** Opacity = **1.0**
- **`#b2` ("Crafting experiences..."):** Opacity = **0.0** (expected at scrollY = 0)
- **`#b3` / `#s2`:** Opacity = **0.0** (expected at scrollY = 0)

The text elements do not disappear; rather, the underlying 3D canvas illustration fails to render, leaving the dark text floating over a blank white background.

---

## 5. Console & Network Errors
- **Console errors:** **0**
- **Console warnings:** **0**
- **HTTP status errors (non-200):** **0**
- **Network request aborts (`net::ERR_ABORTED`):** **143 requests**
  - Caused directly by line 1365: `e.img.removeAttribute('src')`. When `manage()` evicts a frame whose download is still in flight, removing `src` aborts the HTTP request in Chrome.

---

## 6. Hypothesis Verdict

### **Verdict: Confirmed**

### Supporting Evidence:
1. **157 of 171 frames re-requested on scroll back up:** The sliding window (`KEEP = 12`) immediately evicts frames, forcing 157 duplicate network fetches when scrolling back up.
2. **143 network requests aborted (`net::ERR_ABORTED`):** Fast scrolling causes the sliding window to advance past in-flight requests, aborting 143 image transfers before they finish.
3. **2,882 ms continuous blank white screen on scroll up:** Because in-flight frames are aborted and evicted frames must be re-fetched and re-decoded through a bottlenecked asynchronous `im.decode()` pipeline, `draw()` finds 0 valid frames in `cache` within `[-8, +8]`, causing the screen to remain solid white (`#ffffff`) for over 2.6 seconds.
