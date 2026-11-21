# Soak report — Casino (WebGPU)

## Gate: ten 4-player games with mixed bot difficulties — PASSED (final build)
Settings: headless Chrome 153 + WebGPU (SwiftShader), 1280×720, watch mode, `?auto=watch&speed=10&games=10&mute=1&seed=2026`. Partnerships and every-player-for-themselves alternated. A post-GC heap sample was taken every 10 s.

| Game | Difficulties (seats 1–4) | Mode | Final score |
|---|---|---|---|
| 1 | Easy · Medium · Hard · Medium | partners | 26–20 |
| 2 | Hard · Hard · Medium · Easy | free-for-all | 24–14–16–6 |
| 3 | Medium · Easy · Medium · Hard | partners | 23–10 |
| 4 | Hard · Medium · Easy · Hard | free-for-all | 10–12–3–30 |
| 5 | Easy · Hard · Hard · Medium | partners | 7–26 |
| 6 | Easy · Medium · Hard · Medium | free-for-all | 3–13–23–8 |
| 7 | Hard · Hard · Medium · Easy | partners | 25–8 |
| 8 | Medium · Easy · Medium · Hard | free-for-all | 25–7–5–19 |
| 9 | Hard · Medium · Easy · Hard | partners | 22–13 |
| 10 | Easy · Hard · Hard · Medium | free-for-all | 7–24–13–15 |

| Check | Result |
|---|---|
| Games / rounds / moves | 10 / 41 / 1,968 |
| Move mix | 792 captures, 119 builds (29 multiple), 13 raises, 1,044 trails, 22 sweeps |
| Bot moves failing independent legality re-check on the main thread | **0** |
| Invariant violations after every move (52 unique cards; build groups sum to value; every build owner still holds a capturing card) | **0** |
| Round-score errors (points = 11 − tied categories + sweeps) | **0** |
| JS exceptions / page errors | **0 / 0** |
| Post-GC JS heap | 1.55 MB at start → 1.89 MB at end (max 1.97). Plateaus after game 2 (see the leak check below) |
| DOM nodes / event listeners | 293–497 / 31–33, no growth trend |

## Long-run stability: accelerated ~10-hour equivalent
A real 4-hour wall-clock run isn't practical in this sandbox. Instead, a 30-minute run at **20× game speed** covered about **10 hours of play time**. The real-time paths (render loop, shader clock wrap at 1000 s, and so on) still ran for 30 real minutes. This run used an earlier build from before the visual-polish rounds; the rules engine, scheduler and render loop are unchanged since.

| Check | Result |
|---|---|
| Games / rounds / moves | 22 / 93 / 4,458 |
| Builds (multiple) / raises / sweeps | 303 (64) / 27 / 49 |
| Illegal moves / invariant errors / scoring errors / page errors | 0 / 0 / 0 / 0 |
| Post-GC heap | 1.41 → 1.87 MB (max 1.95), plateaued |
| DOM nodes | 323–442, no growth |
| Visual state at the end | identical to the start (screenshot reviewed; no drift, jitter or artifacts) |

## Leak check (heap-snapshot diff, game 2 → game 12, at 20× speed)
Retained heap grew by 119 KB in total, **all of it V8 internals that saturate**:

| Δ bytes | object |
|---|---|
| +47.6 KB | `DoubleStringCache`, a fixed-size number→string cache |
| +40.4 KB | `InstructionStream`, compiled JIT code |
| +12.5 KB | `WeakArrayList` (V8 internal) |
| +11.4 KB | `TrustedByteArray`, JIT metadata |
| < 10 KB each | code/feedback metadata, Blink style data |

No game objects grow: no cards, states, closures, DOM nodes, timers or worker requests.

## Precision and long-run design notes
- Every timer and tween advances by a clamped per-frame delta, with no absolute timestamps to drift. The shader time wraps at 1000 s on an exact multiple of the pulse period.
- Card positions are recomputed from the logical state on each event, never accumulated, and angles are normalized.
- Game logic is deterministic for a given seed. Every soak of the same seed produced the identical game sequence, which makes regressions reproducible.

## Also exercised outside the soak
- **Rules simulator (Node):** 200 bot matches with 2, 3 and 4 players and partners/free-for-all; 739 rounds and 35,472 moves with 0 illegal moves and 0 invariant violations.
- **Bot strength:** Medium beats Easy 99% of the time, Hard beats Easy 100%, and Hard beats Medium 64% (150 two-player matches each).
- **Tutorial:** automated end-to-end walkthrough of all 27 coached steps, run after every change.
