# coin-lang specification

Status: S0 to S8 DONE (2026-10-16). The name is coin-lang (O1).

The detail of each section is in `docs/PLAN.md` (the rulings and the
encoding), `docs/STATUS.md` (each slice) and `docs/host/README.md` (the
commands, the JSON document and the limits).

## 1. Purpose

A program describes uniform random variables on finite coin spaces and the
paths between them. A path between two random variables is a bijective
coupling of their coins. Thus perfect honest-verifier zero knowledge (HVZK)
is a path from the real transcript variable to the simulated one
(`design/DESIGN.md`, "Perfect HVZK as an equivalence of coin spaces").
`langc` checks the program, and it writes one JSON document with the value
of each instance. The host is tcc-json (the TinyCC kit). The target is
JSON.

The type formers are F1 to F15 of `formers/FORMERS.md`. The core data types
and the core operations are only those of `design/DESIGN.md` (the design),
on the minimum base span of `docs/PLAN.md` section 2. The kernel is a
cubical type theory in the CCHM form (O2): the interval, `PathP`, `transp`,
`hcomp`, Glue types, `ua` and higher inductive types (HITs). The kernel is
in this repository, in `src/front/`.

The kernel has no proof irrelevance (`design/DESIGN.md`). Two paths with the
same end points are not equal by conversion: the check refusal
`test/check/path-sym.lang` gives `PATH_UIP` (S1). Thus a loop and a
quotient can be non-trivial. The JSON document writes each path as data.

## 2. Programs

A program is a sequence of definitions:

```
def NAME : TYPE := TERM
```

The checker refuses these forms in a program (`test/check/expect.txt`):

| Code | Form |
|---|---|
| `REFUSE_DATA` | A `family` declaration, also an enumeration (`test/check/data.lang`). Each family is in `domain/` |
| `REFUSE_AXIOM` | An unproved fact (an `axiom` declaration, `test/check/axiom.lang`) |
| `REFUSE_REC` | General recursion: `def rec`, a use of the definition itself or of a later definition (`rec.lang`, `self.lang`, `later.lang`) |
| `REFUSE_NAME` | A definition that uses a core name, a domain name or a reserved name, or a duplicate name |

Recursion comes only from `fold` and `unfold` (F6, F7). A `fold` on a HIT
takes a path case for each path constructor (S4). The compiler writes the
target; a program cannot.

## 3. Type formers

The type formers F1 to F15 are on tcc-json (`formers/tcc-json.md`). The
table gives the deviations only.

| ID | Status on tcc-json | Effect on this language | Open item |
|---|---|---|---|
| F12 | Replaced by `Path` in S1 | The kit `Eq` is gone. `Path A a b` is `PathP (fun _ => A) a b`. `refl`, `sym`, `cong` and `funExt` are terms | O8 (closed) |
| F13 | Replaced by `transp` in S1 and S2a | Transport is `transp` on a line of types. The library gives `transport`, `subst`, `J` and `JRefl` (`domain/domain.lang`) | O8, O11 (closed) |
| F14 | DONE: `Type 0` and `Type 1`, without cumulativity | `FinSet`, `Rand T` and `Bag T` are in `Type 1`. `test/check/cumulative.lang` gives `TYPE_MISMATCH` and `test/check/type2.lang` gives `TYPE_UNIVERSE` | none |
| F15 | DONE in `domain/` only | A program cannot declare a family. The domain families are `S1`, `Trunc1`, `Trunc0`, `Trunc0One`, `Quot` and the kit sample `Lot` | none |

The language adds these forms to the formers:

- S1: the interval `I` with `i0`, `i1`, `~`, `/\` and `\/`; `PathP`, path
  abstraction and path application (`p @ r`).
- S2: `transp` and `hcomp`, with systems of faces.
- S3: `Fin n` and the `finAll` rule (a Pi def over a literal `Fin n` checks
  at each point); `Glue`, `glue` and `unglue`; `hcomp` in the universe.
- S4: HITs with path constructors, formal `hcomp` and `transp` at a family
  with parameters.
- S5: `finCase`, `finVal`, `finAdd`, `finNeg`, `finMul`, `finEq`,
  `decEqFin` and `decEqNat`.
- S6: the builtins `carrierSum` and `fiberMatch` (section 8).

## 4. Structures

- **Monad**: `pure`, `map`, `bind` on Option, Sum E and List (the kit
  builtins). On `Rand T` the domain defs `randPure`, `randMap` and
  `randBind` (S6).
- **Algebra**: `fold` on Nat, List and each family, also a HIT. `unfold`
  into List with a Nat step limit.
- **Filterable**: `filter` on Option and List.

## 5. Core types

Each type is a library def in `domain/domain.lang`, except the two
transcript types of the examples.

| Type | Meaning (design) | Definition |
|---|---|---|
| Coin space | `Ω : FinSet` | `FinSet := Sigma (A : Type 0) (isFinSet A)`, with `isFinSet A := Sigma (n : Nat) (Trunc1 (Equiv A (Fin n)))`. `El` and `card` are projections (O3) |
| Uniform random variable | `Σ (Ω : FinSet) (Ω → T)` | `Rand T := Sigma (Om : FinSet) (El Om -> T)`, in `Type 1` |
| Equivalence | `σ : Ω ≃ Ω'` | `Equiv A B := Sigma (f : A -> B) (isEquiv f)`, with `isEquiv f` as contractible fibers |
| Coupling | a path from `(Ω, f)` to `(Ω', g)` | `Path (Rand T) (Om, f) (Om2, g)` |
| Finite multiset | the set truncation of the groupoid | `Bag T := Trunc0One (Rand T)`. `Trunc0One` is `Trunc0` with its parameter in `Type 1` (F14) |
| Decision | equal fiber sizes need decidable equality on `T` | `Dec A := Sum A (Not A)`, with `Not A := A -> Empty` and `Empty := Fin 0` |
| Schnorr transcript | `(g^r, e, z)` | `SchT := (a : Fin 23) * ((e : Fin 11) * Fin 11)` in `examples/schnorr.lang` |
| Graph isomorphism transcript | the permuted graph, the bit, the response | `GiT := (c : Fin 64) * ((b : Fin 2) * Fin 24)` in `examples/gi.lang` |

## 6. Core operations

| Operation | Type | Meaning (design) |
|---|---|---|
| `randPure`, `randMap`, `randBind` | the monad on `Rand` | `randPure` uses the coin space `Fin 1`. `randBind` uses the Sigma of the coin spaces, and the caller gives its `isFinSet` proof (O4) |
| `ua` | `Equiv A B -> Path (Type 0) A B` | Univalence through Glue. Transport along `ua e` computes to `e` |
| `uaArrow` | pointwise paths to a path over `ua e` | The CTT lemma for a path between two functions on two coin spaces |
| `isoToEquiv` | two inverse maps and their laws to `Equiv A B` | An equivalence from a bijection of the coins |
| `couple` | `T`, two coin spaces, two maps, `s`, `h` and the size path `n` to a path in `Rand T` | The bijective coupling `g ∘ σ = f` as a path. `n` is a path of the two sizes, because the kit has no pigeonhole |
| `randCount` | `T`, a decision, `Rand T` and `T` to `Nat` | The size of a fiber, a `carrierSum` of 1 or 0 |
| `countBag` | `T`, a decision, `Bag T` and `T` to `Nat` | The count on the set truncation, a fold into `Nat` with `isSetNat` |
| `countCouple` | a path in `Rand T` to a path of `randCount` | Coupling gives equal fiber sizes |
| `match` | the decision, two random variables, fallback maps, the laws and the size path to an equivalence | Equal fiber sizes give a coupling: `isoToEquiv` of two `fiberAt` maps |
| `hvzk`, `giHvzk` | `(x : Fin 1) -> Path (Rand SchT) realVar simVar` and the `GiT` form | The HVZK path of each example, a Pi def over `Fin 1` (O13) |

The two directions of the fiber theorem hold at literal instances (S6c,
`examples/bag.lang`). At unequal counts a law of `match` does not check:
`test/check/match-unequal.lang` gives `FIN_CASE`. The S7 mutants (a wrong
witness, a wrong `s` and a non-bijective `s`) also give `FIN_CASE` in each
example (`schnorr-wrong-w`, `schnorr-wrong-s`, `schnorr-nonbij`,
`gi-wrong-w`, `gi-wrong-s` and `gi-nonbij`).

