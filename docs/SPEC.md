# Portfolio Audit — SPEC.md
**Abhiraj Rajeev — Graphic Designer & UI/UX Designer**
*Audit date: 2026-11-24 · Auditor: Antigravity AI*

---

## 1. Overview

A bespoke, single-page-application-style static portfolio built entirely in vanilla HTML/CSS/JS (no framework). The centrepiece is a **scroll-driven canvas frame-sequence animation** (192 WebP frames at 1920 × 1080) that plays like a video as the user scrolls. The site then flows into an About/skills section, two work-grid sections (Graphic Design and UI/UX), and a Contact section. Case studies open in a second HTML page (`case-study.html`) driven by a URL query-parameter and an inline JS data object — no server required.

Deployment is **purely static** (Vercel), with long-lived `Cache-Control: immutable` headers for all assets.

---

## 2. Tech Inventory

### 2.1 Files and Folders

| Path | Type | Purpose |
|---|---|---|
| `index.html` | HTML (1 422 lines, 129 KB) | Entire main page — HTML structure, all CSS (two `<style>` blocks), all JS (two `<script>` blocks) |
| `case-study.html` | HTML (70 lines, 7 KB) | Case-study viewer; content driven by `?p=<slug>` URL param |
| `vercel.json` | JSON config | Sets `Cache-Control: immutable, max-age=31536000` for `frames-hd/`, `frames-4k/`, `assets/` |
| `README.txt` | Docs | Local dev & deploy instructions |
| `DEPLOY-VERCEL.txt` | Docs | Step-by-step Vercel deploy guide |
| `assets/contact.webp` | Image | Hero background for Contact section (65 KB) |
| `assets/work-1.webp` … `work-14.webp` | Images | Thumbnail cards for work grid (43–244 KB each) |
| `assets/case/<slug>/*.webp` | Images | Full case-study imagery (84 files, 46.5 MB total) |
| `frames-hd/f001.webp` … `f192.webp` | Images | 1080p frame sequence for scroll animation (192 files, 11.5 MB total) |
| `frames-4k/f001.webp` … `f192.webp` | Images | 4K frame sequence, **not served by player** (192 files, 33 MB total — dead weight in deployment) |

### 2.2 Third-Party Libraries / CDN Links

| Library | Usage | Version detected | How loaded |
|---|---|---|---|
| **Google Fonts — Geist** | Primary sans-serif | Unknown (URL param only) | `<link>` CDN |
| **Google Fonts — Geist Mono** | Monospace labels | Unknown | `<link>` CDN |
| **Google Fonts — Inter** | Secondary text, case-study body | Unknown | `<link>` CDN |
| **Google Fonts — Instrument Serif** | Display / serif headings | Unknown | `<link>` CDN |

> [!IMPORTANT]
> **No JS libraries at all.** No jQuery, GSAP, AOS, Lottie, Swiper, Framer Motion, or analytics tags. Every animation and interaction is hand-rolled vanilla JS. No third-party tracking/analytics detected.

---

## 3. Pages and Navigation

### 3.1 Pages

| File | URL pattern | Description |
|---|---|---|
| `index.html` | `/` | Main portfolio — single scrollable page |
| `case-study.html` | `/case-study.html?p=<slug>` | Case-study viewer, 15 slugs supported |

**Valid case-study slugs:** `malavo`, `briso`, `linkedin`, `kerala`, `wishes`, `social`, `marketing`, `presentation`, `morpho`, `saelith`, `logofolio`, `rahman`, `wmb`, `tuto`

### 3.2 Navigation

Fixed pill-shaped `<nav>` centred at the top of the viewport:
- **Work** → `#work` (anchor)
- **About** → `#about` (anchor)
- **Contact** → `#contact` (anchor)
- **"Start a project"** CTA → `https://wa.me/919496550431?text=…` (WhatsApp deep-link, `target="_blank"`)

Active link is highlighted with `color: var(--rust)`. Active state updates on scroll via a passive `scroll` event listener on `window`. On mobile (≤430px) the **About** link is hidden via `display:none`.

---

## 4. Sections (index.html, top → bottom)

