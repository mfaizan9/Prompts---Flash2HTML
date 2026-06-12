# Flash → HTML5 Conversion Instructions for Astronomy Education Simulations

You are an LLM coding agent (e.g., Codex, Claude, etc.). Your job is to convert a legacy Adobe Flash astronomy simulation (provided as a `.fla` source file and/or a `.swf` compiled file) into a self-contained, ADA-Title-II-compliant HTML5 simulation that behaves the same as the original. These instructions are generic and must work for any of ~100 simulations in this collection. Follow every phase in order. Do not skip iteration.

---

## 0. Mission summary

For each simulation you are given a folder containing some subset of:

- `<name>.fla` — Adobe Animate source (XFL inside a ZIP-like container, or binary)
- `<name>.swf` — compiled Flash movie
- `<name>.note` — short text describing the most recent author change
- Optional `<name>.html` — a prior, possibly incomplete HTML5 port
- Optional asset files (PNG, MP3, XML, etc.)

You must produce a working HTML5 version (HTML + CSS + JS, no Flash, no Java applets, no proprietary runtime) that:

1. Replicates the **full functional behavior** of the original Flash simulation.
2. Preserves the **original visual style and layout** as closely as practical on the modern web.
3. Meets **ADA Title II digital accessibility requirements** (WCAG 2.1 Level AA as the baseline target, AAA where reasonable).
4. Has been **iteratively compared** against the original Flash content emulated via Ruffle until functional parity is reached.
5. Is **self-contained** — runs from a local file or static host with no build step, no server, and no network dependencies.

Treat parity with the original Flash behavior and ADA compliance as **equal-priority hard requirements**. If they ever conflict, prefer accessibility but document the deviation in `CONVERSION_NOTES.md`.

---

## 1. Required tooling

Before starting, confirm (or install) access to the following. If any tool is missing, stop and report which.

| Tool | Purpose |
|---|---|
| **Ruffle** (desktop or `ruffle-web` selfhosted) | Emulate the original `.swf` to observe ground-truth behavior. |
| **JPEXS Free Flash Decompiler (FFDec)** CLI | Extract ActionScript, shapes, timeline frames, embedded text, and assets from `.swf`. |
| **`unzip`** + a text editor | A `.fla` saved as XFL is just a zip; you can also try `unzip -o file.fla -d fla_extracted/` to read raw XML timeline data. |
| **A modern browser** (Chrome/Firefox/Safari current) | Render and test the HTML5 output. |
| **axe-core** (CLI or browser ext) or **Lighthouse** | Automated accessibility auditing. |
| **A screen reader** — VoiceOver (macOS), NVDA (Windows), or Orca (Linux) | Manual screen-reader verification. |

If you cannot launch a GUI screen reader from your environment, you must still verify accessibility programmatically (axe-core, semantic HTML review, ARIA inspection) and **state in the deliverable notes that human screen-reader QA is still required**.

---

## 2. Inputs and outputs

### Inputs the agent receives
- A directory whose name is the simulation slug (e.g., `hydrogen_atom/`).
- The Flash source/compiled files listed in §0.
- Possibly a partial HTML5 port to extend rather than replace.

### Outputs the agent must produce in that same directory
- `index.html` — main entry point (rename any pre-existing `<name>.html` if needed, or keep its name if that is the established convention in the repo; check sibling sim folders before deciding).
- `styles.css` — all styling.
- `simulation.js` — simulation logic (split into modules only if size justifies it; prefer one file for simple sims).
- `assets/` — any extracted images, sounds, or data tables that the sim needs.
- `CONVERSION_NOTES.md` — short report of: original Flash behaviors detected, the mapping to HTML5 constructs, deliberate deviations, and remaining open questions.
- `ACCESSIBILITY.md` — explicit list of ADA/WCAG affordances added (alt text, ARIA roles, keyboard map, color choices, screen-reader announcements).

Do not delete the original `.fla`, `.swf`, or `.note` files. They are the reference.

---

## 3. Phase 1 — Discover and catalog the source

1. List the directory contents and identify which files are present.
2. Read `<name>.note` if present — it often describes the most recent meaningful change and hints at the simulation's purpose.
3. Open `<name>.swf` in **Ruffle** and:
   - Record the stage dimensions, frame rate, and background color.
   - Click every interactive control. Note what each does.
   - Move sliders to extremes. Note clamping behavior.
   - Trigger every state transition (e.g., reset, fire, pause, ionize).
   - Watch for animations, sound effects, dynamically generated text, and pop-up dialogs (help/about).
   - Capture screenshots of: initial state, each major state, and any modal dialogs. Save these to `assets/reference/` for the comparison phase.
