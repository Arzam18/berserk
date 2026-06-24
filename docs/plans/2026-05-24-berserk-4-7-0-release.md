---
ingested: 2026-06-04
source_type: plan
author: Claude Agent (direction + approval by Jay Honnold)
synthesis: done
---

# Berserk 4.7.0 — release (modern HCE, Windows binaries cross-compiled from Linux)

> Internal plan, authored and executed interactively on 2026-05-24.
> **Outcome:** Berserk 4.7.0 drafted as a GitHub release — tag `4.7.0` targeting commit
> `ca7d01a` on the `modern-hce` branch. Five static Windows `.exe` (cross-compiled from
> Linux with mingw-w64, no PGO) + `resources/berserk.jpg` published as assets — no `.nn`
> (the HCE eval is compiled in from `weights.c`). Created as a **draft** for Jay to review
> and publish.

## Context

The HCE backport designed on 2026-04-05 was completed on the `modern-hce` branch
(HEAD `1c91413` "Bring back HCE"). Keeps current Berserk's whole search/TT/threading/
time-management stack but swaps the NNUE eval layer back to the classic v4.6.0 handcrafted
eval (`eval.c`, `pawns.c`, `endgame.c`, `weights.c`, `kpk.h`, `tuner/`). `src/nn/` and
all network machinery are gone; the eval is compiled in from committed `weights.c`, so the
engine is fully self-contained (no `.nn`, no `download-network`).

Shipped as **Berserk 4.7.0**, continuing the pre-NNUE HCE semver line past v4.6.0. Mainline
NNUE keeps the bare-integer line (Berserk 13, 14). Parallel release line: `main` stays NNUE
(Berserk 14), the `4.7.0` tag lives on a `modern-hce` commit; branches are not merged.

## Decisions

- **Full launch, mirror Berserk 14**: build binaries → branch commits → draft GitHub release → wiki docs; Jay reviews and publishes.
- **No PGO** — same as v14; plain `-O3 -flto -static`. Notes recommend native `make pgo CC=clang`.
- **5 binaries**: `x86-64`, `sse41`, `avx2`, `avx2-pext`, `avx512-pext`.
- **No Elo / OpenBench data** — no test run; qualitatively positioned as weaker than NNUE Berserk 14.

## 4.7.0 vs Berserk 14 comparison

| Aspect | Berserk 14 (NNUE) | Berserk 4.7.0 (HCE) |
|--------|-------------------|-----------------------|
| Source branch / tag target | `main` @ `8ae895a` | `modern-hce` @ `ca7d01a` |
| Version string / tag | `14` | `4.7.0` |
| Bench | `2811728` | `3150010` |
| Network asset (`.nn`) | yes, embedded + shipped | none — eval compiled in from `weights.c` |
| Release assets | 5 `.exe` + `.nn` + jpg (7) | 5 `.exe` + jpg (6) |
| `.exe` size | ~26 MB (embedded net) | ~1.4 MB (no net) |

## Implementation

### `resources/build-windows.sh` (adapted)

Copied from `main` and adapted: `VER=4.7.0`, header comment fixed (HCE is self-contained
from `weights.c`, no embedded net). Loops 5 arches calling `make clean && make build` with
`CC=x86_64-w64-mingw32-gcc-posix`, `EXE=berserk-4.7.0-<arch>.exe`, `LIBS="-static -pthread -lm"`.

### Steps executed

1. Pre-flight: native `avx2` build (`CC=clang`), `./berserk bench` → **`3150010`** (matches
   committed value; tree correct). Confirmed binary runs with no network file present (HCE is
   self-contained).
2. Wrote `resources/build-windows.sh` (adapted), then `src/makefile` VERSION `20250622 → 4.7.0`.
3. Ran `build-windows.sh` → 5 self-contained `.exe` (~1.4 MB each) with `VERSION="4.7.0"` embedded.
4. Three commits to `modern-hce` (each `Bench: 3150010` so CI stays green):
   - `ce0b1c9` — Add Windows cross-compile build script
   - `ca7d01a` — Berserk 4.7.0 ← **the tagged commit**
   - `06d367e` — Bump version for development (`VERSION → 20260524`)
5. Pushed `modern-hce` to origin.
6. Created GitHub release as draft: `gh release create 4.7.0 --draft --target ca7d01a ...`
   with `--target` pinned to commit SHA. Uploaded 6 assets: 5 `.exe` + `resources/berserk.jpg`.
7. Pending: Jay reviews and **publishes** → tag `4.7.0` created at `ca7d01a`.

`.exe` are git-ignored (`*.exe`); they exist only as release assets.

## Verification

- **Bench**: native build → `3150010` nodes, matching committed `Bench: 3150010`.
- **Self-contained**: bench runs with no `.nn` anywhere; binaries ~1.4 MB.
- **Static + portable**: `file` on each `.exe` → `PE32+ (console) x86-64`; imports only
  `KERNEL32.dll` + `msvcrt.dll` (no `libgcc`/`libwinpthread` — fully static).
- **Release**: `gh release view 4.7.0` → draft, target `ca7d01a`, tag `4.7.0`, 6 assets.
- **Caveat**: `.exe` not executed on Linux box (no Wine). Jay recommended to run one
  (`bench` → expect `3150010`) on Windows before publishing.

## Release notes shape

Leads with what 4.7.0 is (modern-search HCE Berserk, v4.6.0 hand-crafted eval, no neural
network, fully self-contained), notes it's separate from NNUE mainline (Berserk 14 is
stronger), standard non-PGO cross-compile warning, copy-paste native `make pgo` block
(with `git checkout modern-hce`), and binary-selection guide.

## Out of scope (declined / N/A)

- PGO cross-compiles (would need Wine) — recommend native `make pgo`
- Plain `avx512` (non-pext) binary
- Elo / OpenBench testing for the release
- Merging HCE to `main` — mainline stays NNUE; 4.7.0 is a separate line
- `.nn` / network handling — HCE has none
