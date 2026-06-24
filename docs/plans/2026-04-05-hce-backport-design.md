---
ingested: 2026-06-03
source_type: plan
author: Claude Agent
synthesis: done
---

# Design: Backport Modern Berserk to HCE (v4.6.0 Eval)

> Internal design doc, dated 2026-04-05. Recovered from local Trash on 2026-05-24
> (it had been deleted, not committed to the berserk repo which is code-only).
> A sibling implementation plan (2026-04-05-hce-backport-impl.md, 828 lines,
> 3-phase) also existed in Trash; not imported here (design only, per request).
>
> **Status: implemented & shipped as Berserk 4.7.0** (2026-05-24) off the
> `modern-hce` branch. Not merged to mainline: berserk main stays
> NNUE-evaluated (Berserk 14). 4.7.0 is a separate HCE release line.

## Goal

Bring all non-NNUE improvements from current Berserk (HEAD, ~441 commits) to the
v4.6.0 HCE eval. Creates a modern HCE engine with all current search, move
ordering, threading, TT, and time management improvements.

## Approach

Start from current HEAD (all modern infrastructure intact), replace the NNUE eval
layer with v4.6.0 HCE, and adapt integration points. The eval is relatively
modular (called through `Evaluate()`) while the search infrastructure is deeply
interdependent and would be far harder to port piecemeal.

## Key Structural Conflicts and Resolutions

### 1. Board state tracking

- **Current**: `BoardHistory` struct stack at `board->history[ply]`
- **v4.6.0**: Flat arrays (`zobristHistory[]`, `pawnHashHistory[]`, etc.) indexed by `moveNo`
- **Resolution**: Keep modern `BoardHistory` approach. Pawn hash updates in MakeMove/UndoMove.

### 2. Incremental material+PSQT score

- **v4.6.0**: Maintains `board->mat` (packed mg/eg Score) incrementally in MakeMove/UndoMove
- **Current**: No equivalent — NNUE uses accumulators
- **Resolution**: Restore `board->mat`, `PSQT[12][2][64]` table, and incremental updates,
  replacing accumulator updates.

### 3. Phase calculation

- **Current**: `PHASE_VALUES[6] = {0,3,3,5,10,0}`, `MAX_PHASE = 64`, tracked incrementally
- **v4.6.0**: `PHASE_MULTIPLIERS[5] = {0,1,1,2,4}`, `MAX_PHASE = 24`, computed on demand via `GetPhase()`
- **Resolution**: Switch to v4.6.0 phase system. Remove incremental phase tracking from board.c.

### 4. Score type semantics

- Both use `typedef int Score` — no type conflict
- v4.6.0 packs mg/eg into a single int via `makeScore(mg, eg)`, `scoreMG(s)`, `scoreEG(s)` macros
- HCE eval uses packed scores internally, interpolates by phase, returns single int to search
- **Resolution**: Restore packing macros in `eval.h`.

### 5. Pawn hash table

- **v4.6.0**: `PawnHashEntry pawnHashTable[PAWN_TABLE_SIZE]` embedded in `ThreadData`
- **Current**: No pawn hash
- **Resolution**: Add `PawnHashEntry* pawnHashTable` to `ThreadData`, allocate dynamically.

### 6. EvalData struct

- **v4.6.0**: Used for king areas, attack maps, mobility squares, outposts, passed pawns, king safety
- **Resolution**: Add `EvalData` to `types.h`. Used only within eval.c and pawns.c.

### 7. EvalCoeffs (tuner)

- **v4.6.0**: Large struct for texel tuning coefficient extraction
- **Resolution**: Add to `types.h`, gated behind `#ifdef TUNE`.

### 8. Correction history

- Current `GetCorrectionScore()` uses `pawnCorrection[pawnZobrist]` and `contCorrection[piece][sq]`
- These are additive adjustments to static eval — works identically with HCE
- **Resolution**: Keep as-is. `pawnZobrist` exists in both versions.

### 9. Eval scale

- Both v4.6.0 HCE and current search operate on 100cp scale. No adjustment needed.
- `Normalize` in `uci.h` is purely for UCI display output.

## File-by-File Plan

### Restore from v4.6.0 (copy, then adapt)

| File | Lines | Notes |
|------|-------|-------|
| `src/eval.c` | 1080 | Full HCE. Adapt Evaluate() signature, field names (side→stm, halfMove→fmr). |
| `src/eval.h` | ~120 | Score packing macros, HCE constant externs. Keep modern EVAL_UNKNOWN, ClampEval(). |
| `src/pawns.c` | 216 | Pawn eval + pawn hash. Adapt field names. |
| `src/pawns.h` | 27 | Restore as-is. |
| `src/endgame.c` | 139 | Endgame scaling. |
| `src/kpk.h` | 3072 | Pure data, restore as-is. |
| `src/tune.c` | 1797 | Tuner framework, gated behind TUNE flag. |

### Remove

| File | Reason |
|------|--------|
| `src/nn/accumulator.c` | NNUE-specific |
| `src/nn/accumulator.h` | NNUE-specific |
| `src/nn/evaluate.c` | NNUE-specific |
| `src/nn/evaluate.h` | NNUE-specific |
| `src/incbin.h` | Used only for NN binary embedding |

### Modify

| File | Key Changes |
|------|-------------|
| `src/types.h` | Remove: Accumulator, AccumulatorKingState, acc_t, NN dimensions, PAWN_CORRECTION. Add: EvalData, PawnHashEntry, EvalCoeffs (#ifdef TUNE). Adapt Board: remove accumulators/refreshTable/phase, add mat. Adapt ThreadData: remove accumulators, add pawnHashTable pointer. |
| `src/board.c` | Remove accumulator push/pop. Restore incremental mat + pawnHash updates. Remove incremental phase. Adapt ParseFen to init mat via PSQT. |
| `src/search.c` | Remove #include "nn/accumulator.h" and accumulator reset calls. |
| `src/thread.c` | Remove accumulator alloc/free. Add pawnHashTable alloc/free. |
| `src/berserk.c` | Remove LoadDefaultNetwork(). Add InitPSQT() call. |
| `src/uci.c` | Remove EvalFile UCI option, network loading. Adapt EvaluateTrace for HCE. |
| `src/makefile` | Remove nn/*.c from SRC. Add pawns.c endgame.c tune.c. Remove EVALFILE/MAIN_NETWORK targets. |

### Unchanged

All modern search, move ordering, TT, threading, time management, SEE, history,
movepick, movegen, attacks, zobrist, random, perft, tb, pyrrhic, bench, move
encoding, bits, util.

## Build & Test Strategy

1. Get it compiling (fix all include/type errors)
2. `perft` — verify move generation unchanged
3. `bench` — different numbers than current (HCE vs NNUE) but must not crash
4. Play test games to verify correctness and strength
