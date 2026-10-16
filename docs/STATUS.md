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

## S3b

S3b (Glue and univalence) is complete. The build has the builtins `Glue`,
`glue` and `unglue` (the core operations `OP_GLUE`, `OP_GLUE_ELEM` and
`OP_UNGLUE`). `Glue A phi (fun (_ : I) => [ k -> pack T e, ... ])` is a
type in `Type 0`. The keys must cover `phi` (`SYSTEM_COVER`), each branch
must be `pack T e` with `e : Equiv T A` (`GLUE_EQUIV`), two branches must
agree where both keys hold (`SYSTEM_OVERLAP`), and the tube binder must be
`_` (`GLUE_BINDER`). `glue (fun (_ : I) => [ k -> t, ... ]) a` checks only
against a Glue type. Where a key holds, `unglue` of its branch must be `a`
(`GLUE_BOUNDARY`). `unglue g` needs a term of a Glue type
(`UNGLUE_TYPE`). A true key reduces `Glue` to its branch type `T` and
`glue` to its branch `t`. `unglue (glue sys a)` is `a`, and at a true key
`unglue g` is `witness e g`. Else `unglue` is stuck. `transp` along a Glue
line gives a `glue` at `i1`: each branch comes from the fiber extension of
its equivalence, and an `hcomp` in `A` at `i1` gives the base. Conversion
of two Glue types compares the systems too, not only the bases. `domain/domain.lang`
adds `ua A B e`, the Glue line with the branch `pack A e` at `~i` and
`pack B (idEquiv B)` at `i`. Transport along `ua A B e` at a closed value
`a` gives `witness e a` (uaβ). The JSON writer prints a path in the
universe from `Fin n` (a literal `n`) as its transport table (rule 1): for
example `uaPlus3` is `{"from": "Fin 11", "to": "Fin 11", "transport":
[3, 4, ..., 2]}`. S3c is next.

Gate: `make check` is GREEN at examples 14, parse 11, check 52, evals 34,
json 14, build refusals 4, regressions 6, printer 2, path regressions 23
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: `examples/glue.lang` (`plus3`, `minus3`, `uaPlus3`, `uaBetaPlus3`,
`loopId`, `loopRot1`, `loopRot2`, `loopRef0`, `loopRef1`, `loopRef2`,
`gRot`, `glueFace`, `unglueGlue`, `glueOverLoop`, `uaAt9`, `rotTwice`) and
`test/json/glue.json` (13 instances). The 6 loops at `Fin 3` give 6
distinct tables: `loopId` `[0,1,2]`, `loopRot1` `[1,2,0]`, `loopRot2`
`[2,0,1]`, `loopRef0` `[0,2,1]`, `loopRef1` `[1,0,2]`, `loopRef2`
`[2,1,0]`. The check refusals `glue-equiv` (`GLUE_EQUIV`), `glue-cover`
(`SYSTEM_COVER`), `glue-boundary` (`GLUE_BOUNDARY`), `unglue-type`
(`UNGLUE_TYPE`), `glue-binder` (`GLUE_BINDER`), `ua-beta-neutral`
(`PATH_END`) and `ua-not-refl` (`PATH_UIP`). The evals `glue uaAt9`
(`1`) and `glue rotTwice` (`2`). The C round-trip cases `Glue type at a
neutral face` and `stuck unglue` in `test/path-regressions.c`. In
`ua-beta-neutral`, uaβ at a neutral value does not reduce: the `i1` end
stays a stuck `hcomp B i0 ...`, because the build has no regularity.

Deviations from the plan (kickoff section 2):

- Counts. Check is 52, not 50: S3a ended at 45 (the `fin-case-stuck`
  deviation), and S3b adds 7 refusals, with `glue-binder`. Build refusals
  are 4, not 2: S3a ended at 4, and S3b adds none. Path regressions are 23
  cases, not 21: 44c917f had 21 cases (S3a added `stuck Fin addition` and
  the 2 printer refusal cases), and S3b adds 2.
- `ua-not-refl`. The planned second end `refl` cannot kill G16: `refl` at
  `Fin 3` is not a Glue type, and G16 changes only the conversion of two
  Glue types (`OP_GLUE`). The second end is `ua (Fin 3) (Fin 3) (idEquiv
  (Fin 3))`, the `ua` image of `refl`, a Glue line with the base `Fin 3`.
  The observed code is `PATH_UIP` (the plan allows `PATH_UIP` or
  `PATH_END`).
- `gRot` is new: `(x : Fin 3) -> PathP (fun i => loopRot1 @ i) x (x + 1)`,
  a `glue` with the branches `~i -> x` and `i -> x + 1` and the base
  `x + 1`. It is under a Pi, so the JSON writer skips it. `glueFace` (`gRot
  2 @ i0` is `2`, the branch, not the base `0`) and `unglueGlue` (`0` at
  the neutral face and at `i0`) use it, for G12 and G13.
