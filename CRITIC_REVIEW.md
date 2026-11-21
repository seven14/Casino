# Critic review — Casino (WebGPU demo)

The critic was a separate agent that saw **only the running game**. It played and watched through a driver that gave it mouse and keyboard input, screenshots, and the on-screen text (status line, move log, dialogs, nameplates). It never saw source code, the DOM or the developer's reasoning. It judged rule adherence, build combinations, card movement, play pacing, and table layout and styling, and returned prioritized defects. We iterated until it reported nothing above **minor**.

## Gate result: PASSED (round 5)
> **Verdict (round 5, verbatim summary):** "nothing above minor remains. The moderate watch-mode Menu/Resume issue is fixed. The minor counter timing issue is only partly fixed, and the sweep callout still needs to be seen on screen."

## How the review converged
| Round | Critical | Major | Moderate | Main themes |
|---|---|---|---|---|
| 1 | 0 | 5 | 8 | Multiple builds indistinguishable; stale "Shuffling…" status; small build badges and stacked cards; log overlapping the table; nameplates clipped at 1024×768 |
| 2 | 0 | 5 | 4 | Silent drops; crowded pips; tutorial lacked sweep and raise practice; anonymous capture piles; status wording |
| 3 | 0 | 1 | 4 | Off-turn and wrong-tutorial drops gave no feedback; pile badge overlap; no scoring lesson; lingering coach text; brief watch summaries |
| 4 | 0 | 0 | 1 | Watch-mode menu had no Resume and Escape didn't close it |
| **5 (final)** | **0** | **0** | **0** | Minor only |

Rules were never at fault: in every round the critic hand-checked the move-log narration and the round-summary arithmetic (52 cards, 13 spades, 4 aces, 11 points plus sweeps) and found them correct.

## Final review (round 5)

### Re-verification of round-4 findings
| # | Finding | Result |
|---|---|---|
| 1 | (Moderate) Watch-mode menu had no Resume, and Escape didn't close it | **Fixed.** "Resume watching" is the highlighted button, and Escape returns to the same game. |
| 2a | Deck counter showed the post-deal count during the shuffle | **Fixed for the shuffle.** No counter appears while shuffling. |
| 2b | Counters ran ahead of the animation | **Still present (minor).** The deck count and "N taken" update when a move commits, slightly before the cards land. |
| 2c | Right-seat hand sat under its nameplate | **Fixed.** |
| 2d | Sweep moment was muted | **Not confirmed by the critic** (no sweep was on screen during its runs). The log line "— SWEEP!", the ★ on the nameplate, and the summary's SWEEPS column were all seen. A "Sweep! +1" floating label and a gold ripple on the felt were added and checked separately. |
| 2e | Watch-mode menu showed the wrong difficulties | **Fixed.** The selectors are labelled "for a new game". |

### Sanity pass, as observed by the critic
- **Deal order:** two cards at a time, starting at the dealer's left, with pairs to each player, the table and then the dealer. Later deals give 4 each and none to the table.
- **Play mode:** a capture by the player works. An ambiguous drop opened the chooser ("Build 8 (A♦ + 7♣) / Multiple build of 8 (A♦ + 7♣ + 8♠) / Cancel"). Illegal drops are refused with a clear message.
- **Tutorial:** clear coach steps; the guided moves behave as described.
- **Watch mode:** round-summary arithmetic is correct, and match totals carry over.
- **1024×768:** the layout fits and nothing important is clipped.
- Every logged capture and build was legal.

### Remaining minor items
1. Deck and "taken" counters update when a move commits, slightly before the cards land.
2. A side nameplate with a long label (e.g. "11 taken · ★") runs close to that player's nearest hand card.
3. Starting a second drag while the build chooser was open cancelled it. This is fixed after the review: the chooser is now modal and the status line says "Pick one of the options, or Cancel".
4. The "Sweep! +1" label was never captured on screen by the critic. It is a 2.2 s CSS animation, which keeps running while the driver pauses game time for a screenshot.

### Not covered by the critic
- 2- and 3-player table layouts. These were exercised by the rules simulator and by the tutorial (2 players).
- Sound. The critic had no audio; see the audio check in `PERF_REPORT.md`.
