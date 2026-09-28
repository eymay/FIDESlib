# FIDESlib coverage baseline

Metrics produced by the local coverage pipeline (`scripts/run_coverage.sh` /
`scripts/run_coverage_sharded.sh`) — see the [README **Coverage** section](README.md#coverage).
Report: `coverage/index.html` (browse locally), machine-readable: `coverage/coverage.xml`.

## What is measured

- **Only `src/` and `api/`** are measured (see `scripts/coverage.gcovr.cfg`); tests,
  benchmarks, examples, `deps`, `nvtx` and build trees are excluded.
- **Host-side code only.** gcov instruments the host pass of `.cpp`/`.cu` files;
  CUDA device code (`__device__`/`__global__` bodies) is not instrumented, so
  kernel-heavy files report lower numbers than "real" coverage.
- Hot loops wrap gcov's 32-bit hit counters (gcc bug 68080); those lines are
  counted as covered (`suspicious_hits.warn`).
- Benchmark and timing suites are excluded (`P2PBenchmark`, `APIbench`,
  `LLMTests`, `Microbench`), as are `DISABLED_` gtest cases.

## Baseline: full test suite (minus the exclusions above)

Hardware for this baseline: RTX 4060 Ti 16 GB, `FIDESLIB_ARCH=89-real`, Debug
coverage build (`--coverage`, no `-G`), OpenFHE at `/usr/local`.

Reproduction (uses the sharded driver, which runs one case per process and
accumulates gcov counters across all of them):

```bash
BUILD_DIR=build-coverage ./scripts/run_coverage_sharded.sh
```

| Directory | Lines | Hit | Line % | Function % |
|---|---|---|---|---|
| api | 1753 | 1238 | 70.6 % | 71.5 % |
| src | 1712 | 783 | 45.7 % | 21.1 % |
| src/CKKS | 7895 | 3653 | 46.3 % | 47.4 % |
| src/CKKS/openfhe-interface | 633 | 568 | 89.7 % | 92.9 % |
| **TOTAL** | **11993** | **6242** | **52.0 %** | 39.7 % (581/1463) |

Branches: **24.4 %** (6442 of 26397 branch points). The full believable suite was
run: 131 core + 192 interface + 44 compat + 72 bootstrap = 439 runnable cases,
0 failures (3 multi-GPU tests skip on this single-GPU machine and 2 `DISABLED_`
cases are excluded).

## Notes on the numbers

- `api/` sits at ~71 %: the OpenFHE-facing API (CryptoContext, serialize,
  plaintext/ciphertext wrappers) is heavily exercised by the interface and
  compat suites. `src/CKKS/openfhe-interface/` is even higher (~90 %).
- The low `src/` function coverage (21 %) reflects many non-inline helpers in
  `CudaUtils`/`Math`/`NTThelper` that the tests only reach indirectly, plus
  host-side scaffolding around uninstrumented kernels.
- `Limb`, `LimbPartition` and the pool code are in `src/CKKS/` (46 %) — still the
  largest unexplored area along with bootstrap/approx-mod paths that the compat
  suite exercises only partially.

## Pipeline details

- `scripts/run_coverage.sh` — monolithic run (good for quick/small subsets via
  `TEST_FILTER`).
- `scripts/run_coverage_sharded.sh` — one-case-per-process run, crash-isolated,
  resumable (done-file in `coverage/.stats/`), used for the full-suite baseline.
  Each case is a foreground child process; gcda accumulates across exits.
- History: `056503b` fixes a Debug-only `Limb` assert; `d348f33` unblocks the
  pipeline (`-G` removal, `suspicious_hits.warn`, `TEST_FILTER`); `97b1a2c` +
  `1f93d6c` + `d8709e5` add the sharded driver (CASE_FILE mode, resumable
  done-file, foreground execution).