- G14 survived the planned tests (the plan named `ua-beta-neutral` or
  `uaBetaPlus3`). Cause: each planned Glue line is a `ua` line. Its base
  `A` is constant, so the `comp` in `A` gives `a0` at closed values. The
  new def `glueOverLoop` transports `glue (fun (_ : I) => [ ]) 0` along
  `fun (i : I) => Glue (loopRot1 @ i) i0 (fun (_ : I) => [ ])`, a Glue line
  with no branch whose base changes along `i`: `unglue` at `i1` is `1`
  (`0` with G14).
- `GLUE_BINDER` (ruling (a)). At the end of the build the refusal had no
  test. The ruling is (a): the 7th check test
  `test/check/glue-binder.lang` (the tube binder is `j`), not a C refusal
  helper (b) and not a note only (c). This is the change from 51 to 52.

Mutants: one at a time, `make -B` for each build. Each file was restored and
its `shasum` was confirmed. Each kill came from a test, not from a compile
error.

| Mutant | Change | Killed by |
|---|---|---|
| G7 | `reduce_glue`: a true key always (`1 \|\|` before `glue_true`) | each example (14): `PATH_END`, `ua` at `i0` is `Glue B i1 [...]` |
| G8 | `glue_branch`: on a failure, check the branch again against `Sigma (T : Type 0) (T -> A)` | `glue-equiv` (accepted) |
| G9 | `rule_glue` gives a NULL face to `system_branches`, which then uses the join of the keys | `glue-cover` (`PATH_END` at `i0`, not `SYSTEM_COVER`) |
| G10 | `rule_glue_elem`: no `glue_boundary` (`0 &&`) | `glue-boundary` (accepted) |
| G11 | `rule_unglue`: a term of a type that is not Glue gives `g` with its type | `unglue-type` (accepted) |
| G12 | `reduce_unglue` rule 2: `g`, not `witness e g` | `glue.lang` (`uaBetaPlus3`: `9`, not `1`), `glue uaAt9`, `glue rotTwice`, `glue.json` |
| G13 | `reduce_unglue` rule 1: the body of branch 0, not `a` | `glue.lang` (`unglueGlue`: `2`, not `0`), the evals, `glue.json` |
| G14 | `transp_glue`: `a1' = a0` (no `comp` in `A`) | NEW def `glueOverLoop` (`0`, not `1`), the evals, `glue.json` |
| G15 | `transp_glue`: `witness e u0` when the `i0` system has a true key | `ua-beta-neutral` (accepted) |
| G16 | conversion: two `OP_GLUE` are equal when their `A` agree | `ua-not-refl` (accepted) |

G8 accepts a branch whose `e` is a plain function `T -> A`, not an
equivalence. G9 is a 2-site edit, because `hcomp`, `Glue` and `glue` share
`system_branches`: the NULL face comes only from `rule_glue`. G14 gets a
new def, as S1 M5a, S2a K3, S2b K10 and S3a G6.

S3b staged review fixed three defects. When a Glue branch type is itself
Glue, transport now recovers the outer Glue metadata before inspecting
the reduced branch type. At a true face, unglue applies the branch
equivalence before testing for a glue constructor of the inner type.
The universe-path JSON table accepts canonical function points and uses
the same source-string encoding as other function path points.

Four C regressions cover the nested Glue transport round trip, nested
unglue face reduction, a Fin 1 to function transport table, and the
existing path-syntax fallback for a stuck transport. Against the original
staged implementation, the first three fail with INTERNAL, GLUE_BOUNDARY
and JSON_TYPE respectively. The corrected gate passes with 27 path
regressions and the other counts above unchanged.

## S3c

S3c (`hcomp` at Glue, `hcomp` in the universe, `uaArrow`) is complete,
and with it S3. `hcomp` at `Glue A phi sys` gives a `glue`: for each key
of `phi`, the branch is `hcomp T psi u u0`, and the base is an `hcomp` in
`A` of `unglue` of the tube and `unglue u0`, with `witness e` of the
filler in `T` at each key of `phi` (CCHM). `hcomp (Type 0) psi (fun (j :
I) => [ k -> A, ... ]) A0` gives `Glue A0 psi (fun (_ : I) => [ k -> pack
(A at i1) e, ... ])`. Each `e` is `lineToEquiv` of the reversed line: the
transport of `idEquiv (A at i1)` along `fun j => Equiv (A at i1) (A at
~j)` (the closure `CLOS_EQUIV_LINE`). A key that is `i0` gives no branch.
`rule_hcomp` has no `HCOMP_UNIVERSE` refusal now, and the cap check
applies at the universe too (`HCOMP_CAP`). The six universe guards stay
(Q-S3c-1 (a)): `hcomp` in `Type 0` reduces only at a direct `hcomp (Type
0)`. Where a Pi, Sigma, PathP or data rule reaches the universe, the
outer operation stays stuck. `domain/domain.lang` adds `uaArrow A B C e
f0 f1 h : PathP (fun (i : I) => (ua A B e @ i) -> C) f0 f1`, by `fun i a
=> hcomp C (~i \/ i) (fun (j : I) => [ ~i -> h a @ ~j, i -> f1 a ]) (f1
(unglue a))` (agda/cubical `ua→`, a constant `C`).

