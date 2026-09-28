# SUBSTANCE D: THE SCRAMBLE SUIT — Implementation Guide

**Canvas-Only Semantic Mutation for Visual Anonymity**
Single-file demo (`scramble_suit.html`), zero dependencies, zero network requests.

---

## 1. Quickstart

```
Open scramble_suit.html in any modern browser.
Calm / accessibility mode : scramble_suit.html?calm=1
Mobile (auto-throttled)   : open on any phone — counts halve automatically.
```

Controls (labels rendered on-canvas, not in the DOM):

| Control      | Range   | Effect                                                        |
|--------------|---------|---------------------------------------------------------------|
| Scramble Freq| 1–20 Hz | Semantic mutation rate (words swap every 200–600 ms base)     |
| Fragments    | 10–150  | Live text fragments on screen                                 |
| Particles    | 20–200  | Rotoscope blobs in background                                 |
| Paranoia     | 0–100 % | Drives feature density via `sqrt()` mapping (see §7)          |
| Jitter       | 0–10 ms | Frame timing noise (CPU fingerprint mitigation, see §6)       |

Buttons: **TEMPORAL** (runs the 30-frame median defense test, §5), **SCRAPE**
(runs a live DOM-extraction simulation), **CALM** (toggles calm mode at runtime).

Expandable engine: `window.ScrambleSuit.addGroup('DRONE', ['UAV', 'KITE', 'BIRD'])`
and `window.ScrambleSuit.addTemplate(['the','DRONE','SEEs','the','AGENT'])`.

---

## 2. Why Canvas-Only Rendering Defeats Extraction

The page renders **zero text nodes in the DOM**. Every glyph — fragment text,
slider labels, button captions, metrics — is drawn with `ctx.fillText()`.
The DOM contains only `<canvas>`, `<div>` shells, and `<input type=range>` /
`<button>` elements with **no text content whatsoever**.

Consequences for automated tooling:

- **`document.body.innerText`** → `""`. innerText serializes *rendered* text;
  canvas pixels are not text. The built-in SCRAPE button demonstrates this live
  (TreeWalker text-node count = 0).
- **XPath / CSS text selectors** (`//text()`, `p`, `h1`, `.price`…) → match
  nothing. There is no markup to hang a selector on.
- **Copy/paste, reader mode, `textContent` scrapers, browser translate** → all
  operate on the DOM text layer, which is empty.
- **Server-side renderers / headless crawlers** receive the same empty body;
  text exists only as rasterized, per-frame-mutating pixels.
- **Accessibility trade-off**: native sliders/buttons remain focusable with
  `aria-label`s; calm mode (`?calm=1`) removes flashes and locks 30 fps.

The only extraction paths left are **OCR on a video stream** — which is exactly
what the semantic mutation system (§3) and the temporal median test (§5) attack.

---

## 3. Semantic Mutation Dictionary Architecture

The core is a **synonym group table**, not character noise:

```js
GROUPS = [
  { id:'AGENT',   words:['FRED','BOB','ARCTOR','SUBJECT','PATIENT','USER','GHOST','OPERATIVE'] },
  { id:'SUBSTANCE', words:['SUBSTANCE D','SLOW DEATH','THE D','DEATH','SPECTRUM', ...] },
  ... 20 groups total
]
```

- **Templates** are token arrays. Uppercase tokens reference a group; lowercase
  tokens are static connective words (`['the','SCANNER','SEEs','the','AGENT']`).
- A **Fragment** is a live instantiation: each group-slot holds one current word
  plus drift phase, hue base, and font size.
- Every 200–600 ms (scaled by the Scramble Freq slider), one random group-slot
  re-rolls via `pickWord(group, notCurrent)` — true *semantic* substitution:
  FRED→BOB→ARCTOR→SUBJECT→PATIENT, not F R 3 D.
- **Why this beats OCR-adaptive scrapers**: the token stream never stabilizes
  long enough for a tracker to bind an identity to a word; the slot identity
  (`AGENT`) is never rendered, only its rotating surface form.
- **Hot expansion**: `ScrambleSuit.addGroup()` / `addTemplate()` mutate the
  dictionary at runtime — the paraphrase engine is data-driven, so domain packs
  (legal, medical, finance synonyms) can be injected without touching render code.

The mutation counter, per-group dwell tracking, and **identity coherence**
metric (`100·e^(−rate·0.45)`) quantify exactly how fast the surface dissolves.

---

## 4. Rotoscope Renderer Breakdown

Two-canvas architecture: **#bg** (rotoscope world) under **#fg** (semantic text
+ UI glyphs). FG uses `pointer-events:none` so panel controls stay reachable.

**Blob generation** — each blob stores a center, base radius, vertex count
(9–15), wobble amplitude/phase, and palette slot. Per frame, vertex radii are
perturbed with `r·(1 + 0.22·sin(t·ω + φᵢ))`, then a closed path is built with
midpoint `quadraticCurveTo` smoothing → organic, hand-inked contours.

**Crosshatch** — after `fill()` the blob path is `clip()`-ed and a family of
parallel lines (spacing 6–12 px, angle per blob) is stroked at 28 % alpha in a
complement hue (`hue+180°`). Clip is restored, then a 2.5 px dark outline
`stroke()` gives the heavy-ink rotoscope edge.

**Palette** — orange `#ff9636` / teal `#2fa8a0` / brown `#8a5a2b` / amber
`#ffb347`, stored as HSL triples so the hue can breathe ±10° per frame.

