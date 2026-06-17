# Jimdependance — Design Brief for Claude (Drawings & Animations)

## 1. What this is
**Jimdependance** is a single-file, offline-first **workout PWA** (`index.html`, no build step,
no framework — vanilla HTML/CSS/JS, data in `localStorage`). It's a personal training app:
sunrise/cardio in the morning, a shoulder-safe strength program, weight tracking, warm-up /
cool-down stretches, and a map-free run/walk tracker.

The whole app must keep working as **one self-contained `index.html`** (it's deployed both inside
the Claude app via `window.storage` and on Vercel via `localStorage`). No external JS/CSS
dependencies beyond the two Google Fonts already linked. Everything you draw or animate must be
**inline SVG + CSS** (and a little vanilla JS where motion needs state). No images, no Lottie
runtime, no canvas libraries.

## 2. Who it's for / the feel we want
A single committed user training mostly at home and while travelling. The vibe today is
"premium dark gym app": near-black background, **lime accent**, condensed display type. We want
to push it from "clean but static" to **"feels alive and considered"** — confident, athletic,
fast. Think Nike Training Club / Whoop / Strava polish, not playful/cartoonish.

## 3. Hard constraints (read before designing)
- **Mobile-first, single column, `max-width:480px`.** It's a phone app. Design for ~390px wide.
- **One file, no dependencies.** Inline SVG + CSS only. No new fonts, no asset files, no libs.
- **Performance:** must stay smooth on a mid-range phone. Animate only `transform` and `opacity`.
  Avoid layout-thrashing properties (width/height/top/left) in anything that runs per-frame.
- **Respect `prefers-reduced-motion`.** Every non-essential animation must have a reduced-motion
  fallback (instant or cross-fade). This is a workout app used at 6am — motion must never get in
  the way of tapping a "set complete" button.
- **PWA / offline:** no network calls for motion or art.
- **Touch ergonomics:** primary actions are big and bottom-reachable; don't let decorative motion
  block or delay taps. Animations on interactive elements must not increase perceived latency.

## 4. Existing design system (match this — don't reinvent)
Pulled from the current `:root` and stylesheet. **Reuse these tokens.**

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0a0b0d` | app background |
| `--panel` / `--panel-2` | `#121418` / `#181b21` | cards / insets |
| `--line` | `#23272f` | borders |
| `--txt` / `--muted` / `--muted-2` | `#f3f4f2` / `#7f8794` / `#565d68` | text tiers |
| `--accent` / `--accent-dim` | `#d7ff3e` / `#9fbe2a` | lime accent |
| `--accent-ghost` | `rgba(215,255,62,.12)` | selected/active fills |
| `--sky` | `#0075FF` | cardio/secondary accent |
| `--ok` / `--danger` | `#39d98a` / `#ff5a4d` | success / error |
| `--radius` | `18px` | card radius |

- **Display type:** `Bebas Neue` (numbers, titles, big stats). **Body:** `Sora`.
- Existing motion vocabulary to stay consistent with: `rise` (screen enter, 10px up + fade),
  `slideup` (sheets/takeovers, cubic-bezier(.2,.8,.2,1)), `pop` (finish ring, springy
  cubic-bezier(.2,1.4,.4,1)), `fade`. **Build on this language; don't introduce a clashing one.**
- There's already a faint film-grain overlay and a top-right lime radial glow. Keep that texture.

## 5. The two jobs

### JOB A — Drawings (exercise & stretch diagrams)
**Current state:** exercise diagrams are procedurally generated stick figures (`figSVG` in the
DIAGRAMS section, ~line 306). Each is a 220×130 SVG: a lime end-pose skeleton, a grey "ghost"
start pose, and a dashed motion arrow on one joint. ~30 poses mapped in `DIA`, keyed by
`EXDIA[exerciseId]`. They're functional but crude (single-width limbs, no body mass, stiff).

**What we want:**
1. **Better static figures.** Same procedural, data-driven approach (a pose = joint coordinates,
   so all ~40 exercises stay cheap to define) but more readable as a *human*: tapered limbs,
   a sense of torso/chest/pelvis volume, clearer hands/feet, better foreshortening on the hard
   ones (pushup, plank, bird-dog, dead-bug, mountain climber). Keep the lime-on-near-black look
   and the ghost-start + arrow convention — it communicates the *movement*, which is the point.