| # | Section / id | CSS class | Purpose |
|---|---|---|---|
| 1 | `#stage` | `.stage > .sticky` | **Scroll-driven canvas animation** — 700 vh tall sticky container. Canvas draws 192 WebP frames controlled by scroll progress. Three layered "beat" overlays (b1 hero text, b2 transition headline, b3 outro headline) fade in/out via JS. Scroll-hint indicator at the bottom. |
| 2 | `#about` | `.behind` | **Behind the Work** — intro statement with gradient text highlight ("5 years"), tool marquee (infinite CSS animation of tool logos: Claude, Photoshop, Illustrator, ChatGPT, After Effects, Figma; 3× repeated for seamless loop), and portrait illustration. |
| 3 | `#wid` *(implicit, inside About)* | `.wid` | **What I Do** — heading, portrait image (base64-inlined WebP), 6-column service grid (Brand Identity, Packaging, Campaigns, Creative Direction, Product Design, UI/UX). |
| 4 | `#work` | `.works` | **Selected Works** — 2-column responsive card grid with 10 projects (graphic design). Each card links to `case-study.html?p=<slug>` in a new tab. |
| 5 | `#uiux` | `.works.uiux` | **UI/UX Works** — same card layout with 4 UI/UX projects. |
| 6 | `#contact` | `.contact` | **Contact** — full-bleed pink-magenta image panel. Headline, sub-copy, WhatsApp CTA button, and social-link row (no social links were observed in the HTML snippet—section uses `soc` div). |

---

## 5. Features and Interactions

### 5.1 Navigation
- **Fixed/floating pill nav:** `position: fixed`, always visible. No show/hide on scroll.
- **Active state:** Vanilla JS `scroll` listener reads `getBoundingClientRect().top` for `#about`, `#work`, `#contact` and toggles `.on` class (changes link colour to `--rust`). Uses `{ passive: true }`.
- **Smooth scroll:** `scroll-behavior: smooth` on `html`. Disabled via media query for `prefers-reduced-motion`.
- **No mobile hamburger menu:** Nav items shrink on small screens; "About" link disappears at ≤430px.

### 5.2 Scroll-Driven Canvas Frame Player
The star feature. A sticky `<canvas>` inside a `700vh` scroll stage:
- Scroll progress (0→1) is mapped to frame index (0→191).
- JS fetches frames progressively: 8 frames ahead, 3 behind, keeps a cache of 12 frames max (older frames are evicted by removing the `src` attribute).
- An exponential-decay lerp smooths frame playback: `cur += (target - cur) * (1 - 0.001^(dt/1000*1.6))`. Disabled when `prefers-reduced-motion` is true (jumps directly to target frame).
- Uses `requestAnimationFrame` for the render loop.
- Canvas is sized to device pixel ratio (capped at 1.5×).
- `ctx.getContext('2d', { alpha: false, desynchronized: true })` for performance.
- First frame (`frames-hd/f001.webp`) is `<link rel="preload" fetchpriority="high">`.
- Background colour of `.sticky` is `#cfe6fb` (matches the sky in the first frame to avoid flash).

### 5.3 Overlay Text Beats (Scroll-triggered)
Three `position:absolute; opacity:0` divs inside the sticky canvas container animate in/out as scroll progresses:

| Element | Enters | Exits | Properties animated |
|---|---|---|---|
| `#b1` (Hero title) | p=0–0.20 | p=0.20–0.36 | `opacity`, `translateY` (-14px to 0) |
| `#b2` ("Let me show you how I work") | p=0.44–0.50 | p=0.62–0.70 | `opacity`, `translateY` (+14px offset) |
| `#b3` ("Here's what I do") | p=0.93–1.0 | — | `opacity` only |
| `#s2` (bottom scrim gradient) | Same as b2 | Same as b2 | `opacity` |
| `#hint` (scroll indicator) | — | p=0–0.03 | `opacity` fade-out |

All set via direct `.style.opacity` and `.style.transform` writes in the rAF loop.

### 5.4 Scroll Hint Indicator
A small "Scroll ↓" label with an animated drip bar (`::after` pseudo-element, CSS `@keyframes drip` moving `top: -40% → 110%`, duration `1.6s`, `ease-in-out`, `infinite`). Hidden with `animation: none` for `prefers-reduced-motion`.

### 5.5 Tool Logo Marquee
CSS-only infinite horizontal scroll. `.track` has `animation: mq 40s linear infinite` where `@keyframes mq { to { transform: translateX(-50%) } }`. The track contains 3× the full set of logos so the loop is seamless. Disabled with `animation: none` for `prefers-reduced-motion`.

