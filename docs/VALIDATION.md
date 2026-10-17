# Validation

Date: 2026-10-16. TinyCC: 0.9.28rc 2026-09-04 mob@0fb54300 (AArch64
Darwin). Apple clang: 21.0.0 (clang-2100.0.123.102). Node: v23.10.0. Host
executable: `build/langc` from the sources of `5877e1d` (S6c), the last
commit that changes `src/`, `domain/`, `gen/`, the `Makefile` or
`test/gate.sh`. S7a and S7b add examples and tests only. S8 changes the
docs only.

`make -B check` passes. It builds `build/langc` and
`build/path-regressions` with TinyCC (`-std=c99 -Wall -Werror`). It checks
the sources with clang (`-Wall -Wextra -Wswitch-enum -Werror
-fsyntax-only`, `check-clang`). Then `test/gate.sh` runs the tests with zero
failures: examples 26 checked, 0 failed; parse refusals 11; check refusals
75; evals 67; json 26 builds; build refusals 4. `test/regressions.js`: 6
checked, 0 failed. `test/printer-regressions.js`: 2 checked.
`build/path-regressions`: 0 failed, with 82 C checks (the `failed += `
lines of `test/path-regressions.c`). A refusal must exit 1 and print
`langc: CODE: ` on stderr. `test/gate.sh` also checks a nesting of 1100
parentheses, a `refl` refusal that shows both normal forms, the eval
refusals, the JSON goldens (byte for byte, and each one must parse as
JSON), `-o` (a refused build leaves no file), the usage exit 2, and the
dash scan.

The final `make -B check` run of S8 (2026-10-16) took 62.84 s of wall time.
Its peak resident memory was 887.0 MiB. This value is the `ru_maxrss` of the
children (`getrusage`): the peak of the largest single process, not a sum.
The run does not name that process. S8 then measured the large processes
alone, one run each, in series: `build/langc check examples/schnorr.lang`
0.99 s, 995.0 MiB; `build/langc build examples/schnorr.lang` 1.00 s, 790.3
MiB; `build/langc build examples/gi.lang` 0.65 s, 664.0 MiB;
`build/path-regressions` 1.35 s, 101.3 MiB. The largest processes are the
two `examples/schnorr.lang` steps. The peak resident memory changes from run
to run: `probe/CAPABILITY.md` section 6 gives other values for the same
steps. The arena limit is 1 GiB (`src/main.c:10`).

The base commit `9f5e0a3` (S0) makes the language from lang-template with
the tcc-json host. It adds the design, the plan and the compiler scaffold.
Gate: examples 10, parse refusals 9, check refusals 24, evals 25,
json 10, build refusals 2.

S1 (`97e66e0`) adds 14 test files: 10 check refusals, 1 parse refusal, 1
JSON golden, `examples/paths.lang` and `test/path-regressions.c`. It
removes the kit `Eq`. They check the interval, `PathP` and `Path`, `refl`,
`sym`, `cong`, `funExt` and the path JSON. The refusals give
`INTERVAL_TYPE`, `PATH_END`, `PATH_UIP` (new) and `TYPE_MISMATCH`, and the
parse refusal gives `PARSE_EXPECT`. Gate: examples 11, parse refusals 10,
check refusals 34, evals 27, json 11, build refusals 2, C checks 6.

S2a (`a4bb138`) adds 5 test files: 3 check refusals (`face-sort`,
`j-refl-conv`, `transp-line`), 1 JSON golden and `examples/kan.lang`. They
check the `transp` builtin on closed, Pi and Sigma lines, and the `J`
library. The refusals give `TRANSP_LINE` (new), `PATH_END` and
`TYPE_MISMATCH`. Gate: examples 12, parse refusals 10, check refusals 37,
evals 29, json 12, build refusals 2, C checks 7.

S2b (`582ecef`) adds 6 test files: 5 check refusals and 1 parse refusal
(`system-top`). They check the `hcomp` builtin and its systems. The
refusals give `SYSTEM_COVER`, `SYSTEM_OVERLAP`, `HCOMP_CAP`,
`HCOMP_UNIVERSE` (new) and `PATH_UIP`. Gate: examples 12, parse refusals
11, check refusals 42, evals 30, json 12, build refusals 2, C checks 18.