FIX B (session 20). Transport along a line made by `hcomp` in `Type 0`
first stopped with `INTERNAL` (the evaluator found a line that changes its
type former). The cause was in `glue_at`: one slot (`glue_out`,
`glue_raw`) held the last Glue that `reduce_glue` reduced at a true key,
and the slot was wrong in 3 ways: `hcomp_true` reduced an `hcomp` in `Type
0` at a true key with no Glue record; a nested `glue_at` (through
`CLOS_GLUE_BASE`) wrote over the slot; and two equal pointers (a cached
`loopRot1 @ i1` and the outer branch type) took the wrong record. The fix
has 3 parts. (1) `hcomp` does not call `hcomp_true` in `Type 0`: a true
key goes to `hcomp_univ`, so its Glue makes a record. (2) `Machine` holds
a list of `GlueRec` (`out`, `raw`, newest first) in place of the slot. At
a true key, `reduce_glue` returns a fresh shallow copy of `T` (a unique
pointer) and pushes a record for it. (3) `glue_at` keeps the list head,
runs `line_at`, and looks only at the new records. The arena grows by one
record for each true-key reduction.

Gate: `make check` is GREEN at examples 15, parse 11, check 53, evals 35,
json 15, build refusals 4, regressions 6, printer 2, path regressions 30
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: `examples/univ.lang` (`loopRot1`, `transTwoLoops`, `transTwo0`,
`transTwo1`, `transTwo2`, `gRot`, `hcompGlue`, `plus3`, `minus3`,
`uaArrowPlus3`, `uaArrowEnd0`, `uaArrowEnd1`, `transTwo`) and
`test/json/univ.json` (8 instances). `transTwoLoops` is the composite of
two `loopRot1` lines: its table is `[2, 0, 1]`, the table of `loopRot2`.
`transTwo0`, `transTwo1` and `transTwo2` check its transport by refl (`0`
to `2`, `1` to `0`, `2` to `1`). `uaArrowEnd0` checks `(f after plus3) 9`
= `2` at `i0`, and `uaArrowEnd1` checks `f 9` = `10` at `i1`, for `f = b +
1`. The check refusal `hcomp-universe-cap` (`HCOMP_CAP`) replaces
`hcomp-universe` (now accepted). The eval `univ transTwo` (`0`).

Deviations from the plan (kickoff section 2; the kickoff text now gives
the rulings):

- Counts. S3c initially kept check at 52: S3b ended at 52, and S3c removes
  1 and adds 1. The review adds the higher-universe refusal, giving 53.
  Build refusals are 4, not 2 (S3a ended at 4). S3b ended at 27 C path
  cases; the planned S3c slice added none (Q-S3c-1 (a), below), and the
  review adds 3, giving 30.
- Q-S3c-1 (a): the six universe guards stay, and the planned C case "a
  stuck `hcomp` in the universe at a neutral face round-trips" is dropped.
  With the first guard (`CLOS_HCOMP_PI`) lifted, the path regression
  `universe function composition` stops with `OOM`. Cause: transport of
  `idEquiv` along the `Equiv` line has no regularity, so `e` expands the
  `isContr` proof into nested stuck `hcomp Nat` terms, also on a constant
  line. `value_print` of the def value uses 456 MB of arena (the
  regression arena is 128 MB) and gives 14302 chars (the buffer is 8192).
  With a 1 GB arena the print completes, but the replay is refused:
  `GLUE_EQUIV: replay: ... TYPE_MISMATCH` (a printed sub-term is typed
  only under its face, and the checker has no face substitution). Guards 2
  to 6 give the same `OOM` in the other universe regressions.
- Q-S3c-2 (a): `transTwoLoops` is the inline form `fun (j : I) => hcomp
  (Type 0) (~j \/ j) (fun (i : I) => [ ~j -> Fin 3, j -> loopRot1 @ i ])
  (loopRot1 @ j)`. The planned `trans (Type 0) ...` is refused
  (`TYPE_MISMATCH`: Type 1, not Type 0), because the domain `trans` takes
  `A : Type 0`.
- `hcompGlue`. The planned closed form is a JSON instance whose type holds
  the Glue type `loopRot1 @ j`, and `langc build` refuses it (`JSON_TYPE`:
  the instance type has no source syntax). So `hcompGlue` is a Pi def
  `(x : Fin 3) -> PathP ...`, and finAll checks `x` = 0, 1, 2. With the
  planned face `~j`, the type fails at a neutral `x` (`PATH_END`: the `i1`
  end stays a stuck `hcomp (Fin 3) i0 ...`, no regularity). So the system
  is `(~j \/ j)`, with the branch `gRot x @ j` on each face.
- `uaArrowPlus3`. A top-level path over `ua (Fin 11) (Fin 11) e @ i` is
  refused by `langc build` in the same way (`JSON_TYPE`). So
  `uaArrowPlus3` is a Pi def over `f : Fin 11 -> Fin 11`, and the closed
  ends `uaArrowEnd0` and `uaArrowEnd1` are the JSON instances.
- G21 and G22 are new rows, for FIX B parts 1 and 2. FIX B part 3 (the
  record scan) has no row.

Mutants: one at a time, `make -B` for each build. Each file was restored and
its `shasum` was confirmed. Each kill came from a test, not from a compile
error.