2. **Animated diagrams (the headline ask).** In the workout takeover (`renderWorkout`, ~line 711)
   and the exercise-detail sheet (~line 516), the diagram should **loop the actual rep**:
   start pose → end pose → back, slowly, on a gentle ease. Implement as a single SVG whose joint
   lines animate between the two keyframes (CSS or SMIL or a tiny rAF tween — your call, but it
   must be driven from the same pose data so we don't hand-author every frame). Subtle, ~2–3s
   loop, pausable, and **frozen to the static start/end under reduced-motion.**
3. Keep the **arrow + ghost** as the fallback/static representation (thumbnails, lists).

Deliver this as a clean, documented drop-in replacement for the DIAGRAMS section so adding a new
exercise stays "define joint coords for start + end, done."

### JOB B — Animations & micro-interactions (whole-app polish)
Make the app *feel* responsive and rewarding. Priority order:

1. **Workout flow (highest value):**
   - **Set completion:** tapping the check on a `.setrow` should feel great — a satisfying
     check-draw + row settle + subtle haptic-style scale. Right now it just recolors.
   - **Rest timer** (`.rest`, `.clock`, ~line 735): the 110px countdown deserves a real treatment
     — a circular progress ring draining around the number, calm pulse, clear "almost done" state
     in the last 3s. Smooth, not anxious.
   - **Progress bar** (`.to-progress .pf`) between exercises: make advancing feel like momentum.
   - **Finish screen** (`.finish`, ~line 151): the springy ring exists — elevate it into a proper
     "session complete" celebration (ring draw-on, stats count-up, restrained confetti/streak
     beat). This is the emotional payoff; make it land. Reduced-motion: clean static version.

2. **Navigation & structure:**
   - Tab bar (`.tabs`/`.tab`): animate the active state (icon + lime indicator that slides
     between tabs) instead of a hard color swap.
   - Screen transitions: the `rise` enter is fine; consider directionality between tabs if cheap.
   - Bottom sheets / takeovers already slide up — make sure dismiss is equally smooth.

3. **Data & feedback:**
   - **Weight chart** (`.wchart`, `screenWeight` ~line 762) and **Progress** stats
     (~line 810): line/area draw-on, points that pop in, stat numbers that count up on view.
   - **Streak pill** (`.streakpill`): a small flare when the streak increments.
   - **Sunrise card** (`.sun`): a gentle ambient sun-glow drift to make the morning screen feel
     warm and alive.

4. **Micro-interactions everywhere:** button press depth, toggle knob spring (`.toggle`),
   option/segment selection (`.opt`, `.segbtn`, `.goalchip`), chip appearance. Consistent
   timing/easing tokens so it feels like one product.

## 6. Deliverables
1. **An upgraded `index.html`** (or a clearly-scoped patch to it) implementing the above, keeping
   the single-file/no-dependency rule.
2. **A short motion spec** at the top of the relevant section: the easing + duration tokens you
   standardized on, and the reduced-motion strategy.
3. **Updated DIAGRAMS section** that's still data-driven and documented for adding exercises.
4. Keep all existing functionality intact (storage, wake-lock, crash recovery, travel/home modes,
   shoulder-safe program). **Don't regress behavior to add polish.**

## 7. Acceptance criteria
- Runs as a single `index.html` with no new external dependencies; works offline.
- Smooth (~60fps) on a mid-range phone; only `transform`/`opacity` animate per-frame.
- Every non-essential animation honors `prefers-reduced-motion`.
- Exercise diagrams read clearly as human movement and (in workout view) animate the rep.
- The workout → rest → finish loop feels rewarding end to end.
- Visual language stays consistent: lime accent, Bebas/Sora, existing radii/tokens.
- No added input latency on primary actions (set complete, next, start).

## 8. Suggested priority if scope must be cut
1. Animated exercise diagrams + better static figures (Job A) — this is the signature feature.
2. Workout flow polish: set complete, rest timer ring, finish celebration.
3. Tab bar + screen/sheet transition polish.
4. Charts/stats draw-on & count-up.
5. Ambient/decorative touches (sun drift, streak flare).