### 5.6 Work Cards
- Hover: `img` scales to `transform: scale(1.03)` (0.5s `transition`), arrow icon rotates `45deg` (0.25s `transition`).
- Hover effects are suppressed on touch screens via `@media (hover: none)`.
- Cards are `<a>` tags opening `case-study.html?p=slug` in `target="_blank"`.

### 5.7 Contact CTA
- Hover: `transform: translateY(-2px)` on the `.cta` pill button.
- Links to WhatsApp: `https://wa.me/919496550431?text=Hi%20Abhiraj…`

### 5.8 Contact / Inquiry Form
**There is no form.** The contact mechanism is a WhatsApp deep-link only. No email `mailto:`, no form fields, no third-party form service (Formspree, Netlify Forms, etc.).

### 5.9 Custom Cursor
**None.** Default browser cursor throughout.

### 5.10 Page Transitions
**None.** Standard browser navigation between `index.html` and `case-study.html`.

### 5.11 Parallax
**None** implemented with CSS `transform` or JS. The sticky canvas creates a *scroll-scrubbing* effect which feels parallax-like, but it is frame-sequence playback, not layer parallax.

### 5.12 Dark / Light Mode
**Light mode only.** `color-scheme: light` on `:root` in `index.html`. `color-scheme: dark` on `:root` in `case-study.html`. No toggle.

### 5.13 Case Study Viewer
Vanilla JS IIFE reads `?p=slug`, looks up a hardcoded `CASES` object (embedded in the HTML), builds `<img>` elements with correct `width`/`height` attributes, sets `loading="lazy"` on all but the first two, sets `decoding="async"` on all. Navigation back to portfolio uses `href` from the `CASES` data (`index.html#work` or `index.html#uiux`).

---

## 6. Animation Inventory

| ID | Element | Trigger | Property | Duration | Easing | Delay | Loops | Compositor-safe? |
|---|---|---|---|---|---|---|---|---|
| A1 | Canvas frame sequence | Scroll (rAF loop) | `drawImage` on canvas | Continuous | Exponential lerp | — | No | ✅ Yes (canvas) |
| A2 | `#b1` hero text | Scroll (rAF loop) | `opacity`, `transform: translateY` | Continuous | Linear (progress-mapped) | — | No | ✅ Yes |
| A3 | `#b2` transition text | Scroll (rAF loop) | `opacity`, `transform: translateY` | Continuous | Linear | — | No | ✅ Yes |
| A4 | `#b3` outro text | Scroll (rAF loop) | `opacity` | Continuous | Linear | — | No | ✅ Yes |
| A5 | `#s2` bottom scrim | Scroll (rAF loop) | `opacity` | Continuous | Linear | — | No | ✅ Yes |
| A6 | `#hint` scroll indicator | Scroll (rAF loop) | `opacity` | Continuous | Linear | — | No | ✅ Yes |
| A7 | `hint i::after` drip bar | Page load (CSS) | `top` (-40% → 110%) | 1.6s | ease-in-out | — | ✅ Infinite | ⚠️ **No — `top` is a layout property** |
| A8 | Marquee `.track` | Page load (CSS) | `transform: translateX(-50%)` | 40s | linear | — | ✅ Infinite | ✅ Yes |
| A9 | Card image scale | Hover (CSS) | `transform: scale(1.03)` | 0.5s | ease (default) | — | No | ✅ Yes |
| A10 | Card arrow icon | Hover (CSS) | `transform: rotate(45deg)` | 0.25s | ease (default) | — | No | ✅ Yes |
| A11 | Nav links colour | Hover / scroll | `color` | 0.2s | ease (default) | — | No | ⚠️ **No — `color` triggers repaint** |
| A12 | Nav CTA background | Hover (CSS) | `background` | 0.2s | ease (default) | — | No | ⚠️ **No — `background` triggers repaint** |
| A13 | Contact CTA button | Hover (CSS) | `transform: translateY(-2px)` | 0.2s | ease (default) | — | No | ✅ Yes |
| A14 | "View all" button | Hover (CSS) | `background` | 0.2s | ease (default) | — | No | ⚠️ **No — `background` triggers repaint** |

---

## 7. Design Tokens

### 7.1 CSS Custom Properties (`:root`, index.html)