| Mutant | Change | Killed by |
|---|---|---|
| G17 | `hcomp_glue`: the `T` part dropped (`glue` of `a1` only) | `univ.lang` (`hcompGlue`: `FIN_CASE`, case 0: `PATH_END`, the `i1` end is `1`, not the `glue`), `univ transTwo`, `univ.json` |
| G18 | `kan_apply` `CLOS_EQUIV_LINE`: `b j`, not `b (~j)` (the line not reversed) | `univ.lang` (`transTwo0`: `PATH_END`, `0`, not `2`), `univ transTwo`, `univ.json` |
| G19 | `rule_hcomp`: `HCOMP_UNIVERSE` put back before the face check | `univ.lang` (`transTwoLoops`: `HCOMP_UNIVERSE`), `univ.json` |
| G20 | `rule_hcomp`: no cap check at the universe | `hcomp-universe-cap` (`PATH_END`, not `HCOMP_CAP`) |
| G21 | `hcomp`: `hcomp_true` kept in `Type 0` (FIX B part 1) | `univ.lang` (`transTwo0`: `INTERNAL`, a line that changes its type former), `univ.json` |
| G22 | `reduce_glue`: at a true key, record and return `T`, not a fresh copy (FIX B part 2) | `univ.lang` (`transTwo0`: `PATH_END`, `0`, not `2`), `univ.json` |

G18, G21 and G22 were first killed by scratch probes (session 20); the
table gives the kills by the repo tests (session 21).

Review fixes:

- System conversion compares branch values and face coverage in both
  directions. A Glue composition with keys `~j, j` now agrees with a
  value written using the combined key `~j \/ j`, which the checker splits
  in the opposite order. Redundant subdivisions also agree. Different
  branch values and missing coverage still compare unequal.
- `glue_restrict` selects a branch when the requested face implies its
  key. A two-dimensional constant composition whose Glue type has an
  additional `i /\ ~j` branch now computes when the element uses `~j`.
  Exact key comparison previously left a Glue element inside an hcomp
  at `Fin 3`, causing `PATH_END`.
- `hcomp` above `Type 0` is refused with `HCOMP_UNIVERSE`. Previously
  `hcomp (Type 1) i0 (fun (_ : I) => []) (Type 0)` checked but normalized
  to `Glue (Type 0) i0 (fun (_ : I) => [])`, which fails to check because
  Glue is restricted to small types.

The three new C path cases check system conversion, reordered Glue
composition and stronger Glue faces. They fail against the original
staged evaluator and pass with the fixes. The check refusal
`hcomp-higher-universe` covers the universe restriction. The full gate,
including the clang pass, is green at the counts above.

## S4a

S4a (path constructor declarations, `S1`, `fold` with path cases, formal
`hcomp`, `transp` at a family with no parameters, JSON rule 4) is
complete. A domain constructor can take point fields, then 1 or 2
interval fields, then a boundary system (R1 (a)). The checker checks each
branch at the family type at the parameters (`hcomp_branch`), the cover
(`SYSTEM_COVER`) and the overlaps (`SYSTEM_OVERLAP`); a path constructor
with no boundary, more than 2 interval fields, or a point field after an
interval field gives the new code `HIT_BOUNDARY`. `domain/domain.lang`
declares `family S1 := base | loop (i : I) [ ~i -> base, i -> base ]`. A
path constructor value is `OP_CTOR` (R2 (a)): `reduce_ctor` evaluates
the boundary at the interval arguments, a true key gives its branch (`loop
i0` is `base`), and else the value is canonical. `fold` takes one case for
each constructor (R4 (a)). The case of a path constructor has the type
`(fields) -> PathP (fun (_ : I) => RES) B0 B1`, with `B0` and `B1` the
fold images of the boundary at `i0` and `i1`, and `fold (loop r)` is
`case @ r` (`fold_path`). `hcomp` at a family with a path constructor
(R3 (a)) is `hcomp_true` at a true key, and else the formal value `VAL_OP
OP_HCOMP [ty, phi, sys, base]` (`hcomp_formal`), also at `i0` and with a
neutral base. `fold` of a formal `hcomp` is `hcomp RES phi (fun (j : I)
=> [ k -> fold u, ... ]) (fold u0)`, through the new closure
`CLOS_SYS_FOLD`, and `reduce_proj` of it is stuck, not `INTERNAL`.
`transp_data` gives the base at an `OP_FAMILY` with no parameters (the
line is constant by the form of the family, not by regularity); a family
with parameters stays stuck (S4b item 11). `conv_loop` does not convert a
path constructor value with its boundary point, so `loop` is not `refl`.
The JSON writer (R9 (a)) prints a path at a HIT whose body mentions the
path variable as a tree (rule 4): `{"path": NAME, "args": [...]}` for a
path constructor at the variable and `{"hcomp": {"phi": ..., "base":
...}}` for a formal `hcomp`. `family_json` prints a formal `hcomp` in the
same form.