4. Open `<name>.swf` in **FFDec** (or unzip the `.fla`) and extract:
   - All ActionScript (`.as`) classes and timeline scripts (both AS2 and AS3 exist in this collection — check which).
   - Library symbols (shapes, MovieClips, fonts, embedded text).
   - Text strings (labels, help text, about text).
   - Any embedded XML/JSON data tables (e.g., transition energies, planetary constants).
5. Write a one-paragraph **behavior model** at the top of `CONVERSION_NOTES.md`: what the simulation teaches, what the user can do, what changes on screen in response.

**Stop and re-examine** before moving on if you cannot describe the simulation's purpose in plain language. Guessing here causes most parity failures later.

---

## 4. Phase 2 — Translate Flash constructs to HTML5

Use this mapping as a baseline. Adapt as needed.

| Flash construct | HTML5 equivalent |
|---|---|
| Stage (e.g., 800×600) | A root container with fixed aspect ratio (`aspect-ratio` CSS or `viewBox` on an `<svg>`). |
| MovieClip with vector art | Inline `<svg>` group with `<path>`, `<circle>`, `<rect>`. Prefer SVG over `<canvas>` whenever the art is vector — SVG is keyboard- and screen-reader-friendly. |
| Bitmap asset | `<img>` with descriptive `alt`, or `<image>` inside SVG. Extract via FFDec to `assets/`. |
| Timeline tween | CSS transition/animation, or `requestAnimationFrame` loop driving SVG attributes. |
| `gotoAndPlay` / frame labels | JavaScript state machine. Encode states explicitly; do not emulate the timeline. |
| `onEnterFrame` | `requestAnimationFrame`. |
| Buttons (`SimpleButton`) | `<button>` elements styled to match. **Never** use `<div onclick>` — it breaks keyboard and screen-reader use. |
| Input text fields | `<input>` / `<select>` / `<output>`. |
| Sliders drawn from MovieClips | `<input type="range">` styled to resemble the original, or an SVG slider that implements `role="slider"` with full keyboard handling (ArrowUp/Down, Home/End, PageUp/Down). |
| `Sound` / embedded MP3 | `<audio>` with controls, or `new Audio()` triggered by interaction. Sound must never be the sole channel for information. |
| `LoadVars` / external XML | `fetch()` of a static JSON/XML file in `assets/`, or inlined as a JS constant if small. |
| `trace()` / debug output | Browser `console.log` during development; remove or gate behind a debug flag before delivery. |
| Modal "help" / "about" popups | `<dialog>` element (preferred) or an ARIA `role="dialog"` overlay with proper focus trap and Escape-to-close. |

### Numerical fidelity
- Preserve all physical constants, formulas, and numeric ranges **exactly** as they appear in the ActionScript. Astronomy sims often hinge on specific values (e.g., 13.6 eV, Rydberg constant, AU, etc.). Copy them verbatim; do not "round to nicer numbers."
- Preserve unit labels, sig figs, and scientific notation formatting (e.g., `1.11 x 10^15 Hz`) as the original displays them.

### Layout fidelity
- Match the **panel structure** of the original (e.g., diagram on top, controls below, log on the side). Use CSS Grid or Flexbox.
- Match the **color palette** as closely as possible **except** where the original colors fail contrast or colorblind requirements — see §6.
- Match the **font family** with a free web-safe equivalent if the original used a proprietary font; otherwise use `system-ui` / a sans-serif stack. Note the substitution in `CONVERSION_NOTES.md`.

---

## 5. Phase 3 — Implement the HTML5 version

Write the code with these rules:

1. **Semantic HTML first.** Use `<main>`, `<section>`, `<header>`, `<nav>`, `<aside>`, `<button>`, `<label>`, `<output>`. Reach for ARIA only when no native element exists.
2. **One single page.** No build tooling, no bundlers, no frameworks unless the original complexity truly demands it. Vanilla JS modules (`<script type="module">`) are fine.
3. **No external network requests at runtime.** All assets are local. No CDN fonts, no analytics, no Google Fonts. If a font is needed, self-host it (with an open license) under `assets/fonts/`.
4. **Deterministic state.** Every visual element should be derivable from a small, named JavaScript state object. Re-render from state after every user action so the screen-reader live region and visuals stay in sync.
5. **Keyboard parity with mouse.** Every action the mouse can perform must be reachable by keyboard alone, with visible focus indicators (`:focus-visible` outlines, not `outline: none`).
6. **Live region for changes.** Provide an `aria-live="polite"` summary (e.g., "Electron now on level 3. Photon energy 12.09 eV matches the Lyman-beta transition.") that updates after each meaningful state change. Throttle so it does not spam during continuous slider movement — announce on release, not on every tick.
7. **Respect `prefers-reduced-motion`.** If the user has reduced-motion set, replace continuous animations with instantaneous state changes (still showing the result) and keep audio cues intact.
8. **Pause and reset.** Even if the original didn't have them, expose **Pause** and **Reset** controls. Long-running motion without a pause violates WCAG 2.2.2.

