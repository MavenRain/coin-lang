# TinyCC JSON host kit

This kit compiles a small typed language to JSON. A C front end (lexer,
parser, checker and evaluator) and a JSON writer build with TinyCC into
one executable, `build/langc`. The output is the final state of each
instance in the program. There is no IR and no lowering step.

From this kit directory, or from a generated language directory:

```sh
make check
build/langc check examples/inventory.lang
build/langc eval examples/inventory.lang stockTotal
build/langc build examples/inventory.lang -o inventory.json
```

The build uses TinyCC (`TCC`, default `tcc`). Validation uses TinyCC
0.9.28rc on AArch64 Darwin. `make check-clang` also compiles the sources
with `cc -Wall -Wextra -Wswitch-enum -Werror -fsyntax-only` (`CC`, default
`cc`). The gate uses POSIX `sh`, `awk`, `cmp` and Node (for `JSON.parse`).
No network access or package installation is needed.

## Commands

| Command | Result |
|---|---|
| `langc check PROG` | Checks the program. Prints `ok`. |
| `langc eval PROG NAME [ARGS...]` | Applies the definition `NAME` to the literal arguments and prints the normal form. A Nat overflow prints `trap`. |
| `langc build PROG [-o OUT]` | Checks the program, evaluates each instance and writes one JSON document to stdout, or to `OUT` |

Exit 0 is success. Exit 1 is a refused program. Exit 2 is a usage or IO
error. A refusal writes one line to stderr: `langc: CODE: NAME: message`.
`langc build` makes the full document in memory before it writes. Thus a
refused build writes nothing to stdout and makes no `OUT` file.

## The JSON document

```json
{"coin-lang":1,"instances":[{"name":"stockTotal","type":"Nat","value":70}]}
```

The key is the language name and the value is the format version, 1.
The generator reserves the name `instances` for this host to keep the two
document keys distinct.
`instances` holds one object for each instance, in source order. An
instance is a definition of the program (not of `domain/`) whose type is
data. These definitions are checked, but they are not instances:

- functions (a Pi type, dependent or not);
- types and families (a universe type);
- the definitions in `domain/domain.lang`.

`type` is the normal form of the definition type in source syntax, for
example `Sigma (n : Nat) (PathP (fun _ => Nat) n 1200)`. Binder names are freshened when
needed to preserve scope. The complete text must fit a 4096-byte buffer;
the build refuses a type that exceeds this size or the printer depth of
200, rather than emitting an abbreviated type.
`value` is the evaluated value. Values carry no types, so the writer walks
the type and the value together (`src/json.c`). The fuel of the evaluator
starts again for each instance.