Gate: `make check` is GREEN at examples 16, parse 11, check 56, evals 37,
json 16, build refusals 4, regressions 6, printer 2, path regressions 39
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: `examples/circle.lang` (`loopPath`, `transpBase`, `code2`, `loopT`,
`loopAt0`, `code3`, `loop2`, `loop2T`, `loop2At0`) and
`test/json/circle.json` (7 instances; `code2` and `code3` are functions
into `Type 0` and do not emit). `code2` folds `S1` into `Type 0` with the
cases `Fin 2` and `ua` of `r -> finAdd 2 r 1`, and `loopT` checks that the
transport along `code2 (loop i)` moves `0` to `1` (`loop` is not
`refl`). `loop2 := fun (j : I) => hcomp S1 (~j \/ j) (fun (i : I) => [ ~j
-> base, j -> loop i ]) (loop j)` is a formal `hcomp`, and `loop2T` checks
that the transport along `code3 (loop2 @ j)` moves `0` to `2` (the fold
of `loop2` is the S3c `transTwoLoops` composite). In `circle.json`,
`loopPath` is `{"path": "loop", "args": []}` and `loop2` is `{"hcomp":
{"phi": "i \/ ~i", "base": {"path": "loop", "args": []}}}`. The
check refusals `fold-missing-path` (`TYPE_MISMATCH`),
`fold-path-boundary` (`PATH_END`) and `loop-not-refl` (`PATH_UIP`). The
evals `circle loopAt0` (`1`) and `circle loop2At0` (`2`). The C path cases
"path constructor declaration" (domain mode, accepted), "path
constructor boundary of a wrong type" (`TYPE_MISMATCH`), "path
constructor boundary that does not cover" (`SYSTEM_COVER`), "path
constructor with no boundary" (`HIT_BOUNDARY`), "formal hcomp at a HIT"
and "path constructor at a neutral interval" (`loop (i /\ j)`, a
connection square), the last two as round-trips.

Deviations from the plan (kickoff section 2; the kickoff text gives the
rulings):