| Token | Value | Usage |
|---|---|---|
| `--ink` | `#0a0a0a` | Primary text, body copy |
| `--blue` | `#2770c4` | Accent (italic serif text, focus outline) |
| `--rust` | `#b24a0a` | Hover / active nav links, CTA hover bg |
| `--paper` | `#ffffff` | Page background |
| `--btn` | `#2e2e2e` | Nav "Start a project" button bg |
| `--lime` | `#e5ff59` | Arrow icon badge bg |
| `--serif` | `"Instrument Serif", "Iowan Old Style", "Times New Roman", serif` | Display headings |
| `--sans` | `"Geist", "Inter", system-ui, …, sans-serif` | Body / UI |
| `--inter` | `"Inter", system-ui, …, sans-serif` | Card captions, body text |
| `--mono` | `"Geist Mono", ui-monospace, …, monospace` | Section labels, tags |

### 7.2 Case-study page tokens (`:root`, case-study.html)

| Token | Value |
|---|---|
| `--bg` | `#0a0a0a` |
| `--fg` | `#fff` |
| `--mut` | `rgba(255,255,255,.65)` |

### 7.3 Typography Scale

| Element | Font | Size | Weight | Line-height | Letter-spacing |
|---|---|---|---|---|---|
| `h1` (hero) | `--serif` | `3.3vw` / `clamp(1.5–2.6rem, responsive)` | 400 | 1 | 0 |
| `h2` (section .serif92) | `--serif` | `clamp(2.6rem, 4.79vw, 92px)` | 400 | 1 | 0 |
| `h2` (b2 overlay) | `--serif` | `clamp(2.4rem, 5.4vw, 6rem)` | 400 | 1 | 0 |
| `h2` (contact) | `--serif` | `clamp(3.2rem, 6.67vw, 128px)` | 400 | 0.97 | 0 |
| `.statement` | `--sans` | `clamp(1.55rem, 1.98vw, 38px)` | 300 | 1.2 | -0.05em |
| `.col h3` | `--sans` | `clamp(1.2rem, 1.4vw, 27px)` | 400 | 1.35 | 0 |
| `.col .body` | `--inter` | `clamp(.95rem, 1.04vw, 20px)` | 400 | 1.35 | 0 |
| `.cap b` (card title) | `--inter` | `clamp(1.15rem, 1.25vw, 24px)` | 500 | 1.3 | -0.02em |
| `.cap` (card subtitle) | `--inter` | `clamp(.9rem, .9vw, 17px)` | 400 | 1.35 | -0.01em |
| `.lab .lt` (labels) | `--mono` | `15px` | 400 | 1.2 | -0.03em |
| `.col .tags` | `--mono` | `clamp(.78rem, .83vw, 16px)` | 400 | 1.35 | 0 |
| Nav links `.l` | `--sans` (implicit) | `22px` → `16px` mobile | 400 | — | -0.28px |
| Nav CTA `.go` | `--sans` | `20px` → `15px` mobile | — | — | -0.28px |

### 7.4 Colour Palette (additional values used in CSS)

| Value | Used for |
|---|---|
| `#cfe6fb` | Canvas stage background (matches intro sky) |
| `#b01a64` | Contact section background (magenta/pink) |
| `#5ca9e4` | Canvas element CSS fallback background |
| `rgba(10,30,60,0–0.7)` | Bottom scrim gradient in canvas |
| `rgba(255,255,255,.08)` | Case-study header border |
| `rgba(10,10,10,.82)` | Case-study sticky header background |
| `linear-gradient(31.78deg, #83bf38 0%, #21aa83 99.8%)` | "5 years" gradient text in statement |
| `#f3f5f9` | Marquee tile border |
| `#b3b3b3` | "View all" button border |

### 7.5 Spacing Scale
No explicit spacing tokens. Spacing is expressed as `clamp()` values:
- Section padding: `clamp(4rem, 6.25vw, 120px)`
- Section gaps: `clamp(2.5rem, 4.2vw, 80px)`
- Max content width: `1569px` (`.wrap`), `1521px` (`.statement`), `1400px` (`case-study main`)
- Grid gap: `20px` (work cards), `40px` (service columns)

### 7.6 Border Radii

