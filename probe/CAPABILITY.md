# Host capability probe: tcc-json

Date: 2026-10-16. Host tree: this repository at `31d73f8`, read only.
Driver: `build/langc`, built 2026-10-16 by `make -B check` from the
sources of `5877e1d`. That commit is the last commit that changes `src/`,
`domain/`, `gen/`, the `Makefile` or `test/gate.sh`. This file gives the
facts that the repository records (the host README, STATUS and the tests).
S8 also ran one new probe (section 8) and four cost runs (section 6) (S8
ruling R2 (a)). Other work can load the machine. Thus each wall time is an
upper bound.

To run the probe again: `make -B check` runs the gate. The gate checks
each refusal of sections 3, 4, 7 and 10 (`test/check/expect.txt`,
`test/parse/expect.txt`). The Sigma eta probe of section 8 is two scratch
programs: write each one to a file and run `build/langc check FILE`. The
probe writes nothing in the repository. `probe/guard.py` stops a command
above a resident memory limit or a time limit; S8 did not use it. The
largest process in the probe is `build/langc build examples/schnorr.lang`,
with a peak of 1011.9 MiB (section 6).

## Questions

Each host must answer these questions. Each answer is one numbered section
below. An answer sets the status of a former in `formers/tcc-json.md`.

| Question | Section | Formers |
|---|---|---|
| Which commands check a file, print a normal form, build and run? | 1 | all |
| Does the host take several files, or must the driver join them? | 2 | all |
| Which recursion does the host accept, and how does it refuse the rest? | 3 | F6, F7 |
| Which `Nat` primitives exist? Does `Nat` have an eliminator? | 4 | F6, F7 |
| How does a result leave the host? | 5 | target |
| What do check and build cost (time and resident memory) at N = 100 and N = 1000? | 6 | all |
| Can a family take a parameter? Can a constructor of such a family appear in a term? | 7 | F4, F12, F15 |
| Can a program project a Sigma? Does the kernel accept Sigma eta? | 8 | F11 |
| How does a program use equality: transport, cong, an index type? | 9 | F12, F13 |
| How does the host report an axiom? | 10 | R2 |

## 1. Commands

| Job | Command | Result |
| --- | --- | --- |
| Check | `build/langc check PROG` | exit 0 and `ok` on stdout; exit 1 and `langc: CODE: NAME: message` on stderr |
| Print normal form | `build/langc eval PROG NAME [ARGS...]` | the normal form of `NAME` applied to the literal arguments, on stdout |
| Build | `build/langc build PROG [-o OUT]` | one JSON document on stdout, or in the file `OUT` |
| Run | none | the JSON document is the result; there is no run step |

- The usage text is at `src/main.c:37-39`. Exit 0 is ok, exit 1 is a
  refusal, and exit 2 is a usage or IO error (`src/main.c:2`).
  `test/gate.sh` checks exit 2 for a usage error, a missing file and a
  build usage error (lines 93, 96 and 133).
- A refusal is one stderr line. A refused build writes nothing to stdout
  and makes no `OUT` file (`docs/host/README.md:34`, `test/gate.sh:126-130`).
- One program file for each run (`parse_options`, `src/main.c:62-78`).

## 2. Several files

A run reads one program file. There is no import form. The domain joins at
build time: the Makefile runs `tcc -run gen/embed.c domain/domain.lang
build/domain.c` (`Makefile:15-17`), and `build/langc` links
`build/domain.c`. Thus a program uses the domain names with no import.

## 3. Recursion and totality

A program cannot recurse by name. The checker refuses the three recursion
forms of `test/check/rec.lang`, `test/check/self.lang` and
`test/check/later.lang` with `REFUSE_REC` (`test/check/expect.txt:3-5`).
Recursion comes only from `fold` (over Nat, List and each family) and
`unfold` (into List, with a Nat step limit) (`formers/tcc-json.md` F6 and
F7, `examples/algebra.lang`). The evaluator stops at a depth of 2000 and
after 20,000,000 steps for each instance (`docs/host/README.md` `##
Limits`, `src/front/core.h:16-17`).