---

## 6. Phase 4 — ADA Title II / WCAG compliance

ADA Title II for state and local government digital content (effective 2026) requires **WCAG 2.1 Level AA** as the technical standard. Apply every item below:

### Text alternatives (WCAG 1.1.1)
- Every `<img>` has `alt`. Decorative-only images get `alt=""`.
- Every `<svg>` used to convey information has `role="img"`, an `<title>`, and a `<desc>` linked via `aria-labelledby`/`aria-describedby`. Decorative SVGs get `aria-hidden="true"`.
- Charts, diagrams, and equations have a text equivalent reachable by screen readers — either inline, or via a "Describe this diagram" disclosure.

### Color and contrast (WCAG 1.4.1, 1.4.3, 1.4.11)
- Text and meaningful UI: ≥ **4.5:1** contrast against background (3:1 for large text and graphical objects). Verify with a contrast checker.
- **Never encode information by color alone.** If the Flash original used red/green to distinguish states, add a shape, label, or pattern as a second channel.
- Use a **colorblind-safe palette**. A safe default is the Okabe–Ito 8-color set: `#000000, #E69F00, #56B4E9, #009E73, #F0E442, #0072B2, #D55E00, #CC79A7`. Spectral / wavelength visuals (a common astronomy case) must remain physically meaningful — keep the spectrum but supplement with numeric or textual labels (e.g., "656 nm — H-alpha (red)").
- Honor `prefers-color-scheme` if the original had no strong stylistic reason to lock to one theme.

### Keyboard (WCAG 2.1.1, 2.1.2, 2.4.7)
- Every control reachable in a logical tab order.
- No keyboard traps (modals must trap focus *while open* and release it on close).
- Visible focus ring on every interactive element.

### Timing / motion (WCAG 2.2.2, 2.3.3)
- No auto-play of motion longer than 5 s without a pause/stop control.
- No flashing > 3 times per second.

### Forms and labels (WCAG 1.3.1, 3.3.2)
- Every input has a `<label>`. `aria-label` only when a visible label is impossible.

