# coin-lang

coin-lang is a language for perfect honest-verifier zero knowledge (HVZK)
as an equivalence of coin spaces. The type formers are products,
coproducts, monads (map and bind), folds and unfolds, and the paths, loops
and quotients of cubical type theory (CTT). The base types and the base
operations are the minimum span for this model. The kernel does not
identify proofs: there is no proof irrelevance, so a loop or a quotient can
be non-trivial. `langc` checks the program and writes one JSON document of
the final state of each instance. The language comes from lang-template
with the tcc-json host (TinyCC).

## Commands

| Command | Result |
|---|---|
| `langc check PROG` | Checks the program. Prints `ok`. |
| `langc eval PROG NAME [ARGS...]` | Applies `NAME` to the literal arguments and prints the normal form. |
| `langc build PROG [-o OUT]` | Checks the program and writes the JSON document of each instance. |

Exit 0 is ok, exit 1 is a refusal, exit 2 is a usage or IO error.

- `make`: builds `build/langc` with TinyCC. The build embeds
  `domain/domain.lang` in the compiler (`gen/embed.c`).
- `make -B check`: the gate. It builds `build/langc` and
  `build/path-regressions`, checks the sources with clang, and runs
  `test/gate.sh`, `test/regressions.js`, `test/printer-regressions.js` and
  `build/path-regressions`. Use `-B`: make 3.81 compares the file times to
  the second.
- `make check-clang`: checks the sources with clang (`-Wswitch-enum`,
  `-fsyntax-only`).

## Examples

Each file in `examples/` checks, and the gate compares its JSON document
with `test/json/`.

| File | Content |
|---|---|
| `algebra.lang` | Formers F6 and F7: fold over Nat, List and a domain family, and unfold with a step limit. |
| `bag.lang` | Decidable equality of pairs (`decEqSigmaFin`), counts, `Bag` and `fiberMatch` (S6c). |
| `circle.lang` | The circle `S1`: a path constructor and a fold (S4a). |
| `couple.lang` | Couplings: `couple` makes a path in `Rand T` (S6b). |
| `dec.lang` | Decidable equality and Hedberg (S5b). |
| `entries.lang` | Entries: first-order Nat and `Fin 2` definitions. |
| `equality.lang` | Paths: `refl`, `sym` and `cong`. |
| `equiv.lang` | The Fin core and the equivalence library (S3a). |
| `fin.lang` | The finite case `finCase` (S5a). |
| `finset.lang` | Finite sets: `FinSet` (S5c). |
| `formers.lang` | Formers F1 to F4: products, sums, options and lists. |
| `functions.lang` | Formers F9 and F10: functions and dependent functions. |
| `gi.lang` | Graph isomorphism HVZK (S7b). |
| `glue.lang` | Glue types and univalence (S3b). |
| `inventory.lang` | The kit sample domain in use, with the indexed family `Lot` (former F15). |
| `kan.lang` | Kan operations: `transp` and the library (S2a). |
| `laws.lang` | Laws that the checker proves by normalization: each proof is `refl`. |
| `monad.lang` | Formers F5 and F8: pure, map and bind over Option, Sum E and List, and filter. |
| `paths.lang` | Paths over the interval: `refl`, `sym`, `cong` and `funExt` (S1). |
| `quot.lang` | The set truncation `Trunc0` and the set quotient `Quot` (S4c). |
| `rand.lang` | Random variables: `Rand T` (S6a). |
| `schnorr.lang` | Schnorr HVZK (S7a). |
| `sigma.lang` | Former F11: dependent pairs. |
| `trunc.lang` | The propositional truncation `Trunc1` (S4b). |
| `univ.lang` | `hcomp` at Glue and in the universe, and `uaArrow` (S3c). |
| `universes.lang` | Former F14: universes. `Type 0 : Type 1`, with no cumulativity. |

## Slices

The slices S0 to S8 are DONE. `docs/STATUS.md` gives each slice and its
tests. `docs/VALIDATION.md` gives the gate counts of each slice.

| Slice | Commit | Content |
|---|---|---|
| S0 | `9f5e0a3` | The design, the plan and the compiler scaffold. |
| S1 | `97e66e0` | The interval and paths. The kit `Eq` is removed. |
| S2a | `a4bb138` | Transport: the `transp` builtin and the `J` library. |
| S2b | `582ecef` | Composition: the `hcomp` builtin and systems. |
| S3a | `44c917f` | The Fin core and equivalences: `Fin`, `finAll`, `isoToEquiv`. |
| S3b | `91d7091` | Glue and univalence: `ua`. |
| S3c | `34acff5` | Glue and universe composition, and `uaArrow`. |
| S4a | `8c79e9a` | Path constructors, `S1` and the formal `hcomp`. |
| S4b | `e0d4f3c` | The propositional truncation and family transport. |
| S4c | `25c14e0` | The set truncation and set quotients. |
| S5a | `a80e17e` | `finCase` and the Fin ring. The kit `Flag` is removed. |
| S5b | `be42cc4` | Decidable equality and Hedberg. |
| S5c | `a08577d` | `FinSet`, the product FinSets and their JSON. |
| S6a | `14ebb60` | `Rand` values, carrier sums and JSON tables. |
| S6b | `9189977` | Couplings, `randBind` and the finite path JSON. |
| S6c | `5877e1d` | Pair decisions, counts, `Bag` and `fiberMatch`. |
| S7a | `7bd9ce1` | The Schnorr HVZK example and its checks. |
| S7b | `31d73f8` | The graph isomorphism HVZK example and its refusals. |
| S8 | this commit | The documents and the final gate. |

## Documents

- `SPEC.md`: the specification.
- `design/DESIGN.md`: the domain design (the USER request). It is the only
  source of the core types and the core operations.
- `formers/FORMERS.md`: the type formers. `formers/tcc-json.md` gives their
  status on the tcc-json host.
- `probe/CAPABILITY.md`: the host probe. `probe/guard.py` runs one command
  with a wall-clock limit and a memory limit.
- `docs/PLAN.md`: the rulings, the design on the kit basis, the
  no-irrelevance rule, the cubical kernel, the base span, the JSON, the
  slices and the gates, and the open items.
- `docs/STATUS.md` and `docs/VALIDATION.md`: the status and the gate
  results.
- `docs/host/README.md`: how to build and test the host kit, the JSON
  document and the limits. `docs/host/CAPABILITY.md`: the facts of the
  host.
- `domain/domain.lang`: the domain library (the coin-lang definitions and
  the kit sample). `domain/README.md`: the domain interface.
- `test/`: `gate.sh` and its test files (`parse/`, `check/`, `eval/`,
  `json/`, `emit/`), `regressions.js`, `printer-regressions.js` and
  `path-regressions.c`.

The open items are in `docs/PLAN.md` section 8.

## License

MIT OR Apache-2.0 (`LICENSE-MIT`, `LICENSE-APACHE`).