| Type | JSON value |
|---|---|
| `Nat` | A number: the full unsigned 64-bit value in decimal |
| `Fin n` | A number below `n`, in decimal. `Fin 2` is the result of `natLe`, `natEq` and `finEq` (S5a) |
| `Unit` | `{}` |
| `Prod A B` | `{"first": a, "second": b}` |
| `Sum A B` | `{"inl": a}` or `{"inr": b}` |
| `Option A` | `null` or the value. When `A` is an `Option` type, `some v` is `{"some": v}`. |
| `List A` | An array |
| `Sigma (x : A) B` | `{"witness": w, "payload": p}`. The payload type is `B` at the witness. |
| `FinSet` (`Sigma (A : Type 0) (isFinSet A)`), a `Fin a` or `Sigma (e : Fin a) (Fin b)` carrier and a `trunc1` point | `{"carrier": "A", "size": n, "index": [k0, ...]}`: the forward map of the equivalence at each point of the carrier, in enumeration order (`0 .. a-1`; `pack 0 0`, `pack 0 1`, .., `pack (a-1) (b-1)`). In `examples/finset.lang`, `fin3` gives `{"carrier": "Fin 3", "size": 3, "index": [0, 1, 2]}` and `prodFinSet11` the 121 numbers in order. An entry that does not compute to a number is refused (`JSON_VALUE`). Any other carrier is a type and is refused (`JSON_VALUE`). |
| `Rand T` (`Sigma (Om : FinSet) (El Om -> T)`), a `Fin a` or `Sigma (e : Fin a) (Fin b)` carrier | `{"coins": n, "values": [v0, ...]}` (S6a): `n` is the number of points of the carrier, and `values` holds the JSON at `T` of the function at each point, in the enumeration order of the `FinSet` row. In `examples/rand.lang`, `r3` gives `{"coins": 3, "values": [1, 2, 0]}` and `r2x3` (over `(e : Fin 2) * Fin 3`) gives `{"coins": 6, "values": [0, 1, 2, 10, 11, 12]}`. At any other carrier the `Sigma` row applies, and the build refuses the function (`JSON_VALUE`). |
| `PathP (fun i => A) a b`, a body without `i` | `{"refl": a}` (rule 5, after rules 1, 2 and 3). A point of a Pi type or a universe is its printed term, as a string. `Path A a b` is `PathP (fun _ => A) a b`. |
| `PathP (fun _ => Type 0) A B`, with `A` = `Fin n`, or `Sigma (e : Fin a) (Fin b)` (S6b), and literals | `{"from": "A", "to": "B", "transport": [t0, ...]}` (rule 1, before rules 5 and 6). `tk` is the value in `B` of `transp (fun i => p @ i) i0 k`, for each point `k` of `A`, in the enumeration order of the `FinSet` row. For a `ua` line, for example `ua (Fin 3) (Fin 3) e`, the table is `witness e` at each `k`. For a line made by `hcomp` in `Type 0`, the table is the composite transport: `transTwoLoops` in `examples/univ.lang`, two `loopRot1` lines, gives `[2, 0, 1]`. When a table value does not compute to a number or a canonical value, rules 5 and 6 apply. |
| `PathP (fun i => Sigma (x : A i) (B i)) a b`, a path abstraction | `{"witness": P0, "payload": P1}` (rule 2, S6b, after rule 1, before rule 5): `P0` is the path of the first components and `P1` is the path of the second components over `P0`, each by these rules. A constant Sigma path prints by rule 2 too: `Path ((e : Fin 2) * Fin 3) (pack 1 2) (pack 1 2)` gives `{"witness": {"refl": 1}, "payload": {"refl": 2}}`. A path that is not a path abstraction goes to rules 5, 4 and 6. |
| `PathP (fun i => (x : A i) -> B i x) f g`, a path abstraction, with `A i0` a carrier of rule 1 | `{"paths": [P0, ...]}` (rule 3, S6b, after rule 2, before rule 5): one path for each point `x` of `A i0`, in the order of rule 1. `Pk` goes from `f x` to `g` at the transport of `x` along the domain line, by these rules. In `test/json/fin.json`, `law2` gives `{"paths": [{"refl": 0}, {"refl": 1}]}`. In `examples/couple.lang`, `coupleShape` (a path in `(A : Type 0) * (A -> Fin 3)`) gives rule 2, with the rule 1 table `[1, 2, 0]` as the witness and a rule 3 table of 3 rule 6 terms as the payload. |
| `Glue A phi sys`, with no true key | Refused (`JSON_VALUE`): a Glue element has no JSON form. A Glue type with a true key is its branch type `T`. |
| `PathP (fun i => A) a b`, a body with `i` | `{"from": a, "to": b, "term": t}` (rule 6). `t` is the printed path term, as a string. A `transp` that does not compute stays in `t` as `transp (fun j => A) phi a`. An `hcomp` that does not compute stays in `t` as `hcomp A phi (fun i => [ k -> u, ... ]) u0`. |
| `PathP (fun _ => F) a b`, `F` a family with a path constructor, a body with `i` | Rule 4 (after rules 1 and 5, before rule 6): the body at `i` as a tree. A path constructor whose last interval argument is `i` gives `{"path": "name", "args": [t, ...]}`, with a tree for each point field. A formal `hcomp` gives `{"hcomp": {"phi": "face", "base": t}}`, with the face printed and the base as a tree. When another part of the body mentions `i` (a point constructor, a field that is not of the family, or a constructor with 2 interval fields), rule 6 applies. In `examples/circle.lang`, `loopPath` gives `{"path": "loop", "args": []}`. |
| A family whose constructors have no fields | The constructor name, as a string |
| A family with one constructor | An object of the fields |
| Any other family | `{"tag": "name", ...}` with the fields of that constructor |
| A formal `hcomp` at a family with a path constructor | `{"hcomp": {"phi": "face", "base": t}}` |
| A `trunc1` or `trunc0` point | `{"trunc": a}`, with `a` the JSON of the field. In `examples/trunc.lang`, `sq1` gives `{"path": "squash1", "args": [{"trunc": 0}, {"trunc": 1}]}`. |

