---
ingested: 2026-08-29
source_type: plan
author: Claude Agent
synthesis: done
---

# Datagen UCI options: `Minimal`, `Normalize`, `SoftNodes`

> Implemented 2026-08-29. Bench verified bit-identical (2,811,728 nodes @ depth 13).

## Context

`openbench/Engines/Berserk.json` has advertised a datagen preset with
`both_options: "Threads=1 Hash=8 Minimal=true Normalize=false SoftNodes=true"`
since upstream commit `2fcb732`. Berserk implemented **none** of the three — each
fell through the end of the `setoption` chain in `src/uci.c` to
`printf("Unknown command: %s \n", in)`. A DATAGEN workload launched before this
change would have silently run hard-capped nodes and emitted `/1.58`-normalized,
scale-inconsistent evals.

This is milestone **M1 / WS1** of the NNUE data pipeline plan, and it gates
everything downstream. `genfens` is the other WS1 item and is **not** included
here.

The highest-risk item was `Minimal`: if suppressing intermediate output left
**zero** `info ... score ...` lines, every PGN move comment becomes `unknown` and
the whole workload is worthless. The trap is structural — the post-vote
`PrintUCI` was guarded by `if (bestThread != mainThread)`, which is false in the
single-threaded datagen case, so suppressing the per-iteration prints without
also relaxing that guard would have emitted nothing at all.

Berserk's `Normalize` is print-only (`#define Normalize(s) ((s) / 1.58)`,
`uci.h`) — it does not touch internal eval, so the option cannot change playing
strength.

## Changes

**`types.h`** — `SearchParams` gains `uint64_t softNodes;` beside `nodes`
(0 = disabled). `ThreadData` gains `completedDepth` beside `depth`.

**`uci.h` / `uci.c`** — new globals `MINIMAL` (0), `NORMALIZE` (1),
`SOFT_NODES` (0). `MINIMAL` and `NORMALIZE` are `extern`'d for `search.c`;
`SOFT_NODES` is read only in `ParseGo` and stays file-local, matching
`MULTI_PV` / `MOVE_OVERHEAD` / `PONDER_ENABLED`.

Three `option name ... type check` lines added to `PrintUCIOptions()`, and three
`setoption` branches following the existing boolean pattern. Matching is
`strncmp` against hardcoded prefix lengths — 29 for `Minimal`, 31 for `Normalize`
and `SoftNodes`. All three echo an `info string set ...` confirmation like every
other option; `Minimal` is scoped to search verbosity only.

`ParseGo` splits the node limit:

```c
  Limits.nodes     = nodes;
  Limits.softNodes = 0;

  if (Limits.nodes && SOFT_NODES) {
    Limits.softNodes = Limits.nodes;
    Limits.nodes     = Limits.softNodes * 100;
  }
```

`hitrate` stays derived from the *hard* limit — it is the `CheckLimits`
countdown, so the coarser value is correct for the 100× budget.

**`search.c`** — four edits:

- `MINIMAL` added to the two intermediate `PrintUCI` guards (aspiration
  fail-high/low, and the end-of-MultiPV-line print). Nothing else in the engine
  changes behaviour under `Minimal`: the `info string time ...` dump, the
  `currmove` lines, the setoption confirmations, and `bestmove` all still print.
- The post-vote print becomes `if (bestThread != mainThread || MINIMAL)`, with
  the thread-swap left under its original condition. `MainSearch()` is the sole
  path to `bestmove` and always runs to completion, so the line count is
  structurally 1.
- Soft node limit in the ID loop, after the mate check (already main-thread-only
  by the `if (!mainThread) continue;` above it):

  ```c
  if (Limits.softNodes && NodesSearched() >= Limits.softNodes)
    break;
  ```

  A plain `break` matches the adjacent mate and soft-TM breaks. The hard cap
  needed no new code — `CheckLimits` already tests `Limits.nodes`.
- `PrintUCI` gains `!NORMALIZE ||` on the existing TB-score branch of the
  `printable` ternary. Mate scores still convert to mate distance. `eval.c` and
  `bench.c` keep dividing by 1.58 unconditionally.
- `PrintUCI` takes the depth to report as a parameter instead of reading
  `thread->depth`, and `Search()` records `thread->completedDepth` once a full
  iteration finishes. See *Reported depth* below.

### Reported depth

`Search()` is `while (++thread->depth < MAX_SEARCH_PLY)`, so three of the six
loop exits leave `thread->depth` one past the last completed iteration: the
`Limits.depth` check (it runs after the increment), the `Threads.stop` breaks
(mid-iteration abort), and the `MAX_SEARCH_PLY` condition. The other three — the
mate, soft-node, and soft-TM breaks — sit at the bottom of the body and leave it
correct.

This never surfaced before because every `PrintUCI` call was *inside* the loop,
where `thread->depth` is the iteration being reported. The post-vote print added
for `Minimal` runs after the loop, so `go depth 13` reported `info depth 14`.