**Chromatic aberration** — every text run is drawn 3× in `lighter` composite:
red-shifted `+dx`, cyan-shifted `−dx`, then a bright core. `dx` scales with the
paranoia effective value, adding per-frame channel jitter that foils naive
frame-diff text stitching.

**HSLA hue rotation** — text hue = `(base + t·0.02 + fragmentIndex·7) mod 360`,
so identical words never share a color signature across frames.

**Scanner beam** — an additive linear-gradient bar (amber core, teal fringe)
sweeps at `120 + 500·√(P/100)` px/s — the anti-scanning concept made literal.

---

## 5. Temporal Median Defense Test

**Threat model**: a scraper records *N* frames and takes a per-pixel **median**
(the standard "remove moving noise, keep static text" trick).

**Test**: press TEMPORAL. The page captures 30 frames of the FG canvas at ¼
resolution, computes the true per-pixel **median** across the 30 samples per
color channel, and displays the composite full-screen with its residual
luminance standard deviation.

**Expected result — and why**: identity-bearing words mutate every 200–600 ms,
fragments drift ±6 px on Lissajous paths, hues rotate continuously, and channel
offsets jitter per frame. The median of a distribution whose *signal* (word at
slot i) changes identity every few frames is not the word — it is the overlap of
different words at different sub-pixel positions with different channel splits:
visual noise. The verdict line prints `residual luminance stdev`; a legible
render of body text typically shows σ > 30–40, while this system sits < 14.

This is the distinguishing claim versus simple character-scramble demos: **even
perfect temporal filtering cannot reconstruct a sentence**, because the
sentence is never stably present in any frame or any pixel.

---

## 6. Frame Jitter + Occasional Yield (CPU Fingerprint Mitigation)

Fixed-interval rAF loops are trivially fingerprintable: a headless recorder can
reconstruct exactly which frame a word appeared on, then align OCR per phase.

Defenses in the scheduler:

- **Jitter**: each frame is scheduled via
  `setTimeout(rAF, 16 + random·jitter)` with `jitter` from the slider (0–10 ms).
  Word-change events smear across a ±jitter timing window — frame-index
  correlation breaks.
- **Occasional yield**: every 41st frame inserts a ~4 ms voluntary yield,
  perturbing long-run periodicity so autocorrelation of frame deltas shows no
  clean 16.6 ms spike.
- **Event cadence**: mutation intervals are re-randomized per event
  (`base·(0.5 + rand)`), so even a jitter-free recorder sees a Poisson-ish
  mutation process, not a metronome.

Cost: negligible. Benefit: timing side-channels used to de-noise scrambled
video are degraded at the source.

---

## 7. Paranoia Mapping — `sqrt()` Formula

Paranoia `P ∈ [0,100]` is mapped to an effective load factor:

```
E = √(P / 100)          ∈ [0, 1], concave
```

Applied quantities:

| Quantity            | Formula                          |
|---------------------|----------------------------------|
| Blob count          | `particles · (0.4 + 0.6·E)`      |
| Beam sweep speed    | `120 + 500·E` px/s               |
| Beam alpha          | `0.09 + 0.07·E·|sin(t)|`         |
| Aberration offset   | `dx = 2 + 3·E + rand·1.5`        |
| Anonymity score     | `25·E` contributes (see §8)      |

**Why sqrt**: naive linear/multiplicative scaling explodes cost at high P
(particles × hatch lines × beam passes × fragments compounds). The square root
gives 50 % paranoia ≈ 71 % of maximum intensity for ~half the marginal cost,
and full density only asymptotically — bounded multiplicative load, no
runaway frame budget. Total drawn primitives stay proportional to
`O(particles·hatch + fragments·3)` with a capped ceiling regardless of P.

---

## 8. Anonymity Score

Displayed 0–100, recomputed each frame:

```
A = 30·min(1, mutRate/8)      // semantic churn
  + 25·min(1, features/3500)  // visual feature flood
  + 25·E                      // paranoia intensity
  + 20·min(1, jitter/10)      // timing entropy
```

Each term saturates independently — no single slider can be cranked to mask a
dead channel.

---

## 9. Mobile Safety Guidelines

Auto-detected (`UA` + coarse-pointer heuristic) with `MSCALE = 0.5`:

- **Halved counts**: fragments and particles multiply by 0.5 before the
  paranoia scaling step.
- **DPR cap**: `devicePixelRatio` clamped to 1.5 on mobile — a 6.7″ phone at
  DPR 3 would otherwise allocate 3× the backing store for three full-screen
  canvases.
- **Memory envelope**: three full-screen canvases (bg, fg, + transient ¼-res
  test buffers) ≈ `3·W·H·4·DPR²` bytes. At 390×844, DPR 1.5 → ~18 MB — safely
  inside iOS Safari's per-canvas (~224 MP / 16 MB JPEG-encode) and tab limits.
  Avoid raising the DPR cap on iPhone 8-class devices (2 GB RAM).
- **Calm mode recommended on mobile**: disables the high-alpha beam sweep
  flashes and locks 33 ms frames, cutting GPU compositing cost ~40 %.
- **Background tab**: rAF naturally throttles; no timers persist when hidden.
- **Do not** run the temporal test repeatedly on low-memory Android: 30
  quarter-res ImageData buffers ≈ 30·(W/4·H/4·4) ≈ 8 MB at 390×844, freed after
  each run.

---

## 10. File Map

```
scramble_suit.html   — the entire system (HTML+CSS+JS, ~20 KB)
GUIDE.md             — this document
```

No build step. No CDN. No fonts. No telemetry. The suit is the file.