Rules 1 and 3 recognize literal carriers before enumerating them. Empty
carriers give `"transport": []` or `"paths": []`. Every fiber of a Sigma
carrier must be a literal `Fin`; otherwise the later path rules apply
to the whole path, rather than printing a partial table.

An indexed family uses the same rules. Its index is in `type` (for
example `Lot 3`), not in `value`.

`langc build` refuses a program with these codes:

| Code | Cause |
|---|---|
| `JSON_VALUE` | An instance holds a function, a type or a stuck term |
| `JSON_DEPTH` | An instance nests deeper than 2000 levels. A list spine does not count. |
| `JSON_TYPE` | An instance type, path or point exceeds the printer limits or has no supported source representation |
| `EVAL_OVERFLOW` | A Nat operation in an instance overflows |

The check and evaluation codes are the same as in `langc check` and
`langc eval`. `test/parse`, `test/check` and `test/emit` hold a program
for each refusal.

`langc eval` also refuses an incomplete normal form with `EVAL_PRINT`
(a 65536-byte buffer and the same printer depth limit). A stuck finite case
prints as `finCase n k c0 .. c(n-1)` (S5a, R12), so `eval` prints it and
`build` emits it (`test/json/fin.json`, the type of `law`). A closed case that
computes to a literal prints as the literal. In an argument position, a
typed identity preserves the case result type so printed path and function
cases can be checked again. The 64-argument application limit does not
apply to `finCase`, whose arguments include its entire case table.

`carrierSum A f` (S6a), with `A : Type 0` and `f : A -> Nat`, is the sum
of `f` over the points of `A`. It reduces only at a literal `Fin a` or
`Sigma (e : Fin a) (Fin b)` carrier, in the enumeration order of the
`FinSet` row. At other carriers it stays stuck. The kernel does not prove
that the sum is invariant under a bijection of the carrier: this is a
trust point (`SPEC.md` section 8). A Nat or Fin operation on a glue
element stays stuck (S6a), so a transport of a point along a `ua` line of
`Fin n` prints its term (rule 6).

`fiberMatch A B T eq f g fallback x` (S6c), with `eq : T -> T -> Fin 2`
(1 for equal values), `f : A -> T`, `g : B -> T`, `fallback : A -> B`
and `x : A`, is a point of
`B`. Let `k` be the number of points of `A` before `x` whose value is
`f x`. The result is the point `k` of `B` (counted from 0) whose value
is `f x`, in the enumeration order of the `FinSet` row. It reduces only
when `A` and `B` are literal carriers of the `carrierSum` kind. It stays
stuck when a carrier is not literal or when `x` is not a point of `A`.
When `B` has no such point, it returns `fallback x`. The fallback is
required to keep a failed search from constructing a value of `Empty`.
`fiberAt` takes the same fallback map, and `match` takes forward and
backward fallback maps after `f` and `g`, before its laws. It is not a trust
point: the domain def `match` takes the inverse laws, and the check
proves them (`SPEC.md` section 8). In `examples/bag.lang`, `matchAt`
gives `pack 0 2`.

## Glue

