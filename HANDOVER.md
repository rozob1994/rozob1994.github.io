# eccho Portfolio — Project Handover

> Handover notes for continuing work on this site (written for a human or another AI).
> Everything lives in **one file**: `index.html`. Read this before editing.

---

## 1. What this is

- Personal portfolio for **Roozbeh Ghanbarzadeh** — trading as **eccho**, a 3D & motion design studio.
- Owner email: `roozbeh.ghanbarzadeh1994@gmail.com`.
- The bulk of the work shown is for the client **Mofid Securities** (a brokerage), plus a "Personal Work" tab.
- Single-page site: dark cinematic hero (video showreel) → light **Work** section (two tabs: Mofid / Personal) → About → Services → Contact.

## 2. Repository & deployment

- **Repo:** `rozob1994/rozob1994.github.io` (GitHub Pages user site).
- **Live site is served from the `main` branch** at the Pages URL. Pushing to `main` = deploying.
- **Everything is in `index.html`** (~4100 lines, ~250KB): inline `<style>` and inline `<script>` blocks. No build step, no framework for the DOM. There is a bundled **Three.js** (`vendor/three.module.min.js`, loaded via an importmap) used only for the About-section floating spheres.
- Directories:
  - `vendor/` — `three.module.min.js`, `RGBELoader.js`, `studio_small_09_1k.hdr` (HDR env for the 3D spheres), `postprocessing/`, `shaders/`.
  - `media/` — all images/video. `media/mofid/<category>/<slug>.jpg|.mp4` for Mofid work; `media/<project>/` for personal; `media/mofid/3d-designs/01.jpg…48.jpg` for the gallery carousel; `media/showreel.mp4`, `media/contact-photo.jpg`, etc.
  - `frames/` — JPEG frame sequence for the intro logo animation (was 82MB PNG, converted to ~2.9MB JPEG; `img.src` uses `.jpg`).

## 3. Git workflow (IMPORTANT)

- **Develop on branch `claude/portfolio-website-design-joq72y`.** Commit there, `git push -u origin <branch>`, then deploy with `git push origin HEAD:main`.
- **Do NOT open PRs** unless explicitly asked.
- Commit message trailer convention used throughout:
  ```
  Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_...
  ```
- **Backup branch: `carousel-css-backup`** (at commit `c24b389`) — the known-good CSS carousel, kept as a restore point after a reverted WebGL experiment (see §7).
- To revert `index.html` to the backup: `git checkout c24b389 -- index.html` then commit + push to branch and main.

## 4. Environment constraints (things that bit us)

- Runs in an **ephemeral cloud container** — no access to the user's local disk. **Assets must already be in the repo.** You **cannot save a pasted image** to disk (a pasted image has no file path). If new image files are needed, the owner must add them to the repo.
- **No GPU** in the dev/test container → WebGL runs in **SwiftShader (software), ~2fps**. You **cannot measure WebGL performance or reliably screenshot GPU-heavy scenes here.** Real perf/visuals must be judged on actual hardware (the live site).
- **Desktop intro can't be screenshotted in-container:** the intro/loader only completes with a GPU. The **touch/coarse-pointer path auto-skips the intro** (via `SKIP_INTRO`) — that's why tests use a mobile/iPad user-agent.

## 5. File map (`index.html` regions, approx line numbers — grep to confirm)

