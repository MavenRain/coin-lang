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