| Builtin | Form | Rules |
|---|---|---|
| `Glue` | `Glue A phi (fun (_ : I) => [ k0 -> pack T0 e0, ... ])`, with `A : Type 0` and each branch of type `Sigma (T : Type 0) (Equiv T A)` where its key holds | A type in `Type 0`. A true key gives its branch type `T`. |
| `glue` | `glue (fun (_ : I) => [ k0 -> t0, ... ]) a`, in check mode against a Glue type | A true key gives its branch `t`. Else `glue` is canonical. |
| `unglue` | `unglue g`, with `g` of a Glue type `Glue A phi sys` | A term of `A`. `unglue (glue sys a)` is `a`. A true key gives `witness e g`. Else `unglue` is stuck. |
| `ua` | `ua A B e : Path (Type 0) A B`, a def in `domain/domain.lang` | `fun (i : I) => Glue B (~i \/ i) (fun (_ : I) => [ ~i -> pack A e, i -> pack B (idEquiv B) ])`. Transport along `ua A B e` at a closed value `a` gives `witness e a` (uaβ). |
| `hcomp` at Glue | `hcomp (Glue A phi sys) psi (fun (j : I) => [ k -> u, ... ]) u0` | A `glue` with the branch `hcomp T psi u u0` for each key of `phi`. The base is an `hcomp` in `A` of `unglue` of the tube and `unglue u0`; at each key of `phi`, its system adds `witness e` of the filler in `T`. |
| `hcomp` in `Type 0` | `hcomp (Type 0) psi (fun (j : I) => [ k -> A, ... ]) A0` | `Glue A0 psi (fun (_ : I) => [ k -> pack (A at i1) e, ... ])`. `e` is the transport of `idEquiv` along the reversed line of `A` (`lineToEquiv`). A key that is `i0` gives no branch. Only this direct form reduces: where a Pi, Sigma, PathP or data rule reaches `Type 0`, the outer operation stays stuck. The base must be the branch at `i0` (`HCOMP_CAP`). |
| `uaArrow` | `uaArrow A B C e f0 f1 h : PathP (fun (i : I) => (ua A B e @ i) -> C) f0 f1`, with `h : (a : A) -> Path C (f0 a) (f1 (witness e a))`, a def in `domain/domain.lang` | `fun i a => hcomp C (~i \/ i) (fun (j : I) => [ ~i -> h a @ ~j, i -> f1 a ]) (f1 (unglue a))`, the agda/cubical `ua→` with a constant `C`. At `i0` it gives `f0 a`, and at `i1` it gives `f1 a`. |

`hcomp` at `Type 1` or higher is refused with `HCOMP_UNIVERSE`: the Glue
type and equivalence library are restricted to `Type 0`.

`transp` along a Glue line gives a `glue` at `i1`. Each branch comes from
the fiber extension of its equivalence, and an `hcomp` in `A` at `i1`
gives the base. `examples/univ.lang` has an `hcomp` in `Type 0`
(`transTwoLoops`), an `hcomp` at Glue (`hcompGlue`) and a `uaArrow`
instance (`uaArrowPlus3`). `langc check` gives these codes for Glue:

| Code | Cause |
|---|---|
| `GLUE_BINDER` | The tube binder of `Glue` or `glue` is not `_` |
| `GLUE_EQUIV` | A branch of `Glue` is not `pack T e` with `T : Type 0` and `e : Equiv T A` |
| `GLUE_BOUNDARY` | Where a key holds, `unglue` of the branch of `glue` is not the base `a` |
| `UNGLUE_TYPE` | The argument of `unglue` does not have a Glue type |
| `SYSTEM_COVER` | The keys of the system of `Glue` or `glue` do not cover `phi` |

## Higher inductive types

A domain family can have path constructors. A path constructor takes its
point fields, then 1 or 2 interval fields `(i : I)`, then a boundary
system `[ phi -> t, ... ]` that covers the faces of its interval fields
(`~i \/ i` for 1 field). Each branch must have the family type at the
parameters, and the branches must agree where their keys overlap. A
program cannot declare a family (`REFUSE_DATA`), so only the domain
declares a path constructor. `domain/domain.lang` declares the circle:
`family S1 := base | loop (i : I) [ ~i -> base, i -> base ]`.

| Term | Form | Rules |
|---|---|---|
| Path constructor | `loop r`, with `r : I` | At a true key of the boundary, the branch term: `loop i0` and `loop i1` are `base`. Else `loop r` is canonical. A path constructor value does not convert with its boundary point, so `loop` is not `refl` (`PATH_UIP`). |
| `fold` with path cases | `fold c1 ... cn x : RES`, one case for each constructor, in order | The case of a path constructor of dimension 1 has the type `(fields) -> PathP (fun (_ : I) => RES) B0 B1`. `B0` and `B1` are the fold images of the boundary at `i0` and `i1`: a field gives its folded value, a point constructor gives its case at the images of its fields. `fold (loop r)` is `case @ r`. |
| `hcomp` at a family with a path constructor | `hcomp F phi (fun (j : I) => [ k -> u, ... ]) u0` | A true key gives the branch at `i1`. Else the `hcomp` is canonical (a formal `hcomp`), also at `i0` and with a neutral base. `fold` of a formal `hcomp` is `hcomp RES phi (fun (j : I) => [ k -> fold u, ... ]) (fold u0)`. |
| `transp` at a family with no parameters | `transp (fun (_ : I) => S1) phi v` | `v`: the line is constant by the form of the family (this is not regularity). |
| `transp` at a family with parameters | `transp (fun (i : I) => F (p i)) phi v` | At a constructor `v`: the same constructor at the parameters of the line at `i1`. Each point field goes along its field type at the parameters of the line, and a recursive field goes along the line. The interval fields do not change. At a formal `hcomp`: `hcomp` at the line at `i1`, with each tube and the base transported. A neutral `v` stays stuck. |
| `fold` at a constructor with two interval fields | `fold c0 .. cn x` | The case of `squash0` or `quotSquash` has the type `isSet RES`: a square over the folded fields. A path field `p` goes to its case as `fun (k : I) => fold (p @ k)`. A fold with no such case gives `TYPE_MISMATCH`. |