| Value | Used for |
|---|---|
| `100px` | Nav pill, buttons, arrow badge |
| `2.54%/3.8%` | Work cards (aspect-ratio-aware rounding) |
| `8px` | Case-study gap-layout images |
| `22%` | Marquee app icon tiles |
| `50%` | Card arrow circle |
| `2px` | Scroll hint bar |

### 7.7 Breakpoints

| Breakpoint | Behaviour change |
|---|---|
| `max-width: 1100px` | Service grid → 2 cols (then 3 cols at ≥1101px per responsive refinements); "What I do" head stacks |
| `max-width: 760px` | Work grid → 1 col; hero text repositions; contact text-shadows added |
| `max-width: 640px` | Service grid → 1 col |
| `max-width: 600px` | Nav font sizes reduce |
| `max-width: 430px` | Nav "About" link hidden |
| `max-width: 380px` | Cards change aspect ratio, nav further compressed |
| `max-width: 340px` (portrait) | Statement max-width tightened |
| `max-height: 520px` (landscape) | Nav shrinks, hero text repositions |

### 7.8 Shadows / Filters

| Element | Value |
|---|---|
| Nav pill | `filter: drop-shadow(0 10px 11px rgba(20,15,31,.1))` |
| Hero card text | `text-shadow: 0 1px 14px rgba(0,0,0,.35)` |
| Scroll hint | `text-shadow: 0 1px 12px rgba(8,24,48,.5)` |
| Case-study header | `backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px)` |

---

## 8. Content and Data Sources

| Content | Source | Editable without code? |
|---|---|---|
| Work card images | Static WebP files in `assets/` | No — requires file replacement |
| Case-study images | Static WebP files in `assets/case/<slug>/` | No — requires file replacement |
| Case-study slugs / metadata | Hardcoded JS object in `case-study.html` line 44 | No — must edit HTML |
| Service descriptions (What I Do) | Hardcoded HTML in `index.html` ~lines 1108–1156 | No — must edit HTML |
| Contact phone / WhatsApp link | Hardcoded in `index.html` and `case-study.html` | No — must edit HTML |
| Tool logos in marquee | Inline SVG paths in `index.html` ~lines 959–1090 | No — must edit HTML |
| Designer portrait | Base64-encoded WebP inline in `index.html` ~line 1103 | No — must edit HTML |
| Frame sequence | 192 WebP files in `frames-hd/` | No — requires regeneration |

> [!NOTE]
> There is no CMS, no JSON data file, no Google Sheets fetch, and no database. All content is hardcoded directly in HTML. Any content update requires editing the source files.

---

## 9. SEO and Accessibility

### 9.1 index.html

| Check | Status | Detail |
|---|---|---|
| `<title>` | ✅ | "Abhiraj Rajeev – Graphic Designer & UI/UX Designer" |
| `<meta name="description">` | ❌ **Missing** | No description tag present |
| Open Graph tags | ❌ **Missing** | No `og:title`, `og:description`, `og:image`, `og:url` |
| Twitter Card tags | ❌ **Missing** | None |
| Favicon | ❌ **Missing** | No `<link rel="icon">` |
| Structured data / JSON-LD | ❌ **Missing** | None |
| `lang` attribute | ✅ | `<html lang="en">` |
| `viewport` | ✅ | `width=device-width, initial-scale=1, viewport-fit=cover` |
| `theme-color` | ❌ **Missing** | Not set on index.html |
| Canonical URL | ❌ **Missing** | None |
| `<h1>` | ✅ | Present inside `#b1` as `<h1>Graphic<br><em>Designer</em> &<br>UI/UX <em>Designer</em></h1>` |
| Heading hierarchy | ⚠️ | h1 → h2 (sections) → h3 (service cards) — logical, but h1 is hidden until scroll clears `opacity:0` |
| `alt` text coverage | ✅ Good | All 14 work card images have descriptive alt text. Canvas has `aria-label`. Portrait has alt. |
| Nav `aria-label` | ✅ | `<nav aria-label="Main">` |
| Marquee `aria-label` | ✅ | `<section class="marq" aria-label="Tools I use: …">` with inner div `aria-hidden="true"` |
| Canvas `role` / `aria-label` | ✅ | `role="img"` with descriptive label |
| `:focus-visible` | ✅ | Global style: `outline: 3px solid var(--blue); outline-offset: 3px` |
| `prefers-reduced-motion` | ✅ Partial | Marquee and scroll-hint animation disabled; canvas lerp replaced with direct jump to target frame. Nav active-link colour transitions and card hover transitions NOT disabled. |
| `<noscript>` | ✅ (case-study only) | `case-study.html` has a noscript fallback message |
| `<noscript>` on index.html | ❌ **Missing** | The canvas animation and nav active-state require JS |

