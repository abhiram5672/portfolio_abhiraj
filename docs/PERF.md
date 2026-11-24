# Performance Baseline Audit

Measured locally against static server at `http://localhost:8000` using Google Lighthouse.

## Core Web Vitals & Performance Summary (Baseline)

| Metric | Mobile (Before) | Desktop (Before) |
|---|---|---|
| **Performance Score** | **45 / 100** | **94 / 100** |
| **LCP (Largest Contentful Paint)** | **4.9 s** | **1.5 s** |
| **CLS (Cumulative Layout Shift)** | **0** | **0.003** |
| **TBT (Total Blocking Time)** | **4,200 ms** | **0 ms** |
| **INP (Interaction to Next Paint)** | **N/A** (no user interaction in synthetic run) | **N/A** (no user interaction in synthetic run) |
| **FCP (First Contentful Paint)** | 1.9 s | 0.8 s |
| **Speed Index** | 6.5 s | 0.9 s |

### Reports Generated
- Mobile: [./docs/perf/before/mobile.html](./docs/perf/before/mobile.html)
- Desktop: [./docs/perf/before/desktop.html](./docs/perf/before/desktop.html)

### Observations
- **Mobile Performance bottleneck**: Significant Total Blocking Time (4,200 ms) and delayed LCP (4.9 s) driven by synthetic CPU throttling, parsing of the 132 KB inline HTML document (containing inline base64 WebP images and triple-duplicated SVGs), canvas frame preloading/decoding, and unthrottled layout queries.
- **Desktop Performance**: Strong baseline (94), with zero Total Blocking Time on unthrottled desktop CPU.
