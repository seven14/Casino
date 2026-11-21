# Performance report — Casino (WebGPU)

## Bundle
| | bytes |
|---|---|
| `index.html`, minified, single self-contained file | **93,293 B (91.1 KB)** |
| gzip -9 | 34.5 KB |
| brotli | 29.3 KB |
| of which the inlined bot Web Worker | 8.7 KB |

The bundle contains no image, model, font or audio files. Card faces, court cards, badges and the felt emblem are drawn with Canvas2D at startup into a 2048² atlas with hand-built mipmaps. The table (felt, padded leather rail, wood band, lamp light) is a WGSL shader. All sounds are synthesized with Web Audio.

## Test setup and caveat
- Headless Chrome 153 with WebGPU on **SwiftShader**, Google's CPU software rasterizer. This sandbox has no GPU.
- 1280×720, DPR 1, watch mode (4 bots, Easy/Medium/Hard/Medium), real-time 1×, 150 s, 2,528 frames.
- No profiler attached during the timing run. CDP heap polling and forced GCs were found to inject their own 90+ ms pauses, so they were excluded.
- **The frame interval is bound by the CPU software rasterizer and is not representative of Windows 11 + Chrome on a real GPU.** The JavaScript cost per frame, allocation behaviour and memory profile *are* representative.

## Frame timing (ms)
| metric | p50 | p95 | p99 | p99.9 | max |
|---|---|---|---|---|---|
| rAF frame interval (SwiftShader-bound, ~17 fps here) | 66 | 83 | 116 | 119 | 167 |
| **JS per frame: game logic, scheduler, tweens, instance and uniform fill** | **<0.1** | **0.1** | **0.8** | **7.3** | **8.6** |

- 85.6% of frames spend under 0.1 ms in JS; 99.05% under 1 ms; 99.5% under 4 ms.
- The few multi-millisecond frames are event frames: a move resolving (legality check, invariant check, layout, move-log update), a deal, or a summary dialog. The worst, 8.6 ms, was a move resolution. Even that fits a 16.7 ms frame alongside GPU work.
- The GPU work per frame is small:
  - one full-screen textured triangle (the table is pre-shaded into a texture on resize);
  - 72 instanced shadow quads and 72 instanced card or badge quads, from a **pre-recorded render bundle**;
  - two `writeBuffer` calls (5.8 KB of instance data and 256 B of uniforms).
- Bot thinking runs in a **Web Worker**, so the Hard bot's determinized lookahead never runs on the render thread.

### Spikes found and removed during verification
| cause | before | after |
|---|---|---|
| Auto-continue in watch mode created the AudioContext and built the reverb (it went through the click handler) | 98–117 ms frame | removed: audio only starts on a real user gesture |
| `offsetWidth` read while placing nameplates forced a synchronous layout | part of the above | removed: widths are estimated from text |
| Cold JIT on the first moves (move generation and capture search) | ~10–25 ms | rules engine and frame path pre-warmed behind the loading screen |
| `legalMoves` on a crowded 12-card table | 26 ms | 2.5 ms (subsets precomputed, cheap rejects) |

## Per-frame allocations
- **Design:** all per-frame state lives in preallocated typed arrays:
  - card transforms, tweens, glows, the instance buffer and uniforms;
  - scalars in a `Float64Array`, because writing doubles to module-level `let`s or object fields would allocate a HeapNumber every frame;
  - scheduler timers in a growable `Float64Array`.

  No objects, arrays, closures or strings are created in the frame loop, and the render-pass descriptor and submit array are reused.
- **Measured** with Chrome's sampling heap profiler (64-byte sampling) in live 4-bot play:
  - **steady state** (after 60 s): **0.9 bytes/frame** attributed to the frame loop.
  - **from startup** (4–24 s): about 3–6 bytes/frame while V8 moves the code through its tiers. The JIT is pre-warmed to keep this window short.
  - The residual is the WebGPU API's own per-frame wrappers (`GPUCommandEncoder`, `GPURenderPassEncoder`, `GPUCommandBuffer`). A bare WebGPU clear loop measured the same order of bytes. These die young and are collected by the scavenger with no visible pause.
- Event frames (moves, deals, dialogs) allocate a few KB per move by design. This is bounded and collected.

## Long-run stability design
- **Time.** No absolute clocks are used. Tweens and timers advance by clamped frame delta (≤0.1 s), and the shader clock wraps at 1000 s. The highlight pulse runs at exactly 1.2 Hz (1200 cycles per 1000 s), so the wrap is seamless.
- **Positions** are recomputed from game state on every event, so they never accumulate. Yaw is normalized toward its target on every tween.
- **Memory.** DOM content is bounded and recycled: the log keeps 5 lines, nameplates are rebuilt per match, and dialogs are replaced. Statistics buffers are capped, and worker requests are deleted on reply.
- **Photosensitivity.** There are no full-screen flashes. Highlight pulses are 1.2 Hz with ±18% amplitude. The sweep effect is a single low-contrast ring expanding over about 1.4 s, plus a 2.2 s floating label. The active-player light is a slow, soft spotlight.

## Audio check (procedural Web Audio, offline-rendered and measured)
| sound | peak | RMS | audible length (s) |
|---|---|---|---|
| shuffle (two riffles, bridge, two taps) | 0.23 | 0.013 | 2.5 |
| deal | 0.14 | 0.004 | 1.1 (incl. reverb tail) |
| place / gather / build | 0.07–0.14 | 0.003–0.004 | ~1.0 |
| sweep chime (inharmonic bell arpeggio) | 0.49 | 0.054 | 1.95 |
| win fanfare / loss motif | 0.33 / 0.17 | 0.035 / 0.020 | 1.9 / 1.7 |
| tutorial guide / hint | 0.11 / 0.09 | 0.009 / 0.007 | 1.3 / 1.0 |
| turn / refusal | 0.08 / 0.07 | 0.006 / 0.003 | 1.1 / 0.8 |

All 12 sounds are non-silent and well below clipping. The signal path is a shared bus, then a procedurally generated stereo reverb impulse, then a compressor.

## Not measurable here
True GPU frame time and vsync pacing on Windows. Run the file in Chrome on the target machine; the JS budget above leaves about 16 ms per frame for the GPU, against a very light GPU load.
