# coin-lang status

> Write here: one paragraph. Say which milestone the build implements, and
> which milestone it does not complete yet.

## Implemented

> Write here: one item for each feature that works. Name the former (F1 to
> F15), the core type or the core operation.

- <feature>.

## Remaining M0 work

> Write here: one numbered item for each missing feature. Say what each item
> needs first.

1. <work item>.

## Known limits

> Write here: one item for each limit of the current build, with its cause
> (a fuel budget, a host limit, a stack size).

- <limit>.

## Internal boundaries

> Write here: one paragraph for each module group. Say which module holds
> which job, which tests check which tables, and which types a program
> cannot use.

## S1

S1 (the interval and paths) is complete. The build has the sort `I` with
`i0`, `i1`, `~`, `/\` and `\/`; `PathP` and `Path`; path abstraction and
`@`; `refl`, `sym`, `cong` and `funExt`; the neutral end point rule; path
eta; and the JSON rules 5 and 6. The kit `Eq` is gone (O8). `trans` and
`transport` come back in S2.

Gate: `make -B check` is GREEN at examples 11, parse 10, check 34, evals 27,
json 11, build refusals 2, regressions 6, printer 2. The S1 target was
check 32. The mutant work added two refusals (`path-sym`, `pathp-level`).

Mutants: one at a time, `make -B` for each build. Each file was restored and
its `shasum` was confirmed. Each kill came from a test, not from a compile
error.

| Mutant | Change | Killed by |
|---|---|---|
| M1 | `path_apply`: no neutral end point rule | `paths.lang` (`ev`), `path-ev`, `paths.json` |
| M2 | `path_ends`: `return 1` after the NULL check | `path-end`, `bad-refl`, `bad-eta`, `path-uip`, `path-k`, `path-ev` |
| M3 | `path_ends`: `if (uip) return 1` | `path-uip`, `path-k`, `path-ev` |
| M4 | `RULE_INTERVAL`: `I` is a term of `Type 0` | `interval-type`, `interval-pi`, `interval-path` |
| M5a | `ival_neg`: no literal flip, so `~i = i` | `path-sym` (new; the S1 tests did not kill it) |
| M5b | `ival_neg`: literals flip, no De Morgan step | `paths.lang` (`deMorganMeet`), `paths.json` |
| M5c | `ival_neg`: each literal becomes negative, so `~~r = r` fails | `paths.lang` (`symSym`), `paths.json` |
| M6 | `conv_loop`: no path eta (a path against a neutral) | `paths.lang` (`symSym`), `paths.json` |
| M7 | `path_json`: the rule 5 test is always true | `paths.json` (`ev` is rule 6) |
| M8a | `rule_pathp`: the level is always `Type 0` | `pathp-level` (new; the S1 tests did not kill it) |
| M8b | `rule_path`: the level is always `Type 0` | `path-level` |

M5a is a consistent reading of `~` (the dual of the lattice), so no accepted
def can kill it. Only a refusal can: `sym q` and `q` must differ.

A fix from the mutant work: `sym`, `cong` and `funExt` gave their interval
binder the line name of `Path`, which is `_`. A printed path term was then
`fun _ => p @ ~_`, and it did not replay. The binder is now `i`
(`path_binder` in `src/front/check.c`). `evBack` in `examples/paths.lang`
checks it through `test/json/paths.json`. The mutants ran before this fix;
the fix changes no mutant site.

## S2a

S2a (transport) is complete. The build has the builtin `transp (fun i =>
A) phi a`, with these rules: `phi` true gives `a`; a line in the universe
gives `a`; a line of a data former goes into the arguments; a Pi line
gives a function that transports its argument back to `i0`; a Sigma line
transports the first part, and the second part along the fill of the
first part. A `transp` that does not compute is a stuck term
(`OP_TRANSP`). The checker refuses a line that is not constant where `phi`
holds (`TRANSP_LINE`). `domain/domain.lang` adds `transport`, `subst`, `J`
and `JRefl`. `JRefl` is a path, not a conversion (O11). `hcomp`, `trans`
and the PathP rule of `transp` are S2b.

Gate: `make -B check` is GREEN at examples 12, parse 10, check 37, evals 29,
json 12, build refusals 2, regressions 6, printer 2, path regressions 7
(0 failed). The clang `-Wswitch-enum` pass is clean.

Tests: `examples/kan.lang` and `test/json/kan.json`; the check refusals
`transp-line`, `face-sort` and `j-refl-conv`; the evals `kan evPi` and
`kan evPair`; the C round-trip case `stuck transport fill` in
`test/path-regressions.c` (also `fillT` in `examples/kan.lang`).

Mutants: one at a time, `make -B` for each build. Each file was restored and
its `shasum` was confirmed. Each kill came from a test, not from a compile
error.

| Mutant | Change | Killed by |
|---|---|---|
| K1 | `transp`: no rule for `phi` true | each example (`JRefl` in `domain/domain.lang` has the wrong end at `i1`), `kan.lang` |
| K2 | `rule_transp`: no check that the line is constant | `transp-line` |
| K3 | `transp`: a line that is constant in `i` gives the base (regularity) | `j-refl-conv` |
| K4 | Pi rule: `f x`, not `f` of `x` transported back to `i0` | `kan.lang` (`transportPi`), `kan evPi`, `kan evPair`, `kan.json` |
| K5 | Sigma rule: the second part along `B i (fst a)`, not the fill | `kan.lang` (`transportSigma`), `kan evPi`, `kan evPair`, `kan.json` |

K3 is a consistent extension (regularity), as M5a was in S1. No accepted def
can kill it. Only a refusal can: `j-refl-conv` must stay refused.

## S2b

S2b (composition) is complete, and S2 is done. The build has the builtin
`hcomp A phi (fun (i : I) => [ k -> t, ... ]) u0`. A system `[ k -> t, ...
]` is only the tube of `hcomp`; at the top of a def it is a parse refusal.
The rules: `phi` true gives the branch whose key is true, at `i1`; a Pi
type gives `fun x => hcomp B phi [u x] (u0 x)`; a Sigma type composes the
first part and fills the second part; a PathP type adds the branches `~j`
and `j` for the ends; a data former goes into the arguments only if each
branch has the head of the base. An `hcomp` that does not compute is a
stuck term (`OP_HCOMP`), and it prints with its normalized face system. `transp` along a
PathP line is a `comp` with the branches `phi`, `~r` and `r` (rule 7). The
checker refuses keys that do not cover `phi` (`SYSTEM_COVER`), two
branches that disagree where both keys hold (`SYSTEM_OVERLAP`), a branch
that is not the base at `i0` (`HCOMP_CAP`) and `hcomp` at a universe
(`HCOMP_UNIVERSE`, Glue is S3). A face drops each contradictory
conjunction (`j /\ ~j`). `domain/domain.lang` adds `trans` by `hcomp`.

Gate: `make -B check` is GREEN at examples 12, parse 11, check 42, evals 30,
json 12, build refusals 2, regressions 6, printer 2, path regressions 18
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: the S2b part of `examples/kan.lang` (`transEnd`, `transNat`,
`evTrans`, `transportPathP`, `hcompPi`, `faceView`) and
`test/json/kan.json`; the check refusals `system-cover`, `system-overlap`,
`hcomp-cap`, `hcomp-universe` and `trans-unit`; the parse refusal
`system-top`; the eval `kan evTrans`; the C round-trip cases `stuck hcomp
with two keys` and `stuck transport in a Path line` in
`test/path-regressions.c`.

Review fixes: face restriction also acts on local variable types. A
disjunctive key keeps one elaborated body per conjunction, so inferred
arguments from one face cannot replace those on another. System values
discard unreachable branches before constructor projection and conversion,
including the empty faces generated by dependent Sigma composition.
Structural rules that would compose in a universe retain the outer Kan
operation until Glue (S3), so their printed terms remain checkable.

Nine additional C round-trip cases cover dead constructor branches,
dependent Sigma composition, universe fields in functions, Sigma, products
and paths, transport of a universe path, face action on a local type and
elaboration on a disjunctive face. All nine fail against the original
staged checker and evaluator and pass with the fixes.

Mutants: one at a time, `make -B` for each build. Each file was restored and
its `shasum` was confirmed. Each kill came from a test, not from a compile
error.

| Mutant | Change | Killed by |
|---|---|---|
| K6 | `hcomp`: no rule for `phi` true | each example (`trans` in `domain/domain.lang` has the wrong end at `i0`), `kan.lang` |
| K7 | `rule_hcomp`: no cover check | `system-cover` (`PATH_END`, not `SYSTEM_COVER`) |
| K8 | `rule_hcomp`: no overlap check | `system-overlap` (`HCOMP_CAP`, not `SYSTEM_OVERLAP`) |
| K9 | `rule_hcomp`: no cap check | `hcomp-cap` (`PATH_END`, not `HCOMP_CAP`) |
| K10 | data `hcomp`: into the arguments without the branch head test | `trans-unit` (accepted) |
| K11 | a face keeps a contradictory conjunction | each example (`trans` in `domain/domain.lang`: `SYSTEM_OVERLAP` at `~j /\ j`) |
| K12 | `rule_hcomp`: a universe accepted | `hcomp-universe` (accepted) |
| K13 | Pi rule: the system dropped | `kan.lang` (`hcompPi`), `kan evPi`, `kan evPair`, `kan evTrans`, `kan.json` |
| K14 | PathP rule of `transp`: the `~r` branch dropped | `kan.lang` (`transportPathP`), `kan evPi`, `kan evPair`, `kan evTrans`, `kan.json` |

K10 is a consistent extension (a unit law of `trans` by conversion), as K3
was in S2a. No accepted def can kill it. Only a refusal can: `trans-unit`
must stay refused. K11 stops each example at `trans` before `faceView`
in `kan.lang` is checked.

## S3a

S3a (the Fin core and the equivalence library) is complete. The build has
the builtin type `Fin n`. A literal `k` checks against `Fin n` only when
`k` is below `n` (else `FIN_RANGE`). The builtins `finAdd n a b` and
`finNeg n a` compute mod `n` on literals and stay stuck on a neutral.
`transp` along a constant `Fin n` line gives the base, as for `Nat`. A
lambda on a `Fin n` domain with a literal `n` checks as usual first. When
that check fails, the checker checks the body once for each value `0` to
`n - 1` (finAll) and makes a case term (`OP_FIN_CASE`)
with one body for each value. A case that fails gives `FIN_CASE` with the
case number and the inner refusal. A case term on a literal evaluates only
the body of that literal; on a neutral it is stuck. `finCase` has no
surface form, so the source printer refuses a stuck case instead of
emitting an unknown name. `eval` gives `EVAL_PRINT`, and `build` gives
`JSON_TYPE` without output when a type, path or point needs such a case.
Closed cases that compute print normally. `domain/domain.lang` adds the equivalence library in the CTT
form: `isContr`, `fiber`, `isEquiv`, `Equiv` (a Sigma of a function and
contractible fibers), `idEquiv` and `isoToEquiv`. Glue and `ua` are S3b.

Gate: `make check` is GREEN at examples 13, parse 11, check 45, evals 32,
json 13, build refusals 4, regressions 6, printer 2, path regressions 21
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: `examples/equiv.lang` (`finWrap`, `finNegOne`, `finTransp`,
`finTrans`, `idEquivFin3`, `plus3`, `minus3`, `plus3At9`, `addMod`, `rot`,
`rot2`) and `test/json/equiv.json`; the check refusals `fin-range`
(`FIN_RANGE`), `fin-all-case` and `fin-case-stuck` (both `FIN_CASE`); the
evals `equiv addMod` and `equiv rot2`; the C round-trip case `stuck Fin
addition` in `test/path-regressions.c`. In `plus3At9`, the section and the
retraction of `isoToEquiv` are `fun (k : Fin 11) => refl` by finAll, and
the equivalence is inline. Two C printer refusals cover an internal case
in a function and in a path. `test/emit/fin-case-type.lang` and
`fin-case-term.lang` require `JSON_TYPE` with no output, for a type and a
path term respectively.

Deviations from the plan (kickoff section 2):

- The C case. The plan was a def with a stuck `OP_FIN_CASE` that
  round-trips. It cannot pass: `finCase` has no surface form, so the
  printer refuses the core. The C case is `stuck Fin addition`
  (`fun (r : Fin 11) => finAdd 11 r 3`), and mutant G6 gets the new check
  refusal `fin-case-stuck`. So check is 45, not 44. A surface `finCase` is
  a design decision, not a test change, and S3a does not add it.
- The evals. An eval entry is a first-order `Nat` or `Flag` def, so `rot 2`
  gives `EVAL_ARGS`. The closed def `rot2 : Fin 3 := rot 2` replaces it
  (`equiv rot2 => 0`). The planned eval `equiv finAdd` uses the def
  `addMod`, because `finAdd` is a core name (`REFUSE_NAME`).
- `fin-case-stuck`, first form. It did not kill G6: each case body referred
  to `k` (`pack k refl`), so body 0 at a neutral `k` gave the witness `k`,
  which is also stuck. The second form proves `p : (k : Fin 2) -> Path (Fin
  2) (finAdd 2 k 0) k` by finAll, with the end value in each case body, and
  `bad` states that `(p k) @ i1` is `0` at a neutral `k`. The clean build
  refuses it: `FIN_CASE`, case 1, `PATH_END: the path at i1 is 1, but the
  type needs 0`.

Helper defs: `isoCap` and `isoPath` are the agda/cubical fill squares
`fill0` and `fill2` at `k = i1`. Each is written in the general fill form,
so its value is the value of the general fill. `isoToEquiv` writes the
other squares inline, and each face key is one literal (no face holds a
compound formula).

Choices:

- `Fin 0` has no finAll fallback. An empty case term would accept a body
  that is not checked, so the first refusal stays.
- finAll has no size limit other than the `uint32_t` arity (`n` below
  2^32 - 1) and the arena. `Fin 4000000000` gives `OOM`.

Mutants: one at a time, `make -B` for each build. Each file was restored and
its `shasum` was confirmed. Each kill came from a test, not from a compile
error.

| Mutant | Change | Killed by |
|---|---|---|
| G1 | Fin literal: no `k < n` test | `fin-range` (accepted) |
| G2 | `finAdd`: no `mod n` | `equiv.lang` (`finWrap`: `PATH_END`, 12, not 1), `equiv addMod`, `equiv rot2`, `equiv.json` |
| G3 | `finNeg`: gives `a` | `equiv.lang` (`finNegOne`: `PATH_END`, 1, not 2), `equiv addMod`, `equiv rot2`, `equiv.json` |
| G4 | finAll: checks only the case `j = 0` | `fin-all-case`, `fin-case-stuck` (both accepted) |
| G5 | `transp_data` OP_FIN: stuck, as OP_FAMILY | `equiv.lang` (`finTransp`: `PATH_END`, the `transp` stays), `equiv addMod`, `equiv rot2`, `equiv.json` |
| G6 | OP_FIN_CASE: a neutral `k` gives body 0 | `fin-case-stuck` (accepted) |

G5 keeps the `case OP_FIN:` label and stops in it: a deleted label is a
`-Wswitch-enum` compile error. G6 is a consistent extension, as K3 in S2a
and K10 in S2b. No accepted def can kill it. Only a refusal can:
`fin-case-stuck` must stay refused.