`thread->completedDepth` is set immediately after the per-iteration
`Threads.stop` check passes, so it records only fully-searched iterations, and
helper threads track it too (the vote may pick one). `MainSearch` passes
`bestThread->completedDepth`; the two in-loop calls pass `thread->depth`
unchanged. `realDepth` also became `Max(1, updated ? depth : depth - 1)` so a
`completedDepth` of 0 cannot print `depth 0`.

This also fixes the same off-by-one on the pre-existing SMP path, where a
helper thread winning the vote reported its aborted depth rather than its
completed one.

**`bench.c`** — `Limits.softNodes = 0;` added to the reset block.
(`Bench()` still does not reset `Limits.nodes`; pre-existing, left alone.)

## Verification results

Driven interactively against a persistent engine process under `screen`
(`tmux` is not installed on this host), then swept non-interactively.

| Check | Result |
|---|---|
| Bench signature | 2,811,728 nodes — bit-identical to `main` |
| `uci` option surface | all three listed with correct defaults |
| `setoption` for all three | confirmations echo, no `Unknown command:` |
| `Minimal=true`, `go nodes 5000` | exactly 1 score line, immediately before `bestmove` |
| `Minimal=false`, same `go` | 11 lines — suppression is real, not an empty search |
| `Minimal=true`, `Threads=8`, ×5 runs | exactly 1 line every time (vote path) |
| `Minimal=true`, `go movetime 5000` | 1 score line; 23 `currmove` lines (no `score` token) |
| `Minimal=true`, `go depth 16` | 1 score line |
| `go depth 5/9/13/18` | reported depth == requested, both `Minimal` settings |
| `go depth 10` across all 49 bench FENs | 49/49 reported depth 10 |
| Hard-cap abort at 3k/20k/100k nodes | reported depth matches the non-minimal stream's last completed depth |
| `Minimal=true`, `MultiPV=3` | exactly 3 lines, `multipv 1..3`, no duplicates |
| `SoftNodes=false`, `go nodes 5000` | 5,004 nodes (hard stop) |
| `SoftNodes=true`, `go nodes 5000` | 14,205 nodes (completed the iteration) |
| `SoftNodes=true`, `go nodes 1/5/50` | 20 / 20 / 71 nodes — hard cap holds, no runaway |
| `Normalize` true vs false, `go depth 12` | cp 46 vs 73 (73/1.58 = 46), same nodes and PV |
| `Normalize` true vs false, mate-in-1 | `score mate 1` under both |
| Sweep of all 49 bench FENs, `Threads=1` | 49/49 exactly one line, correctly ordered |
| Sweep of all 49 bench FENs, `Threads=4` | 49/49 exactly one line, correctly ordered |
| Negative control, `Minimal=false` | 49/49 emitted >1 line — the assertion discriminates |

**Soft-node overshoot (WS2 calibration input):** at soft 5000, mean actual nodes
is **7,405 (1.48×)** at `Threads=1` and 9,041 (1.81×) at `Threads=4`. This is the
number to bracket the datagen opponent's hard node limit around.

### Gotcha: `quit` immediately after `go` aborts the search

`StartSearch` is non-blocking, so a piped
`printf 'go nodes 5000\nquit\n' | ./berserk` has `quit` set `Threads.stop` before
the search does any work. The result is a degenerate
`info depth 1 ... score mate 0 ... nodes 0` line. This is pre-existing behaviour,
not a regression — but it makes the obvious one-shot pipe test misleading.

Two things follow. First, verification must either drive a live session or drop
the `quit` and let EOF fall through to the `ThreadWaitUntilSleep` at the bottom
of `UCILoop`, which waits for the search to finish properly. Second, it is a live
confirmation of the documented depth-1 edge case: even with a search aborted
before it scored a single root move, `Minimal` still emitted exactly one score
line rather than zero. `PrintUCI`'s `updated` fallback and `InitRootMove`'s
`pv.count = 1` make the empty case unreachable.

## Known edges (documented, not fixed)

- **Search stopped inside depth 1** prints `score mate 0` via the `updated`
  fallback. Non-empty and no uninitialized read. Unreachable in normal play,
  since the hard cap is 100× the soft budget.
- **Zero legal moves**: `ParseGo` clamps `Limits.multiPV` to `rootMoves.count`,
  so `PrintUCI`'s loop body never runs and no info line is emitted. The only
  zero-line path — but fastchess adjudicates terminal positions and never issues
  `go` on them.
- **`UCI_ShowWDL` defaults true**, so the line reads
  `score cp 25 wdl 300 500 200 nodes ...`. fastchess parses `score cp <n>` by
  token; harmless, left as-is.

## Follow-up

- `genfens` command — the other half of WS1, still blocking OpenBench DATAGEN.
- Upstream PR to `openbench/Engines/Berserk.json`: replace the preset's
  `both_time_control: N=20000` with the WS2-calibrated value once measured.
