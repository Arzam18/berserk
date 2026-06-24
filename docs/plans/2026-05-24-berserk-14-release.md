---
ingested: 2026-05-24
source_type: plan
author: Claude Agent (direction + approval by Jay Honnold)
synthesis: done
---

# Berserk 14 — release (Windows binaries cross-compiled from Linux)

> Internal plan, authored and executed interactively on 2026-05-24.
> **Outcome:** [Berserk 14 released](https://github.com/jhonnold/berserk/releases/tag/14)
> — tag `14` at commit [`8ae895a`](https://github.com/jhonnold/berserk/commit/8ae895a).
> Five static Windows `.exe` (cross-compiled from Linux with mingw-w64, **no PGO**) +
> the NNUE net + logo published as assets. Jay published the release after review.

## Context

First Berserk release since **Berserk 13** (2024-03-31). Hard constraint: produce
Windows binaries from a Linux machine. The engine already carries `_WIN32` guards
(e.g. `_setjmp` in `src/search.c`), so mingw compilation was feasible — the work was
wiring up the cross-toolchain and matching the historical release shape.

### Historical release pattern

- `src/makefile` `VERSION` is a dev date string between releases; at release it is set
  to the bare number (`14`), committed, tagged, then bumped back immediately.
- Every commit on `main` carries a `Bench: <n>` line (OpenBench convention); CI greps
  HEAD commit message for it.
- Release assets = several Windows `.exe` (one per CPU instruction set) + `.nn` net
  + `resources/berserk.jpg`.

## Decisions (confirmed with Jay)

- **No PGO** — cross-PGO would need Wine to run the instrumented `.exe`. Jay accepted
  the small strength cost; release notes recommend users build their own `make pgo`.
- **5 binaries**: `x86-64`, `sse41`, `avx2`, `avx2-pext`, `avx512-pext`. No plain
  `avx512` — Jay noted no AVX-512 CPUs lack PEXT.
- **Full publish, no Elo/OpenBench data** — no test run for this release. Notes
  explicitly call out the binaries as non-PGO Linux cross-compiles.

## Key technical findings

- **mingw POSIX variant is required.** The engine uses pthreads
  (`src/thread.c`/`thread.h`: `pthread_mutex_t`, `pthread_cond_t`) and
  `<stdatomic.h>`. The default mingw **win32**-threads variant has no `pthread.h`.
  Use `x86_64-w64-mingw32-gcc-posix` (Ubuntu pkg `mingw-w64`).
- **`-static` must be forced.** The makefile only adds `-static` behind a Windows-only
  `ifeq ($(shell echo "test"), "test")` check (on Linux `echo` strips quotes, so the
  branch never fires). Cross-compiling on Linux needs `-static` passed explicitly.
- **No makefile edit needed** — reuse the `build` target with CLI variable overrides:
  `make build ARCH=avx2 CC=x86_64-w64-mingw32-gcc-posix EXE=berserk-14-avx2.exe LIBS="-static -pthread -lm"`

## Implementation

### New artifact — `resources/build-windows.sh` (committed)

Loops the 5 arches calling `make clean && make build` with `CC`/`EXE`/`LIBS`
overrides, then a final `make clean`. Modeled on `resources/update.sh`.

### Steps executed

1. Pre-flight: native `avx2` build (`CC=clang`), `./berserk bench` → **`2811728`**.
2. `src/makefile` `VERSION` `20250622 → 14`.
3. Ran `resources/build-windows.sh` → 5 `.exe` with `VERSION="14"` embedded.
4. Three commits to `main` (each with `Bench: 2811728`):
   - [`dedc2a3`](https://github.com/jhonnold/berserk/commit/dedc2a3) `Add Windows cross-compile build script`
   - [`8ae895a`](https://github.com/jhonnold/berserk/commit/8ae895a) `Berserk 14` ← tagged commit
   - [`27212a2`](https://github.com/jhonnold/berserk/commit/27212a2) `Bump version for development` (`VERSION → 20260524`)
5. `gh release create 14 --draft --target 8ae895a ...` — pinning `--target` to commit
   SHA (not branch) lets post-release VERSION bump land on `main` without moving the
   tag. Uploaded 7 assets: 5 `.exe`, `berserk-9b84c340af7e.nn`, `berserk.jpg`.
6. Jay reviewed and **published** → tag `14` at `8ae895a`.

## Verification

- **Bench**: native build → `2811728` nodes, matching committed `Bench: 2811728`.
- **Static + portable**: `file` on each `.exe` → `PE32+ (console) x86-64`;
  `objdump -p` → imports only `KERNEL32.dll` and `msvcrt.dll` (no libgcc/libwinpthread).
- **Caveat**: `.exe` were not executed (no Wine). Jay recommended to run one
  (`bench` → expect `2811728`) on a Windows machine before publishing.

## Out of scope

- PGO binaries (would need Wine for the profiling run)
- Plain `avx512` (non-pext) binary
- Elo / OpenBench testing for this release