## 7. Target and instance encoding

An instance is a definition of the program (not of `domain/`) whose type is
data. A function (a Pi type) and a type or a family (a universe type) are
not instances. A path is data (S1): a `PathP` def is an instance.
`langc build` writes each instance in one JSON document. Paths do not
erase: a path is never `null` (`src/json.c`). The full table is
`docs/host/README.md` `## The JSON document`.

| Type | Encoding |
|---|---|
| `Nat`, `Fin n` | A number in decimal |
| `Sigma (x : A) B` | `{"witness": w, "payload": p}` |
| `FinSet` at a literal carrier | `{"carrier": "A", "size": n, "index": [k0, ...]}` |
| `Rand T` at a literal carrier | `{"coins": n, "values": [v0, ...]}` |
| A path in `Type 0` between literal carriers | `{"from": "A", "to": "B", "transport": [t0, ...]}` (rule 1) |
| A path in a Sigma type | `{"witness": P0, "payload": P1}` (rule 2) |
| A path in a Pi type over a literal carrier | `{"paths": [P0, ...]}` (rule 3) |
| A remaining path with a body without `i` | `{"refl": a}` (rule 5) |
| A path at a HIT | The body as a tree, with `{"path": ...}` for a path constructor (rule 4) |
| Any other path | `{"from": a, "to": b, "term": t}` with the printed term (rule 6) |
| A `trunc1` or `trunc0` point | `{"trunc": a}` |

Path rules are tried in the order 1, 2, 3, 5, 4, 6. For example, `refl`
at `Path (Type 0) (Fin 2) (Fin 2)` gives the transport table `[0, 1]`.
At a Sigma type, `refl` gives the component paths of rule 2.

The build refuses a `type` text larger than 4096 bytes (`JSON_TYPE`), a
function, a type, a stuck term or a Glue element (`JSON_VALUE`), and a
value that nests deeper than 2000 levels (`JSON_DEPTH`).

The S6/S7 probes of nontrivial paths in `FinSet` and `Rand T`, and the S7b
path between `giLoop 0` and `giLoop 1`, stop with `OOM` (O13). Reflexive
paths can check: `refl` at `Path FinSet (finSetFin 1) (finSetFin 1)` checks,
and its build gives `JSON_VALUE` when the proof payload reaches a function.
Each HVZK path is a Pi def over `Fin 1` with no JSON instance. The JSON
instances of each example include
its coin `FinSet` and its two random variables (`realVar` and `simVar` in
`test/json/schnorr.json`; `giRealVar` and `giSimVar` in
`test/json/gi.json`). `test/json/gi.json` also holds the pin `loopCodes` of
the 24 loops.

## 8. Host and target

The host is the tcc-json kit of lang-template, without change. The build
uses TinyCC 0.9.28rc 2026-09-04 mob@0fb54300 (AArch64 Darwin) and Apple
clang 21.0.0 for the `-Wswitch-enum` check. The repository has no `PIN`
file: the kit is part of this tree. The limits are in `docs/host/README.md`
`## Limits`.

The facts from `probe/CAPABILITY.md` that set the design: one program file
for each run, recursion only from the eliminators, `Nat` is an unsigned
64-bit integer with an overflow trap, a family is possible in the domain
only, and the arena has a limit of 1 GiB (`src/main.c`,
`ARENA_LIMIT_BYTES`).

The output check is the gate (`make check`): it builds each example and
compares the JSON with the golden in `test/json/`. The C harness
`test/path-regressions.c` checks that a printed term checks again (82
cases).

Trust point (S6a, S6 ruling R7 (a)): the builtin `carrierSum A f` sums
`f : A -> Nat` over a literal finite carrier `A` (`Fin a` or `Sigma (e :
Fin a) (Fin b)`). The kernel does not prove that this sum is invariant
under each bijection of the carrier. The operation reads the carrier
type, not its truncated enumeration. At other carriers it stays stuck.
The count `randCount` is a guest def over it (S6c).