S3a (`44c917f`) adds 7 test files: 3 check refusals (`fin-all-case`,
`fin-case-stuck`, `fin-range`), 2 build refusals (`fin-case-term`,
`fin-case-type`), 1 JSON golden and `examples/equiv.lang`. They check the
Fin core, the `finAll` rule and `isoToEquiv`. The refusals give
`FIN_CASE`, `FIN_RANGE` and `JSON_TYPE` (new). Gate: examples 13, parse
refusals 11, check refusals 45, evals 32, json 13, build refusals 4, C
checks 21.

S3b (`91d7091`) adds 9 test files: 7 check refusals, 1 JSON golden and
`examples/glue.lang`. They check Glue types, `ua` and the transport
tables. The refusals give `GLUE_EQUIV`, `GLUE_BOUNDARY`, `UNGLUE_TYPE`,
`GLUE_BINDER` (new), `PATH_END`, `PATH_UIP` and `SYSTEM_COVER`. Gate:
examples 14, parse refusals 11, check refusals 52, evals 34, json 14,
build refusals 4, C checks 27.

S3c (`34acff5`) adds 4 test files: 2 check refusals
(`hcomp-higher-universe`, `hcomp-universe-cap`), 1 JSON golden and
`examples/univ.lang`. It removes the check refusal `hcomp-universe`. They
check `hcomp` at Glue and in the universe, and `uaArrow`. The refusals give
`HCOMP_CAP` and `HCOMP_UNIVERSE`. Gate: examples 15, parse refusals 11,
check refusals 53, evals 35, json 15, build refusals 4, C checks 30.

S4a (`8c79e9a`) adds 5 test files: 3 check refusals (`fold-missing-path`,
`fold-path-boundary`, `loop-not-refl`), 1 JSON golden and
`examples/circle.lang`. They check path constructors, the circle `S1`, a
fold over a HIT and the formal `hcomp`. The refusals give `PATH_END`,
`PATH_UIP` and `TYPE_MISMATCH`. Gate: examples 16, parse refusals 11,
check refusals 56, evals 37, json 16, build refusals 4, C checks 39.

S4b (`e0d4f3c`) adds 2 test files: 1 JSON golden and
`examples/trunc.lang`. They check the propositional truncation `Trunc1`
and the transport in a family. It adds no refusal. Gate: examples 17, parse
refusals 11, check refusals 56, evals 38, json 17, build refusals 4, C
checks 41.

S4c (`25c14e0`) adds 3 test files: 1 check refusal (`trunc0-no-set`), 1
JSON golden and `examples/quot.lang`. They check the set truncation
`Trunc0` and the set quotient `Quot`. The refusal gives `TYPE_MISMATCH`.
Gate: examples 18, parse refusals 11, check refusals 57, evals 39, json
18, build refusals 4, C checks 46.

S5a (`a80e17e`) adds 5 test files: 3 check refusals (`fin-case-arity`,
`fin-case-branch`, `flag-gone`), 1 JSON golden and `examples/fin.lang`. It
removes the 2 build refusals `fin-case-term` and `fin-case-type`, and the
kit `Flag`. They check `finCase` and the Fin ring. The refusals give
`FIN_CASE` and `NAME_UNKNOWN`. Gate: examples 19, parse refusals 11, check
refusals 60, evals 42, json 19, build refusals 2, C checks 50.

S5b (`be42cc4`) adds 4 test files: 1 check refusal (`fin-one-not-empty`),
1 build refusal (`dec-json-function`), 1 JSON golden and
`examples/dec.lang`. They check decidable equality and Hedberg. The
refusals give `FIN_CASE` and `JSON_VALUE`. Gate: examples 20, parse
refusals 11, check refusals 61, evals 48, json 20, build refusals 3, C
checks 51.

S5c (`a08577d`) adds 4 test files: 1 check refusal (`finset-size-inside`),
1 build refusal (`finset-json-function`), 1 JSON golden and
`examples/finset.lang`. They check `FinSet`, the product FinSets and their
JSON. The refusals give `TYPE_MISMATCH` and `JSON_VALUE`. Gate: examples
21, parse refusals 11, check refusals 62, evals 53, json 21, build
refusals 4, C checks 58.