### Screen-reader specifics
- The page has a single `<h1>` matching the simulation name.
- Heading hierarchy is correct (no skipping levels).
- An `aria-live` region announces simulation state changes.
- Complex custom controls (e.g., a dial) implement the correct ARIA pattern (`role="slider"`, `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, `aria-valuetext`).
- Test with at least one of VoiceOver / NVDA / Orca. Log the test in `ACCESSIBILITY.md`.

### Document language
- `<html lang="en">` (or correct locale) is set.

---

## 7. Phase 5 — Iterative comparison against Ruffle

This is the most-skipped and most-important phase. **Loop until parity is reached.**

For each iteration:

1. Open `<name>.swf` in Ruffle in one window.
2. Open the new `index.html` in a browser in another window of the same size.
3. Walk a **comparison checklist** — derived from the behavior catalog in §3 — performing the same action in both windows and comparing the result:
   - Initial state matches (positions, colors, labels, default values).
   - Every interactive control produces the same effect.
   - Every numeric readout matches to the displayed precision.
   - Animations begin, progress, and end at the same logical moments.
   - Sounds (if any) play at the same triggers.
   - Edge cases match (minimum/maximum slider, repeated firing, ionization, reset mid-animation).
   - Help/About dialog content matches the original's text exactly (verbatim copy from the Flash source — do not paraphrase educational text).
4. Record every discrepancy in a `discrepancies` list (in `CONVERSION_NOTES.md`, can be temporary).
5. Fix discrepancies. Prefer fixing the **simulation logic** to match Flash. Only adjust the original-style visuals when accessibility requires it.
6. Re-run the comparison. Repeat until the discrepancies list is empty **or** every remaining item is a documented, justified accessibility-driven deviation.
7. Run automated accessibility audit (`axe`, Lighthouse). Resolve every Critical and Serious finding. Document any remaining Moderate/Minor with rationale.
8. Re-walk the comparison once more after each accessibility fix — accessibility refactors sometimes change focus or rendering in ways that break parity.

Do not declare the job done after a single pass. **Three iterations is a typical minimum.** Stop only when both checklists (parity + accessibility) are clean.

---

## 8. Phase 6 — Final verification checklist

Before declaring the conversion complete, confirm each of the following. If any item is "no", return to the appropriate phase.

**Functional parity**
- [ ] Every interactive control in the Flash version exists and behaves equivalently.
- [ ] All numeric outputs match the Flash version's values and formatting.
- [ ] All on-screen text (labels, help, about) is preserved verbatim.
- [ ] Animations, sounds, and state transitions trigger on the same events.
- [ ] Reset returns the simulation to the exact initial state shown in Ruffle.

**Visual fidelity**
- [ ] Panel layout, proportions, and grouping match the Flash original.
- [ ] Color palette is recognizable as the original or its colorblind-safe equivalent.
- [ ] Typography matches or is documented as substituted.

**ADA / WCAG 2.1 AA**
- [ ] All images and informative SVGs have text equivalents.
- [ ] All text meets 4.5:1 contrast (3:1 for large/graphic).
- [ ] No information conveyed by color alone.
- [ ] Full keyboard operability with visible focus.
- [ ] Live region announces state changes.
- [ ] Pause/reset available; reduced-motion respected.
- [ ] Screen reader (or programmatic equivalent) walked end to end.
- [ ] axe-core / Lighthouse: zero Critical/Serious issues.

**Hygiene**
- [ ] No console errors or warnings on load or interaction.
- [ ] Works offline (no network requests).
- [ ] Works in current Chrome, Firefox, and Safari.
- [ ] `CONVERSION_NOTES.md` and `ACCESSIBILITY.md` are complete.

---

## 9. Style and engineering conventions

- **No frameworks unless justified.** Vanilla HTML/CSS/JS keeps the simulation portable for decades, which matches the longevity needs of educational content.
- **Single source of truth for state.** A plain object updated by reducer-style functions is sufficient.
- **Avoid clever code.** These sims will be maintained by educators and student devs. Favor obvious code over short code.
- **Comment the physics, not the syntax.** A short comment naming the formula (e.g., `// E_n = -13.6 / n^2 eV`) is worth far more than a comment explaining a `for` loop.
- **Don't invent features** the Flash original didn't have, except: pause, reset, accessibility affordances, and `prefers-reduced-motion` handling. Anything else needs a note in `CONVERSION_NOTES.md` and ideally a flag to disable it.

---

## 10. Common pitfalls (read before starting)

- **AS2 vs AS3.** Older sims may be ActionScript 2. Variable scoping and event model differ. Confirm version from FFDec's report.
- **Frame-rate-dependent physics.** Flash often ran at 24 or 30 fps and tied motion to `onEnterFrame`. Use elapsed-time-based integration in JS (`dt = now - then`) rather than per-frame deltas, or your animations will drift across machines.
- **Hidden modal layers.** Many of these sims hide "help" and "about" content as off-stage MovieClips. Extract their text from the SWF library, not just from what is visible in Ruffle.
- **Trigonometric conventions.** Flash uses degrees in some places (rotation) and radians elsewhere. Re-check.
- **Y-axis direction.** Flash Y grows downward, same as DOM, so this usually transfers cleanly — but watch for trigonometric inversions in orbital motion code.
- **Embedded fonts.** Flash often subsetted fonts. If text appears differently sized in the port, it is almost always a font-metric issue, not a logic bug.
- **"Looks the same in Ruffle" ≠ "looks the same in Flash Player."** Ruffle's rendering is excellent but not perfect. If the `.fla` source is available, the source is the higher authority.
- **Don't let accessibility refactors silently change behavior.** When you add a `role="slider"` to an SVG element, re-walk the parity checklist; ARIA changes can shift focus order and keyboard semantics in ways that affect what the user sees.

---

## 11. How to ask the user for help

If during conversion you encounter:
- ambiguity in the original behavior that Ruffle and the source disagree on,
- text content that appears truncated or placeholder-ish in the Flash source,
- assets that fail to extract,
- a physics formula whose constants you cannot find in the ActionScript,

**stop and ask** rather than guessing. Educational correctness is non-negotiable.

---

## 12. Definition of done

The conversion is complete when:

1. A second pass through Phase 5 yields zero new discrepancies.
2. Every box in §8 is checked.
3. `CONVERSION_NOTES.md` and `ACCESSIBILITY.md` exist and accurately reflect the final state.
4. The simulation runs offline from `index.html` in a current browser with no console errors.
5. A screen-reader walkthrough (or a documented programmatic equivalent) confirms a usable experience without sighted use.

Only then report the simulation as converted.