### 9.2 case-study.html

| Check | Status | Detail |
|---|---|---|
| `<title>` | ✅ Dynamic | Set by JS: `c.title + ' – Abhiraj Rajeev'` |
| `<meta name="description">` | ❌ **Missing** | None |
| Open Graph | ❌ **Missing** | None |
| Favicon | ❌ **Missing** | None |
| `theme-color` | ✅ | `#0a0a0a` |
| Alt text | ✅ Dynamic | Generated: `c.title + ' – part N of M'` |
| `lang` | ✅ | `<html lang="en">` |
| Focus styles | ✅ (inherited from browser dark scheme) | No explicit `:focus-visible` rule — relies on browser default |

---

## 10. Performance Findings

### 10.1 Flagged Assets

| Asset | Size | Issue |
|---|---|---|
| `assets/case/kerala/kerala-01.webp` | **2.24 MB** 🚨 | Exceeds 2 MB; single tall image; lazy-loaded (only first 2 not lazy) |
| `assets/case/wishes/wishes-01.webp` | **1.98 MB** 🚨 | Near 2 MB threshold |
| `assets/case/social/social-06.webp` | **1.76 MB** | Very large case-study image |
| `assets/case/malavo/malavo-01.webp` | **1.66 MB** | Large |
| `assets/case/briso/briso-01.webp` | **1.45 MB** | Large |
| `assets/case/social/social-04.webp` | **1.52 MB** | Large |
| `frames-4k/` entire folder | **33 MB** | **Served but never requested by the player.** Pure dead weight in deployment. |
| Portrait (base64 in HTML) | **~45 KB** | Inline base64 blocks HTML parsing and cannot be cached separately |
| index.html | **129 KB** | Bloated by inline base64 portrait (~45 KB), 3× repeated SVG tool logos (~60 KB combined), and both CSS blocks |

### 10.2 Performance Findings Table (ranked by impact)