- `<style>` … all CSS. Carousel CSS ~**1145–1195**.
- Hero / nav / loader HTML ~1220–1245.
- Work section HTML (tabs, panels, `#pcarAll`) ~1255–1280.
- `ECCHO_PROJECTS` (personal projects data) ~1400+.
- `initWorkTabs()` (tab switching; builds panels on demand) ~**2035**.
- `revealWork()` (scroll-reveal of work cards) ~2232; exposed as `window.revealWork`.
- Personal panel: `buildPersonalPanel()` ~**2384**; filter carousel `rebuildCarousel`.
- **`MOFID_SECTIONS_DATA`** (categories → slugs) ~**2432** (one big JSON line).
- **`galleryImgs(from,count)`** ~**2441** → builds `media/mofid/3d-designs/NN.jpg` paths.
- **`buildCarousels(host, configs)`** ~**2453** — the 3D-Designs scroll carousel (see §6). THE main active feature.
- **`buildMofidPanel()`** ~**2604** — builds the Mofid grid + featured + the carousel.
- `MOFID_DESC_BY_SLUG` (per-project descriptions) ~**2708**.
- Mofid overlay/expand/gallery viewer ~2820–3000.
- Project overlay open logic `window.openMofidProject` ~3675+.
- `ECCHO_PROJECTS`/`projects` render + `DEFAULT_CREDITS` ~3308/**3368**.
- **`MOFID_CREDITS`** ~**3565**.
- Three.js About spheres module (importmap + `<script type="module">`) ~**3933–4110**.

## 6. The "3D Designs" gallery carousel — FULL detail (current focus)

This is the most iterated and most complex feature. It's a **single pinned canvas** holding **three hero-stack carousels** that play in sequence as you scroll.

### 6.1 Concept
- **48 images**, `media/mofid/3d-designs/01–48.jpg`, split **16 per carousel**: #1 = 01–16, #2 = 17–32, #3 = 33–48.
- **Carousel #1 (`side:right`)**: a vertical arc bulging right — hero at leftmost-middle, cards fan up-right and down-right. Text **"be bold"** (2nd word blue) to the left of the hero.
- **Carousel #2 (`side:left`)**: mirror of #1. Text **"tell your story"** to the right.
- **Carousel #3 (`side:bottom`)**: a horizontal dome — hero at the top peak, cards fan down to both sides. Text **"make your brand Resonate"** centered above.
- One tall parent (`#pcarAll`) pins a single sticky layer; a vertical **track** slides so each carousel cycles its hero, then rolls up and out while the next rises in from below. **No cross-fade** — it's vertical flow.

### 6.2 DOM structure (built by JS)
```
#pcarAll (.pcar-all, full-bleed 100vw)
  .pcar-sticky (position:sticky; top:0; height:100vh; overflow:hidden)
    .pcar-track (absolute; translateY driven by scroll)
      .pcar-stage.side-right   (top: 0*100vh)   → .pcar-text, .pcar(cards)
      .pcar-stage.side-left    (top: 1*100vh)
      .pcar-stage.side-bottom  (top: 2*100vh)
```
Each card `.pcar-card` is a real **CSS-3D slab**: `.pcar-face` (image, front) + `.pcar-back` (translateZ(-t)) + four `.pcar-edge` glass walls (`.pe-l/-r/-t/-b`, rotated 90° via `--t` thickness = 24px). `.pcar{perspective:1250px}` gives real perspective; `.pcar-card{transform-style:preserve-3d}`.

### 6.3 Controller: `buildCarousels(host, configs)` (~line 2453)
`configs` = 3 objects `{side, text (HTML, `<em>` = blue word), imgs}`.

Each carousel object: `{side, dir (left=-1 else +1), horizontal (side==='bottom'), stage, text, cards[], cur, target, heroIdx}`.
- `cur` = eased hero index (float). `target` = scroll-derived hero index.
- `ENTRANCE = 6` (how many cards back the stack starts for swipe-in). `EASE = 0.11` (rAF ease toward target).

**`dims()` tuning constants** (all recomputed on resize; `vw/vh` = window size):
| const | value | meaning |
|---|---|---|
| `SPY` | `vh*0.128` | side arc vertical spacing per card |
| `BVX` | `vw*0.026` | side arc rightward bend (× a²) |
| `SPX` | `vw*0.088` | dome horizontal spacing per card |
| `BVY` | `vh*0.011` | dome downward bulge (× a²) |
| `PULLX` | `vw*0.05` | hero pop-out distance (side) |
| `PULLY` | `vh*0.075` | hero pop-out distance (dome, up) |
| `DZ` | `vw*0.05` | how far each card recedes in Z per step (real depth) |
| `TXTOFF` | `vh*0.08` | text swipe-up distance on entrance |
| `STEP` | `max(90, vh*0.21)` | scroll distance per card (bigger = smoother, no snap) |
| `CYCLE` | `(N-1)*STEP` | scroll to cycle one carousel end-to-end |
| `SHIFT` | `vh*1.3` | inter-carousel transition slide length |
| `LEADIN`| `vh*0.45` | carousel #1's own (shorter) entrance length |
| `PERIOD`| `CYCLE+SHIFT` | one carousel's full scroll span |

Canvas height: `vh + LEADIN + M*CYCLE + (M-1)*SHIFT` (M=3).

**Scroll schedule** (g = scroll depth into canvas = `max(0, -host.getBoundingClientRect().top)`):
- `trackUnits(g)`: `g<=LEADIN → 0`; else offset by LEADIN, `j=floor(gg/PERIOD)`, parked at `j` during CYCLE then ramps `j→j+1` during SHIFT. Track transform = `translateY(-trackUnits*vh)`.
- `carTarget(k,g)`: cycle for carousel k starts at `cyc = LEADIN + k*PERIOD`. Entrance window length `inLen = (k===0? LEADIN : SHIFT)`. Before window → `-ENTRANCE`; in window → `-ENTRANCE*(1-easeIO(frac))` (swipe in along arc); in cycle → `clamp((g-cyc)/STEP,0,N-1)`; after → `N-1` (held).
- `easeIO` = cubic ease-in-out. This is the "standard ease."

**`placeCar(c)`** — per card i: `u = i-cur`, `a = |u|`, `pop = exp(-a²*0.8)`.
- side: `x = dir*(a²*BVX) - dir*PULLX*pop`, `y = -u*SPY`, `rot = dir*u*0.6`.
- dome: `x = u*SPX`, `y = a²*BVY - PULLY*pop`, `rot = (u<0?-1:1)*a*1.2`.
- depth/turn: `z = -a*DZ`; `ry = clamp(-dir*u*5, -34, 34)` (Y-turn); `s = 0.92 + 0.12*exp(-a²*0.7)` (hero bump).
- `o = clamp((4.0-a)/1.1, 0, 1)` (opacity fade); `shade = min(0.6, a*0.08 + |ry|/78)` → `--shade` (face darkens on turn/recede via `.pcar-face::after`).
- transform: `translate(-50%,-50%) translate(x,y) translateZ(z) scale(s) rotateY(ry) rotate(rot)`; `z-index = 300 - round(a*6)`.

**Text swipe** (top of `placeCar`): `tp = clamp(1 - max(0,-cur)/ENTRANCE, 0,1)`; sets `text.style.opacity=tp` and `--ty=(1-tp)*TXTOFF`. Tied to `cur`, so it swipes up on entrance and **reverses on scroll-up**.

**Loop:** `frame()` eases every car's `cur += (target-cur)*EASE`, calls `placeCar`, reschedules rAF while moving. `onScroll()` sets `curG`, track transform, and each `target`, then `kick()`s the frame. An `IntersectionObserver` on `.pcar-sticky` sets `started=true` on first view.

### 6.4 Current behavior (as of latest commit)
- Cards are CSS-3D glass slabs (real perspective, real translateZ depth, glass side walls, face shading on turn).
- #1 has a **short fast entrance** (LEADIN 0.45 screen); it swipes in centered (no empty vertical gap) and **reverses out** on scroll-up. #2/#3 rise from below on their SHIFT transitions.
- Texts swipe up from below with the ease and reverse on the way up.

## 7. Carousel evolution & REJECTED approaches (don't repeat these)

The owner iterated heavily. Things that were tried and **explicitly rejected**:
- **Hover-to-scroll** the carousel → rejected; must be pure page-scroll driven.
- **Yellow matte ball** next to hero, then a **12-ball matte-blue field with card collision** → all **removed** ("way too much").
- **Hero frame** (thin blue/yellow border) → **removed**.
- **S-curve (sigmoid) path** for #1/#2 → reverted; owner wanted the **arc** back.
- **Cross-fade** between carousels → rejected; wanted **vertical flow** (one moves up/out, next comes up).
- **WebGL/Three.js real-3D cards** ("Option A hybrid": glass hero + solid slabs, `window.__pcarData` bridge, per-carousel takeover, CSS fallback). Built on commit `f50c4a3`, **owner said "it's not good, revert"** → reverted in `b38761f`. Backup of the good CSS state is branch `carousel-css-backup`. **Note:** couldn't be verified in-container (no GPU). If revisiting real 3D, that's the reference point.
- Owner's taste: minimal, premium, "actual" depth, subtle turns ("don't turn them that much"), fast entrance, hero clearly dominant and popped out of the fan, no cutoffs.

## 8. Mofid data model

- **Categories → slugs:** `MOFID_SECTIONS_DATA` (~2432). Each `{key, label, slugs[]}`. `key` is the media folder name (`media/mofid/<key>/<slug>.jpg|.mp4`).
- **Grid order:** `MOFID_GRID_ORDER` (~1977) — explicit card order; `'__ident__'` is a merged "Identity R&D" card (own thumbnail `media/mofid/mofid-ident-r-d/gallery/05.jpg` + carousel popup).
- **Hidden:** `MOFID_HIDDEN` (~1967) — slugs excluded from the grid.
- **Featured:** `MOFID_FEATURED` (~1959) — `{slug, catKey}` shown in the featured row.
- **Descriptions:** `MOFID_DESC_BY_SLUG` (~2708) — per-slug HTML blurbs (`<strong>` for keyword highlights). Fallback: `MOFID_DESC[catKey]` then a generic string.
- **Credits:** `MOFID_CREDITS` (~3565) = `[{name:'Ehsan Abbasi', title:'Art Director', url:'behance.net/ehsanabbbbasi'}, {name:'Roozbeh Ghanbarzadeh', title:'3D & Motion Designer'}]` — applied to all Mofid projects.
- Card titles are humanized from slugs; several were renamed to "… Fund"/event names (e.g. Octane Fund, Tavan Fund, AIX Event, Jahat 1402 Teaser). Check `mofidSlugToTitle` / title overrides.

## 9. Personal projects data model

- `ECCHO_PROJECTS` (~1400) — array of `{id, title, type, tags[], gallery (count of stills), hasFull, ...}`. Stills load from `media/<id>/NN.jpg`.
- `DEFAULT_CREDITS` (~3368) = Roozbeh ×3 (owner hasn't supplied real per-project collaborators for most personal work).

## 10. Testing (no unit tests — visual + syntax)

- **JS syntax check** (there's no build): extract non-`src`, non-importmap `<script>` blocks and `node --check`:
  ```bash
  python3 - <<'PY'
  import re; h=open('index.html').read()
  b=re.findall(r'<script((?![^>]*\bsrc=)[^>]*)>(.*?)</script>',h,re.S)
  open('/tmp/_chk.js','w').write('\n;\n'.join(x for a,x in b if 'importmap' not in a and 'application/json' not in a))
  PY
  node --check /tmp/_chk.js
  ```
- **Visual/responsive:** local server + Playwright (playwright-core in the scratchpad, Chromium at `/opt/pw-browsers/chromium-*/chrome-linux/chrome`).
  - Serve: `python3 -m http.server 8199 --bind 127.0.0.1`.
  - **Kill the server with `fuser -k 8199/tcp`. NEVER `pkill -f "http.server 8199"`** — that pattern matches the shell's own command line and kills the shell (exit 144).
  - Test scripts live in the scratchpad (`carshot2.js` = the carousel screenshotter; edit its `STEP/CYCLE/SHIFT/LEADIN` constants to match `dims()` when they change).
  - Tests use an **iPad/mobile UA (coarse pointer)** so `SKIP_INTRO` reveals content instantly and the mofid panel auto-builds.
  - **Scroll math:** use `getBoundingClientRect().top + window.scrollY` for absolute page position, **not `offsetTop`** (`#work` is positioned, so `offsetTop` is relative to it).
  - Owner's monitor: **27" 2K = 2560×1440** — test the carousel at that size.