- Counts. Before review, path regressions were 36, not 35: next to the 3 planned
  declaration refusals, a `domain_accepts` case ("path constructor
  declaration") checks that the domain accepts a path constructor.
- The fold of a constructor with 2 interval fields, and a `p @ j`
  boundary image, move to S4c (`HIT_BOUNDARY` in S4a): no family of
  dimension 2 exists before S4c item 15.
- `OP_FOLD_FAMILY` carries the result type `RES` as one more argument,
  before the scrutinee. The planned fold term had no result type, and
  the fold of a formal `hcomp` is an `hcomp` in `RES` (for `code3`, an
  `hcomp` in `Type 0`, so `hcomp_univ` makes the Glue that `loop2T`
  transports). The printer does not print `RES`.
- `glue_at` has a fallback that the kickoff did not plan: when no Glue
  record matches and the line at the end is not a Glue type, it takes
  the Glue of the line at a fresh variable and puts the end in for the
  variable (`val_isubst`). Transport along a fold of a path constructor
  (`loopT`) needs it: at an end, the fold gives the end case, not the
  case path at the end, so no Glue record is made.
- JSON rule 5 comes before rule 4 (the order is rule 1, 5, 4, 6; the
  kickoff put rule 4 right after rule 1). A HIT path whose body does not
  mention the variable stays `{"refl": ...}`, so a point keeps the path
  form. Rule 4 applies to a path constructor with 1 interval field only:
  a constructor with 2 interval fields, a point constructor or a field
  that is not of the family that mentions the variable goes to rule 6
  (S4c items 17 and 18 extend `hit_tree`). In `"phi"`, the path variable
  prints as `i`, and the backslash of `\/` is written as `\`.
- The `{"trunc": a}` form of `trunc1` and `trunc0` moves to S4b, with
  `Trunc1`.
- The evals name the value defs `loopAt0` and `loop2At0` (the
  `glue.lang` `uaAt9` form), not the path defs `loopT` and `loop2T`: an
  eval of a path def gives a path, not the `Fin` value.
- `loop-not-refl` gives `PATH_UIP` (the kickoff allowed `PATH_UIP` or
  `PATH_END`).
- `test/check/self.lang` renames its def `loop` to `spin`: `loop` is now
  a domain constructor, and a program def with its name is refused
  (`REFUSE_NAME`).
- H4 is not killed by the planned `fold-missing-path`: `fold 0 x` is
  `TYPE_MISMATCH` with and without the mutant, as a missing fold case is
  `TYPE_MISMATCH`. The mutant refuses each correct `S1` fold, so
  `circle.lang` (`code2`) and `fold-path-boundary` kill it. No new
  refusal or def is necessary.
- G1 is a new row, for the `glue_at` fallback.

Mutants: one at a time, `make -B` for each build, by hand. Each file was
restored and its `shasum` was confirmed. Each kill came from a test, not
from a compile error. H6 and H8 add a `(void)` use, and H1, H2, H4 and H5
add a static next to the def that they change, so that the clang pass
compiles the mutant.

| Mutant | Change | Killed by |
|---|---|---|
| H1 | `hcomp_branch`, `boundary_core`: a boundary branch infers its type, with no check at the family | C case "path constructor boundary of a wrong type" |
| H2 | `system_branches`, `boundary_core`: no `SYSTEM_COVER` for the boundary | C case "path constructor boundary that does not cover" |
| H3 | `reduce_ctor`: no boundary reduction | `circle.lang` (`loopPath`: `PATH_END`, the end stays `loop i0`), `loop-not-refl`, `fold-path-boundary`, both evals, C cases "formal hcomp at a HIT" and "path constructor at a neutral interval" |
| H4 | `rule_fold`: the fold arity counts the point constructors only | `circle.lang` (`code2`: `TYPE_MISMATCH`), `fold-path-boundary` (`TYPE_MISMATCH`, not `PATH_END`), both evals |
| H5 | `fold_family`: a path case infers its type (its own ends) | `fold-path-boundary` (accepted) |
| H6 | `fold_path`: the fold at `i0`, not `case @ r` | `circle.lang` (`loopT`: `PATH_END`, `0`, not `1`), both evals, `circle.json` |
| H7 | `hcomp`: the true-key test skips a family | `circle.lang` (`loop2`: `PATH_END`), both evals, `circle.json`, C case "formal hcomp at a HIT" |
| H8 | `fold_family`: `stuck_fold` in place of `fold_hcomp` | `circle.lang` (`loop2T`: `PATH_END`), both evals, `circle.json` |
| H9 | `conv_loop`: an `OP_CTOR` with an interval field converts with each `OP_CTOR` of the same family | `loop-not-refl` (accepted) |
| H10 | `transp_data` at `OP_FAMILY`: no `return base` | `circle.lang` (`transpBase`: `PATH_END`), both evals |
| H11 | `path_json`: no rule 4 (`hit_family` skipped) | `circle.json` |
| G1 | `glue_at`: the fallback returns `INTERNAL` at once | `circle.lang` (`loopT`: `INTERNAL`), both evals, `circle.json` |

Review fixes: `val_isubst` preserves De Morgan interval terms when
recovering a Glue endpoint. Face normalization remains limited to faces,
so a connection such as `j /\ ~j` is preserved inside path constructors.
A transported path with this connection now prints a normal form that
checks on replay. `hit_tree` keeps each recursive field's completion flag
local, so a later field that varies along the path triggers rule 6 instead
of emitting an incomplete JSON object. Three C cases cover the connection
round-trip, the JSON fallback and the supported recursive path tree. The
two defect cases fail against the original staged sources and pass with
the fixes; the full gate is green at the counts above.

## S4b

S4b (recursive fields under parameters, `Trunc1`, `transp` at a family
with parameters, the `{"trunc": a}` JSON form) is complete. A domain
family with parameters can have a recursive field of the type `F p1 ..
pn`, the family at its own parameter variables (`own_family`); a
recursive field at other arguments gives `DOMAIN_FAMILY`.
`domain/domain.lang` adds `isProp` and `isSet` (after `isContr`) and
`family Trunc1 (A : Type 0) := trunc1 (truncOf : A) | squash1
(squashFrom : Trunc1 A) (squashTo : Trunc1 A) (i : I) [ ~i ->
squashFrom, i -> squashTo ]`. `transp` at a family with parameters (R7
(a)) is `transp_family`: at a constructor, it gives the same constructor
at the parameters of the line at `i1`; each point field goes along its
field type at the parameters of the line (the new closure
`CLOS_FAMILY_FIELD`, evaluated by `field_type_at`), a recursive field
goes along the line, and the interval fields do not change. At a formal
`hcomp`, it gives `hcomp` at the line at `i1`, with each tube transported
(the new closure `CLOS_SYS_TRANSP`) and the base transported. A neutral
value stays stuck. `family_json` prints a `trunc1` or `trunc0` point as
`{"trunc": a}`.

Gate: `make check` is GREEN at examples 17, parse 11, check 56, evals 38,
json 17, build refusals 4, regressions 6, printer 2, path regressions 40
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: `examples/trunc.lang` (`sq1`, `map1`, `map1At`, `rot3`,
`truncUa`, `truncAt`, `truncSq`, `truncHcomp`) and `test/json/trunc.json`
(7 instances; `map1` is a function with a `Type 0` binder and does not
emit). `map1` folds `Trunc1 A` into `Trunc1 B` with the cases `fun a =>
trunc1 B (f a)` and `squash1 B` (eta-expanded), and `map1At` checks it at
`r -> r + 1`. `truncUa` checks that the transport along `Trunc1 (rot3 @
i)` moves `trunc1 (Fin 3) 0` to `trunc1 (Fin 3) 1`. `truncSq` checks that
the transport of `squash1 (Fin 3) (trunc1 (Fin 3) 0) (trunc1 (Fin 3) 1)
j` is `squash1 (Fin 3) (trunc1 (Fin 3) 1) (trunc1 (Fin 3) 2) j`, as a
path of paths (the ends alone do not show the recursive fields).
`truncHcomp` checks that the transport of a formal `hcomp` with the tubes
`trunc1 (Fin 3) 0` and `squash1 ...` moves each tube and the base. In
`trunc.json`, `sq1` is `{"path": "squash1", "args": [{"trunc": 0},
{"trunc": 1}]}` and `truncAt` is `{"trunc": 1}`. The eval `trunc truncAt`
(`trunc1 (Fin 3) 1`). The C path case "transport of a formal hcomp at
Trunc1 along a parameter line" (a neutral line `Trunc1 (p @ i)`, so the
fields of the tubes stay stuck transports) round-trips.
`test/regressions.js` also refuses a recursive field at other arguments
(`DOMAIN_FAMILY`).

Deviations from the plan (kickoff section 2; the kickoff text gives the
rulings):

- Counts. Path regressions are 40, not 36: S4a ended at 39 C cases (3
  review-fix cases and 1 `domain_accepts` case), and S4b adds 1.
- The `Trunc1` fields are `truncOf`, `squashFrom` and `squashTo`, not `a`,
  `x` and `y`. A domain field name is a core name (R4): a field `x` made
  the printer rename a program binder `x` to `_x0`, and the eval
  `universes Endo` failed.
- `trunc.lang` names the `ua` path `rot3 : Path (Type 0) (Fin 3) (Fin 3)`
  (the `glue.lang` form) and uses `Trunc1 (rot3 @ i)` as the line. Its
  type holds no Glue type and no `Equiv`, so `langc build` prints it by
  rule 1 (`[1, 2, 0]`). The equivalence stays inline in `rot3`.
- The eval names the value def `truncAt` (the S4a `loopAt0` form), not
  the path def `truncUa`: an eval of a path def gives a path.
- `{"trunc": a}` applies in each JSON rule that prints a `trunc1` or
  `trunc0` value (rule 4 path arguments, `{"refl": ...}`, and a top-level
  value), not only in rule 4.

Mutants: one at a time, `make -B` for each build, by a runner
(`lang-template-work/coin-lang-S4b-mut/mut.sh`: a site assert, a one-line
edit of `src/front/eval.c`, restore). Each file was restored and its
`shasum` was confirmed (a `src` shasum file before and after). Each kill
came from a test, not from a compile error. M9a ran by hand in item 9.

| Mutant | Change | Killed by |
|---|---|---|
| M9a | `own_family`: the arguments of the field type are not compared | `test/regressions.js` (the recursive field at other arguments is accepted) |
| H12 | `transp_family`: each point field copied, not transported | `trunc.lang` (`truncUa`: `PATH_END`, the end stays `trunc1 (Fin 3) 0`), `trunc truncAt`, `trunc.json` |
| H13 | `transp_family`: a recursive field copied, not transported along the line | `trunc.lang` (`truncSq`: `PATH_UIP`), `trunc truncAt`, `trunc.json` |
| H14 | `transp_family`: at a formal `hcomp`, the system kept, the tubes not transported | `trunc.lang` (`truncHcomp`: `PATH_UIP`), `trunc truncAt`, `trunc.json` |

### S4b review fix

The original constructor transport rule did not commute with general HIT
boundaries. In a family `ParamHit a` with a boundary `paramPoint a a`,
transport from `a = trunc1 (Fin 3) 0` to `a = trunc1 (Fin 3) 1` normalized
a valid loop with endpoints `paramPoint a 0` to one with endpoints
`paramPoint a 1`. Its printed normal form failed replay with `PATH_END`.

`field_transport_boundary` now restricts direct constructor rebuilding to
boundaries whose faces use constructor interval fields and whose branches
are recursive fields. Other path constructors retain opaque transport
until a general boundary correction is implemented. Point constructors,
`squash1` and formal compositions retain their S4b reductions.

The new C case checks the parameter-dependent boundary through printing,
rechecking and conversion. It passes with the base evaluator, fails on
the original staged evaluator and passes with the fix. `make check` and
the clang pass are green; path regressions are now 41, with 0 failures.

## S4c

S4c (field telescopes, path fields, constructors with two interval
fields, `Trunc0`, `Quot`) is complete. A constructor field can use the
earlier fields (a field telescope): `check_field` pushes the earlier
fields, and `ctor_app` checks each field at the earlier field values. A
field `Path F x y` (with `x` and `y` earlier recursive fields) is a path
field, and `fold` gives it to its case as `fun (k : I) => fold (p @ k)`
(`fold_path_field`). A constructor can have two interval fields: its
boundary is a square with the four faces `~i`, `i`, `~j` and `j`, and its
fold case has the square type over the folded fields (`square_type`).
`domain/domain.lang` adds `family Trunc0 (A : Type 0)` (`trunc0` and the
square `squash0`) and `family Quot (A : Type 0) (R : A -> A -> Type 0)`
(`quot`, the path `quotEq` and the square `quotSquash`). The `squash0`
and `quotSquash` fold cases have the type `isSet RES` (R8 (a)). The
transport of `squash0` or `quotSquash` along a parameter line stays stuck
(R7 (a)): `field_transport_boundary` gives 0, because their branches are
path applications (`squash0Left @ j`), not recursive fields.

Gate: `make check` is GREEN at examples 18, parse 11, check 57, evals 39,
json 18, build refusals 4, regressions 6, printer 2, path regressions 44
(0 failed). The clang `-Wswitch-enum` pass is clean. The dash scan is
clean.

Tests: `examples/quot.lang` (`map0`, `map0At`, `sq0`, `map0Sq`,
`parity`, `Par`, `qPath`, `toTrunc`, `toTrunc2`, `toTrunc3`,
`toTruncPath`) and `test/json/quot.json` (6 instances; the function defs
do not emit). `map0` folds `Trunc0 A` into `Trunc0 B` with the cases `fun
a => trunc0 B (f a)` and `squash0 B` (eta-expanded). `sq0` gives a square
between `refl` and `trans ... refl refl` by `squash0` (they are not equal
by conversion, because there is no regularity). `map0Sq` checks, at a
neutral `x` and `p`, that `map0` of a `squash0` square is the `squash0`
square of the folded path fields. `qPath` is `{"path": "quotEq", "args":
[0, 2, {"refl": 0}]}` (rule 4). `toTrunc` folds `Quot (Fin 4) Par` into
`Trunc0 (Fin 4)`, `toTrunc2` checks it at `2` by `refl`, and
`toTruncPath` checks the fold of `qPath`. The check refusal
`trunc0-no-set` (a fold of `Trunc0 (Fin 2)` into `Nat` with the point case
only) gives `TYPE_MISMATCH`. The eval `quot toTrunc3` gives `trunc0 (Fin
4) 2`. The C path cases: the transport of `squash0` along a parameter
line stays stuck and round-trips; the stuck square is not the field
rebuild (`PATH_UIP`); a dimension-2 boundary with the faces `~i`, `i` and
`~j` only gives `SYSTEM_COVER`.

Deviations from the plan (kickoff section 2; the kickoff text gives the
rulings):

- Parity ruling. There is no map from `Fin 4` to `Fin 2` (no `finVal`;
  `finAdd` and `finNeg` stay in `Fin n`). Thus `parity : Fin 4 -> Fin 4
  := fun a => finAdd 4 a a` (0 for an even `a`, 2 for an odd `a`), `Par a
  b := Path (Fin 4) (parity a) (parity b)`, and `toTrunc` goes into
  `Trunc0 (Fin 4)`. The eval `quot toTrunc3` gives `trunc0 (Fin 4) 2`, not
  `trunc0 (Fin 2) 1`.
- Counts. Path regressions are 44, not 38: S4b ended at 41 (with the
  review fix case), and S4c adds 3 (the kickoff 2 and the
  not-the-field-rebuild refusal that pins R7 (a)).
- The `Trunc0` fields are `trunc0Of`, `squash0From`, `squash0To`,
  `squash0Left` and `squash0Right`, and the `Quot` fields are `quotOf`,
  `quotA`, `quotB`, `quotR`, `quotFrom`, `quotTo`, `quotLeft` and
  `quotRight` (a domain field name is a core name, R4).
- The eval names the value def `toTrunc3`, not a path def (the S4b
  `truncAt` form). `quot.lang` adds `map0At` and `toTruncPath`.
- H16 is killed by `quot.lang`, not by `trunc0-no-set`: under H16 the
  refusal still gives `TYPE_MISMATCH` (the arity passes, and the
  scrutinee is then checked as the `squash0` case).
- H18 is killed by each example: the `Trunc0` declaration in the domain
  gives `SYSTEM_COVER` (the keys must equal the cover). The C case with 3
  faces stays the killer of a cover that is too small.
- Not run: the item 13 self-review mutants and the candidate mutant
  `field_transport_boundary` gives 1 (expected killer: the
  not-the-field-rebuild C case).

Mutants: one at a time, `make -B` for each build, by the runners in
`lang-template-work/coin-lang-S4c-mut/` (`lib.sh`, a copy of the S4a one:
a site assert, an awk edit, the gate, the C cases, restore). Each file was
restored and its `shasum` was confirmed (15 files, all OK). Each kill came
from a test, not from a compile error.

| Mutant | Change | Killed by |
|---|---|---|
| H15 | `fold_family`: a path field goes to its case as `p`, not as `fun k => fold (p @ k)` | `quot.lang` (`map0Sq`: `PATH_UIP`), `quot toTrunc3`, `quot.json` |
| H16 | `rule_fold`: the fold arity does not count the constructors with two interval fields | `quot.lang` (`map0`: `TYPE_MISMATCH`, 3 arguments), `quot toTrunc3`, `quot.json` |
| H17 | `path_case_type`: the ends B0 and B1 swapped | `quot.lang` (`toTrunc`: `PATH_END`), `trunc.lang` (`map1`: `PATH_END`), `trunc truncAt`, `quot toTrunc3` |
| H18 | `boundary_core`: the cover has the faces of the first interval field only | each example (`Trunc0`: `SYSTEM_COVER`), the domain C cases |
## S4c review

Fixed a type-preservation defect in the fold images of boundary point
constructors: `fold_image` passed path fields unchanged while the evaluator
folded them. A fold into `Fin 2` could therefore check with boundary ends
in the source family and normalize to a source constructor instead of a
`Fin 2` value. Boundary images now map a path field under a fresh interval
binder and preserve the surrounding field and interval environment.

Two C round-trip regressions cover the wrong-family normal form and a
square boundary with a path field that captures a surrounding interval.
The first fails on the original staged checker with `TYPE_MISMATCH` when
its printed normal form is checked again. The second also fails if its
surrounding interval is reused as the fresh path binder (`INTERNAL`).
`make check`, including the clang pass, is green; path regressions are now
46 with 0 failures. No CI or test gate was weakened.