Constructor transport reduces a path constructor only when its boundary
faces use its interval fields and each branch is a recursive field, as
with `squash1`. Other boundary terms retain an opaque `transp`: rebuilding
the constructor alone could change its endpoints when a parameter changes.
General boundaries require a boundary correction before they can reduce.

`squash0` and `quotSquash` have two interval fields, and their branches
are path applications (`squash0Left @ j`), not recursive fields. Thus
their transport along a parameter line retains an opaque `transp` (R7
(a)).

`examples/circle.lang` has `loopPath`, the transport along `code2 (loop
i)` (a fold of `S1` into `Type 0` with a `ua` path case), which moves
`0` to `1`, and the formal `hcomp` `loop2`, whose transport along
`code3` moves `0` to `2`. `langc check` gives these codes for path
constructors:

| Code | Cause |
|---|---|
| `HIT_BOUNDARY` | A path constructor has no boundary, more than 2 interval fields, or a point field after an interval field; a constructor with no interval field has a boundary; a fold meets a boundary term that is not a field, a path field at an interval or an earlier point constructor, or an unsupported field over an earlier recursive field |
| `SYSTEM_COVER` | The boundary keys do not cover the faces of the interval fields |
| `SYSTEM_OVERLAP` | Two boundary branches do not agree where both keys hold |
| `TYPE_MISMATCH` | A boundary branch does not have the family type, or a fold has no case for a path constructor |
| `PATH_END` | The case of a path constructor does not have the fold images of the boundary as its ends |

## The domain

`domain/domain.lang` is the coin-lang library and the kit sample
families `Color`, `Item`, `Stack` and `Lot`. `gen/embed.c` embeds it in
the executable at build time. `domain/README.md` tells how to replace it.
It holds the equivalence library: `isContr`, `fiber`, `isEquiv`, `Equiv`,
`idEquiv` and `isoToEquiv`, the univalence defs `ua` and `uaArrow`
(see Glue), and the circle `S1` (see Higher inductive types). S5b adds
the decision defs `Empty` (`Fin 0`), `absurd`, `Not`, `Dec`, `Stable`,
`decStable` and `isPropNot`, the Hedberg defs `hedbergStable`,
`hedbergMap`, `hedbergConst`, `hedbergSquare` and `hedberg`, and
`isSetNat` and `isSetFin`. S5c adds the coin spaces: `isFinSet`,
`FinSet` (in `Type 1`), `El`, `card` and `finSetFin`. The domain also
holds `Rand`, `randPure`, `randMap`, `randBind` and `couple` (S6).
`randBind` takes the `isFinSet` proof of its Sigma coin space as an
argument (R3), and `couple s h n` takes the size path `n` (R5): the kit
has no pigeonhole. S6c adds `decEqProd` and `decEqSigmaFin` (R11), the
family `Trunc0One` (`Trunc0` with its parameter in `Type 1`, R8), and the
defs `randCount`, `Bag` (`Trunc0One (Rand T)`), `countBag`,
`countCouple`, `fiberEq`, `fiberAt` and `match` (R9, R10). The count def
is `randCount`, not `count`, because `examples/formers.lang` declares a
def `count`. A program
def cannot have the name of a domain constructor, for example `loop`
(`REFUSE_NAME`). An `Equiv` value holds a function, so a
program def of an `Equiv` type is refused by `langc build`
(`JSON_VALUE`). Write the equivalence inline in a data def, or under a Pi
type. A program def whose type holds a Glue type, for example a path
over `loopRot1 @ j` or over `ua A B e @ i`, is refused by `langc build`
too (`JSON_TYPE`: the type has no source syntax). Write it under a Pi
type: a `Fin n` binder lets the checker's finAll fallback check each closed
case. The builtins `finAdd`, `finNeg`, `finMul`, `finVal`, `succ`,
`decEqFin` and `decEqNat` are not domain defs: the checker gives them
(`BUILTINS` in `src/front/check.c`).

