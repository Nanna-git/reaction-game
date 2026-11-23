# Reflex · Reaction Timer

A reaction-time game with a three.js scene. Wait for the scene to turn green, then click, tap or press Space as fast as you can. Play five rounds, see your average, and try to beat your personal best.

**[Live demo](#)** · ![Screenshot](docs/screenshot.png)

> Replace the demo link and screenshot above once deployed. A short GIF showing the four states (wait, go, too soon, result) works well here.

## Features

- **Full game loop:** idle → wait → go → result, with a "Too soon" state for early presses.
- **Five-round sessions** with a per-round breakdown, a running average and a rating label (Lightning, Excellent, Great, Good, Average, Keep practicing).
- **Personal best** saved in `localStorage`, with a button to reset it.
- **Three input methods:** mouse click, touch, and keyboard (Space or Enter). Esc cancels a round.
- **3D scene** built with three.js that changes with the game state: a glossy icosahedron, a wireframe shell and 18 floating background shapes.
- **Works without WebGL:** if the 3D scene can't load, the game falls back to a flat colored background and stays fully playable.
- **Accessible:** screen reader announcements, keyboard-only play, text cues on every state and reduced-motion support.
- **Responsive:** usable from 320px wide up, with touch-friendly controls.

## How to play

1. Press **Space**, **Enter**, click, or tap the play area to start a round.
2. Wait while the scene shows **Wait…** (red). The delay is random, between 2 and 5 seconds.
3. When the scene turns green and says **Click now!**, press as fast as you can.
4. Pressing before green counts as **Too soon**. The round isn't recorded and you can retry it.
5. After five rounds, you'll see your average and rating. Press again to start a new session.

Switching tabs or windows resets the current round, since a round can't be fair if you can't see it.

### Ratings

| Reaction time | Rating |
|---|---|
| under 200 ms | Lightning |
| under 250 ms | Excellent |
| under 300 ms | Great |
| under 375 ms | Good |
| under 500 ms | Average |
| 500 ms or more | Keep practicing |

After a full session, the rating is based on your average rather than the last round.

## How it works

### The state machine

Game logic lives in a `ReactionGame` class with five states: `idle`, `waiting`, `ready`, `tooSoon` and `result`. It is a pure module with no DOM, no three.js and no timers. The caller owns time: it schedules the random delay, tells the game when the "go" frame is painted, and passes in input timestamps. That keeps the logic deterministic and easy to unit test, and the random source can be injected for tests.

### Timing accuracy

Timing is the most important part of a reaction game, so a few details matter:

- **High-resolution clock.** All timestamps use `performance.now()`, never `Date.now()`.
- **The timer starts on the "go" frame.** The timeout that ends the random delay only *requests* the go transition. The actual switch happens inside a `requestAnimationFrame` callback: the game updates its state, the scene draws the green frame, and only then is the start time read. This avoids starting the clock a frame or more before the player could see green.
- **The stop time comes from the input event.** Handlers read `event.timeStamp` as their first action, so the measurement reflects when the input happened, not when JavaScript got around to handling it. If the browser reports a timestamp outside the expected timebase, it falls back to `performance.now()`.
- **Anticipation guard.** An input timestamped before the go frame was painted is treated as too soon, not as a reaction.
- **Unpredictable delay.** The 2-5 second delay is continuous (not stepped) and drawn from `crypto.getRandomValues`, so there is no rhythm to learn.

**A note on accuracy:** the result measures the time from the start of the green frame to the input event. It can't account for display refresh latency or input hardware latency, so treat times as accurate for comparing your own runs on the same device, and not as lab-grade measurements.

### The 3D scene

- **Hero shape:** a low-poly icosahedron with a glossy clear-coat finish and flat shading, so each facet catches the light as it turns. A faint wireframe shell surrounds it.
- **Background shapes:** 6 spheres, 6 octahedrons and 6 torus rings, drawn as three instanced meshes (three draw calls in total).
- **Environment lighting:** a small procedural "photo studio" of bright panels is baked once into an environment map, giving glossy reflections without loading an HDR file or any three.js addons.
- **State-driven animation:** each state has its own preset for spin, drift, scale, pulse and glow, which the scene eases toward.

| State | Scene behavior |
|---|---|
| Idle | Blue tint, slow rotation |
| Waiting | Red tint, slow pulse, shapes drift slightly inward |
| Ready | **Instant** switch to green, scale burst and a spin kick |
| Too soon | Amber tint with a quick shake |
| Result | Blue tint, with a scale pulse that grows the faster your time was |

The "ready" transition deliberately skips easing, so the go signal is unambiguous on the very first frame. Every other transition is smoothed.

### Performance

- Pixel ratio is capped at 2, and antialiasing is only enabled on lower-density displays.
- The frame loop does not allocate: vectors, colors and matrices are preallocated and reused.
- The render loop stops when the tab is hidden and resumes with a reset clock, so there is no large time jump on return.
- The canvas resizes with a `ResizeObserver`, and the camera adjusts for portrait screens.
- With reduced motion enabled, the scene only re-renders while a color change is settling.
- The DOM overlay updates only on state transitions, never per frame.

### Fallbacks and resilience

- The page checks for WebGL2 before loading three.js. If it's missing, the flat UI is used.
- three.js is loaded with a dynamic `import()`, so a blocked or failed CDN request degrades to the flat UI instead of breaking the page.
- If the WebGL context is lost mid-session, the game switches to the flat UI.
- All `localStorage` access is wrapped in `try/catch` (private browsing, blocked storage and quota errors won't break the game).
- Esc, window blur, `pagehide` and tab visibility changes all cancel an in-progress round safely.

## Accessibility

- The play area is a focusable `role="button"` with a descriptive label, and the page has a skip link.
- The game is fully playable by keyboard (Space or Enter to play, Esc to cancel). Holding a key down is ignored, so it can't count as a second reaction.
- Every state shows a **text cue and a badge** ("Wait", "Go", "Too soon"), so color is never the only signal.
- State changes are announced through ARIA live regions: polite for routine updates, assertive for "Go" and "Too soon". Repeated identical messages are still re-announced.
- The canvas is decorative and hidden from assistive technology.
- A dark scrim behind the on-screen text keeps contrast independent of what the 3D scene is showing.
- Visible focus outlines, 44px minimum control height, and `prefers-reduced-motion` support. With reduced motion, the scene drops spin, shake and bursts and keeps only color changes, and CSS transitions are effectively disabled.

## Design

- **Theme:** dark, with a single cool accent so the state colors carry the experience.
- **Palette:**

  | Role | Hex |
  |---|---|
  | Background | `#0F1419` |
  | Surface | `#1A2129` |
  | Border | `#2A343E` |
  | Text | `#F2F5F7` |
  | Secondary text | `#9AA7B2` |
  | Accent | `#6C8CFF` |
  | Waiting | `#C93A3F` |
  | Go | `#1F9D5C` |
  | Too soon | `#F5A524` |

- **Fonts:** [Inter](https://fonts.google.com/specimen/Inter) for the interface and [JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono) for the millisecond readout and stats, set with tabular numbers so digits don't jitter.

## Design decisions

- **three.js from a pinned CDN version** (`0.170.0`, via an import map) rather than a bundler. This keeps the project a plain static site with nothing to build or install. The trade-off is a network dependency, which the fallback above covers.
- **Progressive enhancement.** The game is fully usable before the 3D scene even starts loading. The scene fades in once ready.
- **One animation loop** drives both the 3D scene and the go transition, so the green frame and the start timestamp stay in sync.
- **Logic separated from rendering.** The state machine doesn't know about the DOM or three.js, which keeps it testable and keeps rendering work out of the timing path.

## Browser support

Any modern browser with WebGL2 gets the full 3D experience. Browsers without WebGL2, or where three.js fails to load, get the flat fallback with identical gameplay.


## Ideas for next steps

- A history chart of recent sessions.
- A global leaderboard with server-side validation.
- Sound cues for the go signal and results.
- A light theme.