The builtin `fiberMatch A B T eq f g fallback x` (S6c) is not
a trust point. It gives the forward map between two literal finite
carriers, with the fibers of `f` and `g` matched in enumeration order.
It requires `fallback : A -> B` and returns `fallback x` when the target
fiber has no point at the required rank. This argument prevents a
failed search from constructing a value of `Empty`.
It reads the carrier types, as `carrierSum` does, and at other carriers
it stays stuck. The domain def `match` takes the two inverse laws, the
value law and the size path as arguments, plus fallback maps in both
directions. The check proves each law
at the instance. So a wrong map gives a refusal, not a false path.

The builtins `decEqFin` and `decEqNat` (S5b) are not trust points. At two
equal literals the value is `inl refl`. At two different literals the value
is `inr` of a refutation by a real transport, not an axiom
(`src/front/eval.c`, `dec_eq`). The printed refutation checks again (the C
case "decEqFin inr round-trips"). A neutral argument leaves the operation
stuck.

## 9. Open items

The items are in `docs/PLAN.md` section 8.

| ID | Item | Status |
|---|---|---|
| O1 | The name | RULED 2026-10-16 (USER): coin-lang |
| O2 | The kernel route | CLOSED (S1): CCHM, local to this repository |
| O3 | `FinSet` as a library Sigma | CLOSED (S5c): a library Sigma |
| O4 | `bind` on `Rand` | CLOSED (S6b): the Sigma of the coin spaces |
| O5 | The Schnorr group | CLOSED (S7a): the subgroup of order 11 mod 23, generator 2 |
| O6 | "JSON deserialization" | CLOSED (S8): the kit JSON document |
| O7 | `Trunc0` in the language | CLOSED (S4c) |
| O8 | The kit `Eq` | CLOSED (S1): `Eq` is gone |
| O9 | The graph isomorphism coins | CLOSED (S7b): `(b : Fin 2) * Fin 24` with a permutation table |
| O10 | Decidable equality on `T` for `match` | Open: narrowed in S6c; a Sigma with a dependent second component has no decision |
| O11 | The `J` rule on `refl` | CLOSED (S2a): a path, not a conversion |
| O12 | Fill `probe/CAPABILITY.md` | CLOSED (S8) |
| O13 | `OOM` in the S6/S7 nontrivial path probes and the S7b path between `giLoop 0` and `giLoop 1` | Open: located in S7 (the interval normal form under the JSON path rules) |

## 10. Slices

Each slice is DONE. `docs/PLAN.md` section 7 gives the content and
`docs/STATUS.md` gives the tests.

| Slice | Commit | Content |
|---|---|---|
| S0 | `9f5e0a3` | The repository from lang-template, the design and the plan |
| S1 | `97e66e0` | The interval and paths; the kit `Eq` removed |
| S2a | `a4bb138` | `transp` and the `J` library |
| S2b | `582ecef` | `hcomp`, systems and path composition |
| S3a | `44c917f` | `Fin`, `finAll` and `isoToEquiv` |
| S3b | `91d7091` | Glue, `ua` and transport tables |
| S3c | `34acff5` | `hcomp` at Glue and in the universe; `uaArrow` |
| S4a | `8c79e9a` | Path constructors, `S1` and formal `hcomp` |
| S4b | `e0d4f3c` | `Trunc1` and transport at a family |
| S4c | `25c14e0` | `Trunc0` and set quotients |
| S5a | `a80e17e` | `finCase` and the `Fin` ring; `Flag` removed |
| S5b | `be42cc4` | Decidable equality and Hedberg |
| S5c | `a08577d` | `FinSet` and the product FinSets |
| S6a | `14ebb60` | `Rand`, `carrierSum` and the `Rand` JSON |
| S6b | `9189977` | `couple`, `randBind` and the finite path JSON |
| S6c | `5877e1d` | Pair decisions, `randCount`, `Bag` and `fiberMatch` |
| S7a | `7bd9ce1` | The Schnorr HVZK example and its mutants |
| S7b | `31d73f8` | The graph isomorphism HVZK example and its mutants |
| S8 | this commit | This specification, the probe, the status, the validation and the README |