## Limits

| Limit | Value | Source |
|---|---|---|
| Source size | 1 MiB | `src/front/front.h:7` |
| Arena | 1 GiB | `src/main.c:10` |
| Parser nesting | 1000 | `src/front/parser.c:16` |
| Checker depth | 2000 | `src/front/check.c:10` |
| Evaluator depth | 2000 | `src/front/core.h:16` |
| Evaluator fuel, for each instance | 20,000,000 steps | `src/front/core.h:17` |
| JSON nesting | 2000 | `src/json.h:15` |
| `type` text | 4096 bytes | `src/json.c:7` |
| Type and normal-form printer depth | 200 | `src/front/eval.c` |
| finAll cases (`Fin n`, a literal `n`; a `finCase n` takes its `n` cases as arguments) | `n` below 2^32 - 1; the arena stops a large `n` first | `src/front/check.c` (`fin_cases`) |

## Origin

This kit is a fork of the tcc-wasm kit front end. It does not have the
Wasm writer (`wasm.c`), the IR (`ir.h`) or the target interface
(`target.h`). It adds `src/json.c` and `src/json.h`, and `src/main.c` has
the verbs `check`, `eval` and `build`. The front end also fixes dependent
result inference, Product and Sigma eta, Sum bind error-type preservation,
domain name checking, the F14 universe rules, and complete,
scope-preserving value printing. The snapshot below records the original
fork, before these fixes. To find the changes after the fork,
compare the hashes with `shasum -a 256 src/front/*` in the tcc-wasm kit.

SHA-256 of the tcc-wasm kit sources at the fork (paths relative to that
kit):

| Path | SHA-256 |
|---|---|
| `src/front/ast.h` | `5dbe74e54c8207da17a57810624bf049d729f272c73a0765b9fd504fb3f18720` |
| `src/front/base.c` | `d994f71d11a352896212d97ed6edb5de062dc37ad6f31bf927487c4d8e675d5b` |
| `src/front/base.h` | `d36daae15e9a79ae298eadf5f1f9c3687e077c7305e63fad6cc5b477b58cebca` |
| `src/front/check.c` | `526a8207c66f3276c18767c62de05026866e7e8adc30e2c5383a2975c0a57114` |
| `src/front/check.h` | `5d781eb3f52a7a17db10d375c0fb7c5b578bf0527dd358cfe49ebb779d008d06` |
| `src/front/core.h` | `d04aa9412bb0f25d470bd3b91e3a073284b19e89aa90083e18d63a6d468fd9dc` |
| `src/front/eval.c` | `5a044b8935f77614a2321883afc8df7d2b1d745e198b8f861538dc1a61773c55` |
| `src/front/front.c` | `9b7d8890d0854fe989780431521189fabf0a79e624359733f5a90822bbf08c27` |
| `src/front/front.h` | `7eb0b8de605ad7336eac1bffc9b65587d7113b2e3849a95a9e6adf75fd12989a` |
| `src/front/lexer.c` | `6812190d933fa5dfb1ad60c9d40a46aeb460aab5342feed6c4697fe31a2425bc` |
| `src/front/lexer.h` | `85046ffc1f76878b8dc0102db08458a59fef914d7951462c9e7bf590cb5db314` |
| `src/front/parser.c` | `6220dcc67e0fc4782d60cf9516bae979d2d4eb3c5eb3e047d92b85806b0bb02d` |
| `src/front/parser.h` | `e9caa8e0dbcfc716b572fa754daa3023ee1a55fb16ff77fb0235c97a49adaa1c` |
| `src/ir.h` | `13021f895667552ec1d527267258e8a3437cadde74744a348e1219dff6d773b9` |
| `src/main.c` | `2294beb677fb4c208811937bbf48f1570911de6e52a6068874e90b3489b211ba` |
| `src/target.h` | `077dff69391f69dbd4623858553dee7b5547a8c1edc0c75660e0aede9b15e429` |
| `src/wasm.c` | `4f69fba5ea1f335e95c7f030d970c4e89da84ee13c57d95879137cf96d8235bd` |

`FORMERS.md`, copied to `formers/tcc-json.md` in a generated language,
gives the status of each former. `docs/CAPABILITY.md` gives the host
facts.