| Rank | Issue | File / Line | Property Type | Fix |
|---|---|---|---|---|
| **P1** | `frames-4k/` folder (192 files, 33 MB) is included in deployment but never fetched by the player | `vercel.json` / README | N/A — wasted bandwidth + CDN cost | Exclude `frames-4k/` from the Vercel deployment (add to `.vercelignore` or move out of the project root) |
| **P2** | **No lazy loading or srcset on work card images** (`work-1.webp` through `work-14.webp`). Images are referenced with no `width`/`height` attributes in CSS, causing CLS (Cumulative Layout Shift). Card images use `loading="lazy"` ✅ but no `srcset` or `sizes`. | `index.html` lines 1173–1280 | N/A — no responsive images | Add `width`/`height` to `<img>` tags (or set via CSS aspect-ratio which is already done via `.card`). Add `srcset` with 2× variants or convert to `<picture>`. |
| **P3** | **Portrait image is base64-encoded inline** (~45 KB) inside the HTML. This bloats HTML, blocks the parser, is uncacheable, and re-decoded on every page load. | `index.html` line 1103 | N/A — delivery | Save as a real `.webp` file (e.g., `assets/portrait.webp`), reference with `<img src="…">`, add `width`/`height` and `loading="lazy"`. |
| **P4** | **Tool logos SVG repeated 3× in HTML** to achieve marquee loop. Each set of 6 SVGs is copied verbatim, adding ~20 KB of markup per copy (60 KB total repeated content). | `index.html` lines 959–1090 | N/A — DOM size | In React migration, use a JS `Array.from({length:3})` map or CSS-only clone via `::before`/`::after`. Do not triplicate DOM nodes. |
| **P5** | **`@keyframes drip` on `hint i::after` animates `top`** (a layout property). Every frame of this infinite animation causes the browser to recalculate layout for that element. | `index.html` line 348–352, 337–346 | ⚠️ **Layout property** | Replace with `transform: translateY()` from `-40%` to `110%`. Same visual result, compositor-only. |
| **P6** | **`backdrop-filter: blur(12px)` on case-study sticky header** is a paint-layer operation. It is applied to a full-width header that is always visible, which creates a stacking-context cost on every frame. | `case-study.html` line 18 | ⚠️ Paint/composite | Acceptable on modern hardware for a single small element, but test on low-end mobile. Consider reducing to `blur(8px)` or using a solid semi-transparent background fallback for `prefers-reduced-transparency`. |
| **P7** | **`filter: drop-shadow(…)` on `.nav`** is applied to the entire nav pill on every frame that it needs to be composited. | `index.html` line 76 | ⚠️ Paint | Use `box-shadow` instead of `filter: drop-shadow` for a rectangular/pill shape — `box-shadow` is handled by the compositor layer without triggering a repaint on the element itself. |
| **P8** | **`color` and `background` CSS transitions on nav links and buttons** cause repaints on hover. | `index.html` lines 93–115 | ⚠️ Repaint properties | For nav link hover, consider transitioning `opacity` of a coloured pseudo-element instead of `color`. For button backgrounds, `opacity` on a layered bg or use `box-shadow` trick. (Low severity for hover-only interactions.) |
| **P9** | **No `width`/`height` on the canvas element.** Canvas is sized 100%/100% of its container via CSS; actual pixel dimensions are set by JS in `resize()`. This is correct but means no native aspect-ratio hint for the browser layout engine before JS runs. | `index.html` line 927 | N/A — CLS | Add `width="1920" height="1080"` attributes to `<canvas>`; CSS `width:100%; height:100%` will still override. |
| **P10** | **`stage.getBoundingClientRect()` and `stage.offsetHeight` both called on every rAF tick** (`progress()` function, line 1391–1393). `offsetHeight` forces a layout reflow if any pending style change exists. | `index.html` lines 1390–1394 | Layout thrashing risk | Cache `stage.offsetHeight` on resize (it only changes on window resize). Only call `getBoundingClientRect()` in the rAF tick. |
| **P11** | **Google Fonts loaded via two `<link rel="preconnect">` + one render-blocking stylesheet link.** The font `<link>` has `display=swap` which helps, but the stylesheet is still a render-blocking request. | `index.html` lines 9–13 | Render-blocking | In React migration, use `next/font` (Next.js) or self-host fonts via `@font-face` with pre-downloaded WOFF2 files to eliminate the CDN round-trip and render-blocking. |
| **P12** | **`case-study.html` large images are not served with responsive `srcset`.** Images like `kerala-01.webp` (2.24 MB, 3240px wide) are served at full resolution even on a 390px mobile screen. | `case-study.html` JS image builder (line 58–64) | N/A — no responsive images | Generate 1× and 2× variants, use `srcset` and `sizes`. Or serve via an image CDN (Cloudflare Images, Cloudinary, Vercel OG) that resizes on the fly. |
| **P13** | **Infinite CSS animations running off-screen.** The `.track` marquee and `hint::after` drip animate continuously regardless of scroll position or whether the element is in the viewport. | `index.html` lines 447–484, 337–352 | Infinite off-screen animation | Use `IntersectionObserver` to toggle `animation-play-state: paused` when the element scrolls out of view. For the hint indicator, the JS already fades it out at p>0.03 — adding `display:none` or `visibility:hidden` at that point would stop the animation entirely. |
| **P14** | **`will-change: opacity, transform` on `.beat` elements** (all three overlay divs). These are present in the CSS but the overlays are always `position:absolute; opacity:0` — they are promoted to their own GPU layer even when invisible. | `index.html` line 179 | `will-change` overuse | Move `will-change` to a JS-added class applied only when the scroll position is in the range where the element will animate, and remove it after. Or rely on the browser's heuristics — since `opacity` and `transform` are already being changed, most browsers will promote automatically. |

---

## 11. Migration Risks

