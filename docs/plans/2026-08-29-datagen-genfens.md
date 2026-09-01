---
ingested: 2026-08-29
source_type: plan
author: Claude Agent
synthesis: done
---

# Datagen `genfens`: opening generation for OpenBench

> Implemented 2026-08-29. Bench verified bit-identical (2,811,728 nodes @ depth 13).

## Context

OpenBench DATAGEN workloads build their own opening book by invoking the **dev**
engine as:

```
./berserk "genfens N seed S book <None|Books/x.epd> <extra>" "quit"
```

and scraping stdout for lines beginning `info string genfens `. Berserk did not
implement the command, so it fell through `main()` to `UCILoop()`, read EOF from
the empty stdin, and exited having printed nothing. The client's
`output.get(timeout=15)` would then raise `OpenBenchFailedGenfensException` and
the workload would die before a single game was played.

This is the second and last item of **WS1 / milestone M1** of the NNUE data
pipeline plan. The other item — `Minimal` / `Normalize` / `SoftNodes` — landed in
`f5aaf80`, and this change depends on it: `MINIMAL` is what keeps the per-attempt
verification searches from flooding stdout.

## What the client requires

These are hard constraints, not preferences:

- Only `info string genfens <FEN>` lines are consumed; everything else on stdout
  is read and discarded. **Extra output is harmless.**
- The client's FEN-to-EPD conversion does `fen.split()[4:]` and `int()` on both
  halves, so the FEN **must** carry all six fields with integer halfmove and
  fullmove counters.
- The 15-second timeout is **per opening received**, not for the whole run. If
  stdout sat in a full buffer while generation took longer than 15s, the client
  would fail even though the engine was working. `setbuf(stdout, NULL)` is
  therefore required — `UCILoop()` sets it, but genfens never reaches
  `UCILoop()`.
- The command string ends with a **trailing space** when the extra-args field is
  empty, so the book must be parsed as a single token, not as "rest of string".

---

## Changes

### `src/board.h` — hoist `START_FEN`

`START_FEN` was `#define`d locally in `uci.c`. Moved to `board.h` beside the
`ParseFen` declaration so `datagen.c` can use it without duplicating the literal.

### `src/datagen.h`, `src/datagen.c` (new)

One public entry point, `void Genfens(char* args)`.

**Argument parsing** uses `strstr` + offset, matching `ParseGo`'s house style.
It is order-independent, and `%255s` for the book stops at whitespace so the
trailing space and any extra tokens are ignored:

```c
if ((ptrChar = strstr(args, "genfens")))
  n = atoi(ptrChar + 8);
if ((ptrChar = strstr(args, "seed")))
  seed = strtoull(ptrChar + 5, NULL, 10);
if ((ptrChar = strstr(args, "book")))
  sscanf(ptrChar + 5, "%255s", bookPath);
```

**Randomness** reuses Berserk's own generator: `SeedRandom(seed)` then
`RandomUInt64() % k`. No `srand`/`rand`. `SeedRandom` burns 64 values after
seeding, so adjacent per-thread seeds decorrelate.

**Book loading** slurps the EPD into a `char**` (the largest book in use is
4852 lines / ~340 KB). `\r\n` stripped with `strcspn`, blank lines skipped. A
missing or empty file prints `info string unable to open book <path>` and
returns rather than spinning forever.

**Random-move play** is iterative on a single `Board`, not recursive over copies —
Berserk's `Board` is ~32 KB because it carries `history[MAX_SEARCH_PLY + 100]`,
and the position is rebuilt with `ParseFen` on every attempt so nothing ever needs
undoing:

```c
static int PlayRandomMoves(Board* board, int n) {
  SimpleMoveList moves;

  for (int i = 0; i <= n; i++) {
    RootMoves(&moves, board);
    if (!moves.count)
      return 0;

    if (i < n)
      MakeMoveUpdate(moves.moves[RandomUInt64() % moves.count], board, 0);
  }

  return 1;
}
```

`RootMoves()` (`src/uci.c`) already yields fully legal moves via
`InitPerftMovePicker`. `MakeMoveUpdate(..., 0)` skips the accumulator work, which
is what `ParsePosition` does with its own stack `Board`, so the uninitialized
`accumulators` / `refreshTable` pointers are never touched.

**Verification search** copies the synchronous-search idiom already established
by `Bench()`. Depth 10 with a 1M-node ceiling, so a pathological position cannot
eat the client's 15s budget:

```c
static int VerificationSearch(Board* board) {
  Limits.depth       = VERIFICATION_DEPTH;
  Limits.multiPV     = 1;
  Limits.nodes       = VERIFICATION_NODES;
  Limits.softNodes   = 0;
  Limits.hitrate     = 1000;
  Limits.max         = INT_MAX;
  Limits.timeset     = 0;
  Limits.mate        = 0;
  Limits.infinite    = 0;
  Limits.searchMoves = 0;
  Limits.start       = GetTimeMS();

  TTClear();
  SearchClear();

  StartSearch(board, 0);
  ThreadWaitUntilSleep(Threads.threads[0]);

  return Threads.threads[0]->rootMoves[0].score;
}
```

`StartSearch` → `SetupMainThread` copies only up to `offsetof(Board,
accumulators)`, so the caller's `Board` never needs valid accumulator pointers —
the same contract `Bench()` relies on.

### `src/berserk.c` — dispatch

A new branch beside the existing `bench` one. Unconditionally exiting after
`Genfens` (rather than checking `argv[2]` for `quit`) matches `bench` and is what
the client wants, since it always passes `"quit"`:

```c
} else if (argc > 1 && !strncmp(argv[1], "genfens", 7)) {
  Genfens(argv[1]);
} else {
  UCILoop();
}
```

### `src/makefile`

`datagen.c` added to `SRC`.

---

## Design decisions

**Search output is not suppressed.** Each attempt still prints its one `Minimal`
PV line and a `bestmove`. The client reads and discards every line that is not
`info string genfens `, so this is harmless, and it keeps `search.c` completely
untouched by this change. `MINIMAL = 1` is set for the process and never
restored — `Genfens` always exits the process, so there is nothing to restore it
for. Without it each attempt would emit ~10 PV lines instead of one.

**argv only.** `genfens` is not reachable from `UCILoop`. This is exactly how
OpenBench invokes it, so tests exercise the production path rather than a
parallel one.

**Move counts** are `8 + rand%2` from startpos and `6 + rand%4` with a book. Each
range spans an even *and* an odd count, which is what keeps the side-to-move
split near 50/50.

**The `> 1000` reject threshold** is on Berserk's **raw internal** score. The
internal unit is ~1.58x a reported centipawn, so this is about +/- 6.3 pawns as
displayed. It is a shaping knob, not a correctness constraint — worth revisiting
once the first real shard's eval histogram exists.

**`ClearBoard` defaults** `moveNo = 1` and `fmr = 0`, so a book line lacking
halfmove/fullmove fields still produces a valid six-field output FEN.

---

## Verification

| Gate | Check | Result |
|---|---|---|
| A | `bench` signature unchanged | 2,811,728 nodes — identical |
| B | `genfens 64 seed 1 book None` → 64 well-formed 6-field FENs | 64/64, 0 malformed |
| C | Client's own `convert_fen_to_epd` accepts every line | 64/64 converted |
| D | Same seed identical, different seed differs | both hold |
| E | Side-to-move balance over 256 openings | 121 w / 135 b (47/53) |
| F | Real book (`UHO_Lichess_4852_v1.epd`), worker-shaped cwd, trailing space | 32/32, all mid-opening positions |
| F | Missing book file | `info string unable to open book`, immediate exit |
| G | Every emitted FEN re-parses and plays (round-trip through a live engine) | 64/64 `bestmove`, 0 `Unknown command` |
| H | Time per opening vs the 15s-per-opening client timeout | **3.7 ms/opening** (0.24s for 64) |
| I | Score filter is live, and bisects at the threshold | max accepted raw **988**, min rejected **1013** |

Gate I is the strongest evidence the filter is wired correctly: over 69 attempts,
64 were accepted and 5 rejected, and the accept/reject boundary falls exactly
either side of the raw-1000 threshold.

Gate G was run by driving one persistent engine in a tmux pane and replaying every
emitted FEN through `position fen ... / go depth 6`. Note that appending `quit`
immediately after `go` aborts the search before it starts — pre-existing
behaviour, and it would make every position look broken.

---

## Follow-up (not in this change)

- **WS2 node calibration.** M1 is complete with this change. The measured
  soft-node overshoot (7,405 mean nodes at soft 5000, Threads=1) brackets the
  opponent's hard-node candidates.
- **Upstream PR to `openbench/Engines/Berserk.json`** — the datagen preset's
  `both_time_control: N=20000` should become the WS2-calibrated value.