## 4. Nat

`Nat` is an unsigned 64-bit integer (`docs/host/CAPABILITY.md:19`). Thus
the largest `Nat` is 18446744073709551615 (2^64 - 1). The lexer refuses a
larger literal with `LEX_NAT_RANGE` (`test/parse/nat-range.lang`,
`test/parse/expect.txt:2`). The primitives are `natAdd`, `natSub`,
`natMul`, `succ`, `natEq` and `natLe` (BUILTINS, `src/front/check.c:153-158`).
`natAdd` and `natMul` trap on overflow, and `natSub` is truncated at 0
(`docs/host/CAPABILITY.md:21-24`). `langc eval` prints `trap`, and `langc
build` refuses an instance that holds a trap (`EVAL_OVERFLOW`). `Nat` has
an eliminator: `fold step start n` applies `step` n times (`square`,
`examples/algebra.lang:4`).

## 5. How a result leaves the host

`langc check` prints `ok` on stdout. `langc eval` prints a normal form on
stdout. `langc build` writes one JSON document on stdout, or in `OUT` with
`-o` (`docs/host/README.md:29`). The document is the only output of a
program.

## 6. Cost

Cost at N = 100 and N = 1000 was not probed: a coin-lang program has no
size parameter (S8 ruling R2 (a)).

| Step | N = 100 | N = 1000 |
| --- | --- | --- |
| Check | not probed | not probed |
| Build | not probed | not probed |
| Run | none | none |

S8 measured the two HVZK examples on 2026-10-16: one run of each step, in
series. Each cell gives the wall time and the peak resident memory of the
`build/langc` process (`getrusage`, `ru_maxrss`, in MiB).

| Step | `examples/schnorr.lang` | `examples/gi.lang` |
| --- | --- | --- |
| Check | 0.57 s, 714.3 MiB | 0.30 s, 577.3 MiB |
| Build | 0.51 s, 1011.9 MiB | 0.38 s, 572.4 MiB |

Each build wrote the same bytes as its golden (`test/json/schnorr.json`,
`test/json/gi.json`). The arena limit is 1 GiB (`src/main.c:10`). The peak
resident memory changes from run to run. At the S8 close, one more run of
each schnorr step gave 0.99 s, 995.0 MiB (check) and 1.00 s, 790.3 MiB
(build). The cost of the whole gate is in `docs/VALIDATION.md`.

## 7. Parametric families

A domain family can take a parameter, and its constructors can appear in a
term. The domain has `Lot (size : Nat)`, `Trunc1 (A : Type 0)` and `Quot
(A : Type 0) (R : A -> A -> Type 0)` (`domain/domain.lang:8`, `15` and
`29`). `examples/inventory.lang:9` builds `makeLot 3 250` of type `Lot 3`.
A path constructor also appears in a term: `loop i` in
`examples/circle.lang:6`. The checker refuses a program family with
`REFUSE_DATA` (`test/check/data.lang`, `test/check/expect.txt:1`). Thus
each family is in `domain/`.

## 8. Sigma projection and eta

`witness` and `payload` are builtins (BUILTINS, `src/front/check.c:133-134`),
and a program uses them (`gSig`, `examples/bag.lang:41`; `fwd`,
`examples/finset.lang:15`). A pair is `pack a b`, and a pair type is `(e :
A) * B`. A product `A * B` does not parse (`docs/PLAN.md:41`). The probe
did not try a Sigma pattern.

The kernel accepts Sigma eta. The conversion `conv_pair`
(`src/front/eval.c:3158-3164`) compares a pair with the projections of a
neutral, and `docs/host/README.md:271` names Product and Sigma eta. The S8
probe (2026-10-16) checked this program with `build/langc check`:

```
def etaFin : (s : (e : Fin 2) * Fin 3) -> Path ((e : Fin 2) * Fin 3) (pack (witness s) (payload s)) s := fun (s : (e : Fin 2) * Fin 3) => refl
def etaGen : (A : Type 0) -> (B : A -> Type 0) -> (s : (a : A) * B a) -> Path ((a : A) * B a) (pack (witness s) (payload s)) s := fun (A : Type 0) => fun (B : A -> Type 0) => fun (s : (a : A) * B a) => refl
```

The result is exit 0 and `ok`. The control program puts the payload of a
second neutral `t` in the pair:

```
def etaBad : (s : (e : Fin 2) * Fin 3) -> (t : (e : Fin 2) * Fin 3) -> Path ((e : Fin 2) * Fin 3) (pack (witness s) (payload t)) s := fun (s : (e : Fin 2) * Fin 3) => fun (t : (e : Fin 2) * Fin 3) => refl
```

The result is exit 1 and `langc: PATH_END: etaBad: the path at i1 is pack
(witness s) (payload t), but the type needs s`. The examples use Sigma eta:
`retEta` (`examples/finset.lang:19`) closes a law at a neutral `p` with
`retCurried (witness p) (payload p)`.

## 9. Equality

A program writes `Path A x y` and `PathP` with `refl`, `sym`, `cong`,
`funExt`, `transp` and `hcomp` (BUILTINS, `src/front/check.c:119-147`). The
domain adds `transport`, `subst`, `J` and `JRefl` (`domain/domain.lang:47`,
`49`, `52` and `55`). One former, `Path`, serves each type. The kit `Eq` is
gone since S1 (O8, `docs/PLAN.md:336-338`): `Path A a b` is `PathP (fun _
=> A) a b`. A domain HIT adds path constructors, for example `loop` of
`S1` (`domain/domain.lang:11`). No proof reduces by irrelevance: in the
JSON, a path is never null (`src/json.c:108-109`).

## 10. Axiom detection

The checker refuses an axiom in a program with `REFUSE_AXIOM`
(`test/check/axiom.lang`, `test/check/expect.txt:2`). Thus a program that
checks has no axiom, and there is no separate axiom report. The trust
points of the builtins are in `SPEC.md` section 8 (`carrierSum`).

## Consequences

1. All memory comes from one arena with a limit of 1 GiB (`src/main.c:10`,
   `docs/host/CAPABILITY.md:38`). The schnorr build used 1011.9 MiB at its
   peak (section 6).
2. There is no cumulativity. `def wrong : Type 1 := Nat` gives
   `TYPE_MISMATCH` (`test/check/cumulative.lang`), and `def Large : Type 2
   := Type 1` gives `TYPE_UNIVERSE` (`test/check/type2.lang`;
   `test/check/expect.txt:22-23`).
3. There are no implicit arguments (`docs/STATUS.md:1213`). Each library
   def takes its type arguments, for example `transport X Y p x`
   (`domain/domain.lang:47`).
4. The `finAll` rule is a fallback. When the ordinary check of a function
   fails on a `Fin n` domain with a literal `n`, the checker checks each
   case (`src/front/check.c:2346-2350`, `fin_cases` at 2291). Thus a
   pointwise law on a finite carrier is a def over `Fin` binders, and a
   law on a Sigma closes by eta (section 8).
5. The JSON has limits: the `type` text is at most 4096 bytes
   (`src/json.c:7`), and the nesting is at most 2000 levels
   (`src/json.h:15`). `langc build` refuses a program def of an `Equiv`
   type (`JSON_VALUE`) and a program def whose type holds a Glue type
   (`JSON_TYPE`) (`docs/host/README.md:239-244`).
6. The S6/S7 probes of nontrivial paths in `FinSet` and `Rand T`, and the
   S7b path between `giLoop 0` and `giLoop 1`, stop with `OOM`
   (`docs/PLAN.md` section 8, O13). Reflexive paths can check; the
   `finSetFin 1` reflexive probes build with `JSON_VALUE` when their
   proof payload reaches a function. Each HVZK path is a Pi def over `Fin 1`,
   with no JSON instance (`hvzk`, `examples/schnorr.lang:54`; `giHvzk`,
   `examples/gi.lang:70`).