| Risk | Severity | Detail |
|---|---|---|
| **Scroll-driven canvas frame player** | 🔴 High | This is the centrepiece and most complex piece of code. The rAF loop, frame-caching strategy, exponential lerp, and overlay timing windows must all be re-implemented faithfully in React. Consider wrapping the canvas in a `useEffect` with a ref; the JS logic is self-contained and can be ported almost verbatim. The key challenge is ensuring the sticky container's scroll geometry is correct after React's DOM reconciliation. |
| **Case study data embedded in HTML** | 🟡 Medium | The `CASES` object in `case-study.html` will need to become a data file (JSON, TypeScript const, or CMS). The React version will likely use dynamic routing (e.g., `/case/[slug]`). |
| **Inline base64 portrait** | 🟡 Medium | Must be extracted to a file asset before migration; the base64 string is not suitable as a React `src` prop value in source control. |
| **Triple-duplicated SVG logos** | 🟡 Medium | Easy to fix in React (map over an array), but the logo SVG paths are very long inline strings — these should be extracted to separate SVG component files or sprite. |
| **All CSS in a single `<style>` block** | 🟡 Medium | The CSS is well-structured but completely unseparated from markup. Migrating to Tailwind utility classes will require manually mapping every CSS rule, checking every breakpoint, and verifying no style is missed. The responsive refinements `<style id="responsive-refinements">` block contains overrides that sometimes conflict with earlier rules (e.g., `.hl h1` font-size is set twice). |
| **`frames-4k/` in deployment** | 🟡 Medium | 33 MB of unused frames should be excluded before or during migration. If left in, they will bloat the Vercel deployment and slow CI. |
| **No backend / form handling** | 🟢 Low | Contact is a WhatsApp link only. No backend needed. If a contact form is added in the React version, a service like Resend, EmailJS, or Formspree will be needed — no existing backend to wire up. |
| **Google Fonts dependency** | 🟢 Low | Easy to self-host in Next.js with `next/font/google`. The four families (Geist, Geist Mono, Inter, Instrument Serif) all have Google Fonts support. |
| **`scroll-behavior: smooth` on `<html>`** | 🟢 Low | In React/Next.js with client-side routing, smooth scroll anchor navigation needs to be re-implemented (the browser's native CSS `scroll-behavior` works for hash changes, but Next.js Link components may not trigger it reliably). |
| **`target="_blank"` on all case-study links** | 🟢 Low | All 14 work cards open in a new tab. In React migration, decide if this is intentional UX or if SPA navigation to `/case/[slug]` is preferred. |
| **`prefers-reduced-motion` partial coverage** | 🟢 Low | The canvas lerp and CSS animations are handled. CSS `transition` properties on nav links and hover effects are NOT. During migration, audit all Tailwind `transition-*` utilities and add `motion-reduce:transition-none` where appropriate. |

---

## 12. Open Questions for the Owner

1. **`frames-4k/` folder** — Should the 4K frame sequence be kept for future use (e.g., if you add a 4K display mode), or can it be permanently deleted? Excluding it removes 33 MB from the deployment.

2. **Contact mechanism** — Is the WhatsApp link the permanent contact method, or would you like a contact form added during the React migration? (If a form is added, we'll need to decide on a backend service: Resend, Formspree, EmailJS, etc.)

3. **Case-study data** — Should the case-study content (titles, file lists) move to a JSON/TS file, or would you prefer a lightweight CMS (Contentful, Sanity, Notion API) so you can add new projects without touching code?

4. **New case studies** — The nav only has Work / About / Contact. Should the React version add separate `/case/[slug]` routes with their own URLs for SEO, or keep the current `?p=slug` pattern on a single page?

5. **Dark mode** — `case-study.html` is dark-themed while `index.html` is light. Is this intentional? Should the React portfolio offer a user-toggleable dark/light mode?

6. **SEO priority** — The site currently has no meta description, no Open Graph tags, and no favicon. Should these be added in the React migration? If so, what should the OG image be — a screenshot of the canvas animation, the portrait, or a custom graphic?

7. **Analytics** — No analytics tracking is present. Should the React version include Google Analytics 4, Plausible, or another analytics tool?

8. **"Start a project" / WhatsApp number** — Confirm the WhatsApp number (`+91 94965 50431`) is correct and should remain the primary CTA.

9. **Social links** — The contact section markup has a `.soc` div but no actual links were visible in the parsed HTML (the section ends before them). Please confirm which social profiles (Behance, Dribbble, LinkedIn, Instagram) should appear there, and what the URLs are.

10. **Scroll-animation strategy for React** — The canvas frame player is tightly coupled to the browser scroll API. Should the React version keep the same vanilla-JS-in-`useEffect` approach, or migrate to a library like GSAP ScrollTrigger or Framer Motion scroll? (Using a library trades bundle size for maintainability.)