S6a (`14ebb60`) adds 4 test files: 2 check refusals
(`carrier-sum-codomain`, `rand-type0`), 1 JSON golden and
`examples/rand.lang`. They check `Rand` values, the carrier sums and the
JSON tables. The refusals give `TYPE_MISMATCH`. Gate: examples 22, parse
refusals 11, check refusals 64, evals 56, json 22, build refusals 4, C
checks 62.

S6b (`9189977`) adds 4 test files: 2 check refusals (`couple-point`,
`couple-size`), 1 JSON golden and `examples/couple.lang`. They check
couplings, `randBind` and the finite path JSON. The refusals give
`FIN_CASE` and `PATH_END`. Gate: examples 23, parse refusals 11, check
refusals 66, evals 58, json 23, build refusals 4, C checks 74.

S6c (`5877e1d`) adds 5 test files: 3 check refusals (`bag-fold-no-set`,
`count-no-dec`, `match-unequal`), 1 JSON golden and `examples/bag.lang`.
They check the pair decisions, the counts, `Bag` and `fiberMatch`. The
refusals give `FIN_CASE` and `TYPE_MISMATCH`. Gate: examples 24, parse
refusals 11, check refusals 69, evals 61, json 24, build refusals 4, C
checks 82.

S7a (`7bd9ce1`) adds 5 test files: 3 check refusals (`schnorr-nonbij`,
`schnorr-wrong-s`, `schnorr-wrong-w`), 1 JSON golden and
`examples/schnorr.lang`. They check the Schnorr HVZK path `hvzk`. The
refusals give `FIN_CASE`. Gate: examples 25, parse refusals 11, check
refusals 72, evals 64, json 25, build refusals 4, C checks 82.

S7b (`31d73f8`) adds 5 test files: 3 check refusals (`gi-nonbij`,
`gi-wrong-s`, `gi-wrong-w`), 1 JSON golden and `examples/gi.lang`. They
check the graph isomorphism HVZK path `giHvzk`. The refusals give
`FIN_CASE`. Gate: examples 26, parse refusals 11, check refusals 75, evals
67, json 26, build refusals 4, C checks 82.

## Mutants

Each mutant changes the source (one site, or two sites for G9), runs `make -B check`, and
restores the source. `docs/STATUS.md` gives each mutant row in a table
`| Mutant | Change | Killed by |`, and it records a `shasum` check of each
restore. Each kill came from a test, not from a compile error.

| Slice | Mutants | Result | New test for a mutant |
|---|---|---|---|
| S1 | M1 to M4, M5a to M5c, M6, M7, M8a, M8b | 11 KILLED | `path-sym` (M5a), `pathp-level` (M8a) |
| S2a | K1 to K5 | 5 KILLED | `j-refl-conv` (K3) |
| S2b | K6 to K14 | 9 KILLED | `trans-unit` (K10) |
| S3a | G1 to G6 | 6 KILLED | `fin-case-stuck` (G6) |
| S3b | G7 to G16 | 10 KILLED | the def `glueOverLoop` (G14) |
| S3c | G17 to G22 | 6 KILLED | none |
| S4a | H1 to H11, G1 | 12 KILLED | none |
| S4b | M9a, H12 to H14 | 4 KILLED | none |
| S4c | H15 to H18 | 4 KILLED | none |
| S5a | J1 to J12 | 12 KILLED | the def `mulAt2` (J6) |
| S5b | J13 to J17 | 5 KILLED | `fin-one-not-empty` (J13) |
| S5c | J18 to J20 | 3 KILLED | none |
| S6a | L1 to L6 | 6 KILLED | none |
| S6b | L7 to L9 | 3 KILLED | none |
| S6c | L10 to L12 | 3 KILLED | none |
| S7a | N1 to N4 | 4 KILLED | none |
| S7b | N5 to N8 | 4 KILLED | none |

Total: 107 mutant rows, 107 KILLED. The name G1 is on two rows: the S3a
row (the Fin literal: no `k < n` test) and the S4a row (the `glue_at`
fallback). G14 survived the planned S3b tests, and the new def
`glueOverLoop` kills it. J6 survived run 2 (3 * 7 and 3 + 7 are both 10
in `Fin 11`), and run 3 kills it with the new def `mulAt2`. J1 to J3 are
the S4c mutants that did not run in S4c; S5a ran them on the clean tree
`25c14e0`. M9a sits in the S4b table. S8 runs no mutant (ruling R9 (a),
docs only).