## 11. Other notable prior work (done)

- Mofid frames converted PNG→JPEG (size). Git history was shrunk once (owner approved).
- Mobile fixes: hero byline alignment, gallery overlap removed, hamburger text visibility, contact photo sized down + centered layout, reel/subtext spacing.
- Click-echo effect: blue echo on bright surfaces.
- Homepage image collage reverted to original per owner.

## 12. Pending / deferred (open items)

- **Four text-free thumbnails need image files added to the repo** (owner must supply — can't paste-to-disk): `media/mofid/pre-rolls/atlas-pre-roll-edited-2.jpg`, `.../bakhshi-pre-roll-edited.jpg`, `.../daroono-pre-roll-with-sound.jpg`, `media/mofid/exhibition-explainer-motion/safe-motion-expo-ready.jpg`.
- **Personal-project collaborators** not provided (most still use `DEFAULT_CREDITS`).
- Fine-tuning of carousel ball/text exact positions was deferred earlier; balls since removed.
- Real-3D (WebGL) gallery: rejected for now; revisit only if owner asks (reference `carousel-css-backup` + the reverted `f50c4a3`).

## 13. Model identity note

The model ran "undercover": configured id `claude-opus-4-8`. **Do not put the model id or session URL into anything pushed to the repo** beyond the established commit trailer.

---
*Keep this file updated as the project evolves.*
