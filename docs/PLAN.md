# coin-lang plan

## 1. Rulings

The USER gave these rulings in the request (`design/DESIGN.md`):

- The type constructors are products, coproducts, monads (map and bind),
  algebras and coalgebras (fold and unfold), and paths, loops and quotients
  as in cubical type theory (CTT).
- The base types and the base operations are the minimum span that models
  perfect honest-verifier zero knowledge (HVZK) as an equivalence of coin
  spaces (sections 2 and 5).
- Proof irrelevance is prohibited, so that loops and quotients can be
  non-trivial (section 3).
- Host: TinyCC. Output: the JSON document of the final state of each
  instance in the program.

These are the builder's calls. The USER did not rule them. Section 8 lists
them as open items:

- Name: coin-lang (O1). The compiler binary is `build/langc`.
- Host kit: `hosts/tcc-json` of lang-template, without change. The cubical
  kernel (section 4) goes into this repository, not into lang-template (O2).
- The design says "JSON deserialization". This plan reads it as the JSON
  document that the kit writes and that a consumer deserializes (O6).
- Tests: unit and integration tests are in the guest language. A host
  runner that loads and runs guest test files is permitted. Do not add a
  node, Python or shell harness. The kit JS regressions stay.

## 2. The design on the kit basis

| Concept | Design text | Encoding (proposal) | Slice |
|---|---|---|---|
| Coin space | `Ω : FinSet` | `FinSet := Sigma (A : U) (isFinSet A)` and `isFinSet A := Trunc1 (Sigma (n : Nat) (Equiv A (Fin n)))`. A library def, not a kernel primitive (O3). `El : FinSet -> U` is the first projection. | S5 |
| Uniform random variable | `Σ (Ω : FinSet) (Ω → T)` | `Rand T := Sigma (Om : FinSet) (El Om -> T)`. The distribution of `(Om, f)` is the count of each fiber of `f`, divided by the size of `Om`. | S6 |
| Monad | map and bind | `pure x := (Fin 1, \_ -> x)`. `map` composes. `bind` uses the Sigma of the coin spaces (O4). The monad laws are paths made from `ua` of the Sigma unit and associativity equivalences. They are not conversions. | S6 |
| Equivalence | `σ : Ω ≃ Ω'` | `Equiv A B := Sigma (f : A -> B) (isEquiv f)`, with `isEquiv f` as contractible fibers (the CTT library form). `isoToEquiv` makes an equivalence from two inverse maps. | S3 |
| Univalence | a path from `(Ω, f)` to `(Ω', g)` | `ua : Equiv A B -> Path U A B` through Glue types. Transport along `ua e` computes to `e` by definition (uaβ). A path in `FinSet` is a path in `U` and a path over it in `isFinSet`, which is a proposition. | S3, S5 |
| Bijective coupling | `g ∘ σ = f` | `couple s h : Path (Rand T) (Om, f) (Om', g)` for `s : Equiv (El Om) (El Om')` and `h : (w : El Om) -> Path T (f w) (g (s w))`. A library def from `ua`, `uaArrow` (the CTT lemma that turns pointwise paths into a path over `ua`) and Sigma paths. | S6 |
| Equal fiber sizes | equal distributions exactly when `σ` exists | Both directions are guest functions. Coupling to equal counts: `count` is invariant under transport. Equal counts to coupling: `match` makes `s` from the fibers in enumeration order. `match` needs decidable equality on `T` (O10). | S6 |
| Schnorr HVZK | `z = r + e·w`, `r ↦ r + e·w` on `Z_q` | Group: the subgroup of order 11 of the units mod 23, generator 2 (O5). Transcript: commitment, challenge, response. Coins: `(e, r)` (real) and `(e, z)` (simulated), both `Fin 11 * Fin 11`. `s (e, r) := (e, r + e*w)`. Real: `(g^r, e, r + e*w)`. Simulated: `(g^z * y^(-e), e, z)` with `y := g^w`. Each pointwise path is `refl` after enumeration (`finAll`, section 5). | S7 |
| Graph isomorphism HVZK | | Graph: `Fin n -> Fin n -> Fin 2`. Coins: a challenge bit and a permutation. The example has n = 4 (24 permutations, 48 coins). The coin space is `Fin 2 * Fin 24`, and a table maps each index to its permutation (O9). Each permutation `p` also gives the loop `ua p` in `FinSet` at `Fin 4`. The 24 loops are distinct in the JSON output. | S7 |
| Finite multisets | the set truncation of the groupoid | `Bag T := Trunc0 (Rand T)`. `Rand T` is a groupoid when `T` is a set. `count : Bag T -> T -> Nat` is an elimination into `Nat`, which is a set. | S6 |

## 3. The no-irrelevance rule

A proof is data. A path is data. The kernel and the JSON writer must keep
them. These are prohibited:

- a universe with definitional proof irrelevance (`Prop` or `SProp`);
- UIP or axiom K, as an axiom or as a conversion rule;
- a conversion rule that makes two paths equal because they have the same
  type or the same end points;
- a groupoid law as a conversion (for example `p . sym p` to `refl`). A
  groupoid law is a 2-path;
- a JSON writer that erases a path or a proof.

These are permitted, because they do not identify proofs by their type:

- the CTT computation rules: the end point rules of `PathP`, the rules of
  `transp`, `hcomp`, Glue and the HIT eliminators, and uaβ;
- `Trunc1` and `Trunc0` as HITs. Their truncation constructors are path
  constructors: they add paths and they identify no terms by conversion.
  The design asks for the set truncation of the groupoid of coin spaces, so
  `Trunc0` is in the language (O7);
- `isProp A` and `isSet A` as ordinary types (Pi types of paths);
- the conversions of ruling R11: path eta, and the normal form of the
  interval (`~~r = r` and the De Morgan laws). Path eta compares two paths
  at a fresh interval variable, not by their type or their end points.

The reason in the design: the loops of `FinSet` at `Fin n` are the n!
permutations of `Fin n`. With proof irrelevance each loop is `refl`, and
the permutation coins of graph isomorphism become one coin.

Gaps in the kit at the start (the same as the strategy-lang S0 survey).
S1 closed both:

- `src/json.c` wrote each `Eq` value as `null`, and a def of an `Eq` type
  was not an instance. Closed in S1: `Eq` is gone, a def of a `PathP` type
  is an instance, and its value is `{"refl": a}` or `{"from": a, "to": b,
  "term": t}` (`docs/host/README.md`). strategy-lang b40e45d was the
  reference.
- The kit `Eq` was the book identity type with `refl`, `symm`, `trans`,
  `cong` and `transport`. Closed in S1: `PathP` replaces it (O8). `refl`,
  `sym`, `cong` and `funExt` are builtins. `trans` and `transport` come
  back in S2, with `hcomp` and `transp`. S2a: `transp` is a builtin with
  rules for the universe, the data formers, Pi and Sigma. `transport`, `subst`, `J` and `JRefl`
  are library defs in `domain/domain.lang`. S2b: `hcomp A phi (fun (i :
  I) => [ k -> t, ... ]) u0` is a builtin with rules for a true face, Pi,
  Sigma, PathP and the data formers, and `transp` has the PathP rule.
  `trans` is a library def by `hcomp`. S2 is done.

## 4. The cubical kernel

The route is the CCHM cubical type theory as in Cubical Agda: a De Morgan
interval, separate `transp` and `hcomp`, general Glue types, and HITs with
a formal `hcomp` (O2).

- Interval: `I` with `i0`, `i1`, `~r`, `r /\ s` and `r \/ s`. `I` is not a
  type in `U`. A face formula is a disjunction of conjunctions of
  `(i = 0)` and `(i = 1)`.
- Paths: `PathP (\i -> A) a b`, the path abstraction `\i -> t` and the
  application `p @ r`. The end point rules hold by definition: `p @ i0` is
  `a` and `p @ i1` is `b`. `Path A a b` is `PathP (\_ -> A) a b`. Function
  extensionality is a term, not an axiom.
- Kan operations: `transp (\i -> A) phi a` and `hcomp {A} [phi -> u] a` for
  each former: Pi, Sigma, PathP, the universe, Glue, `Nat`, `Fin n`, Unit,
  Sum, Option, List, the kit families and the HITs. `transport`, `subst`,
  `J`, path composition, `sym` and `cong` are library defs. The `J` rule on
  `refl` is a path, not a conversion (O11).
- Glue: `Glue A [phi -> (T, e)]`, `glue` and `unglue`. `hcomp` in `U` gives
  a Glue type. `ua e := \i -> Glue B [(i = 0) -> (A, e), (i = 1) -> (B, idEquiv B)]`.
- HITs: a declaration gives point constructors and path constructors of
  dimension 1 or 2, each with its boundary. The eliminator has one case for
  each constructor. The checker checks the boundary of each path case.
  `hcomp` is a constructor of each HIT. `transp` in a HIT with parameters
  goes into the constructors.
- The HITs of the prelude: `Trunc1 A` (`|a|`, `squash1`), `Trunc0 A`
  (`|a|`, `squash0` of dimension 2), the set quotient `A / R`, and the
  circle `S1` (`base`, `loop`). A test shows that `loop` is not `refl`: a
  map `S1 -> U` sends `loop` to `ua` of the swap of `Fin 2`, and transport
  of `0` along the image of `loop` gives `1`.

## 5. The base span

These are the only base types and base operations:

| Base | Operations | Why the design needs it |
|---|---|---|
| `Nat` | literals, `succ`, `natAdd`, `natMul`, decidable equality with evidence | the size of a coin space, fiber counts, the index of `Fin` |
| `Fin n` | literals `k` for `k < n`, `finAdd`, `finNeg`, `finMul` (the ring `Z_n`), `finVal : Fin n -> Nat`, decidable equality with evidence, `finAll` | `Z_q`, `Z_p`, the challenge bit, the vertices of a graph, the base of `FinSet` |
| `U` | universes (`FinSet` is in `U1`) | `FinSet` is a Sigma over `U` |

- `finAll` is the eliminator of `Fin n` for a literal `n`. The checker
  checks `\(k : Fin n) -> t` against `(k : Fin n) -> P k` when
  `t[j/k] : P j` for each `j < n`. It is the case analysis of `Fin n` with
  n cases. The pointwise paths of a coupling use it: after evaluation of
  the closed modular arithmetic, each case is `refl`.
- The Schnorr group operations are defs on `Fin 23`: `pow b r` is a fold
  over `finVal r` with `finMul`.
- Decidable equality gives evidence both ways:
  `Dec (Path A a b) := Sum (Path A a b) (Path A a b -> Empty)`, with
  `Empty := Fin 0`.
- These kit parts are outside the span. S5 removes them: `Flag` (use
  `Fin 2`), the kit `Eq` (S1 replaces it) and the Nat operations that the
  design does not use. Unit, Sum, Option and List stay, because they are
  formers (products, coproducts and monads), not base types.

## 6. JSON

The kit document stays: `{"coin-lang":1,"instances":[...]}`. These rules
are new or change:

- Each def whose type is a `PathP` type is an instance.
- A function whose domain is `Fin n`, or a Sigma of `Fin` types, is a
  table: a JSON array of its values in enumeration order.
- A type is its printed normal form, as a string (strategy-lang S1 ruling
  (b)).
- A `Rand T` value is `{"coins": n, "values": [...]}`.
- A path is never `null`. The writer uses the first rule that applies:
  1. a path in a universe with a finite end point:
     `{"from": A, "to": B, "transport": table}`. For `ua s` the table is `s`;
  2. a path in a Sigma type: one object for each component path;
  3. a path in a function type with a finite domain: a table of paths;
  4. a path in a HIT: its constructor tree (`{"path": "loop", "args": []}`,
     `{"trunc": a}`, `{"hcomp": ...}`);
  5. a path whose body does not use its interval variable: `{"refl": v}`;
  6. any other path: `{"from": a, "to": b, "term": "normal form"}`.
- Thus a coupling (a path in `Rand T`) shows the table of `s` and the
  pointwise paths. A loop is not only its end points, so rules 1 and 4 come
  before rule 6.

## 7. Slices and gates

- S0 (this commit): `bin/new-lang.sh coin-lang tcc-json`,
  `design/DESIGN.md` (the request without change) and this plan.
- S1: interval and paths. `I`, `PathP`, path abstraction and application,
  the end point rules, `refl`, `sym`, `cong` and function extensionality as
  terms. The kit `Eq` goes. Path defs become instances; JSON rules 5 and 6.
  Refusals: a wrong end point, `I` used as a type in `U`, and a conversion
  that holds only by irrelevance.
- S2: Kan operations. `transp` and `hcomp` for each former of section 4
  except Glue and the HITs. Library: `transport`, `subst`, `J`, path
  composition. Tests: transport in Pi, Sigma and PathP lines; `J` rule as a
  path.
- S3: Glue and univalence. `Glue`, `glue`, `unglue`, `hcomp` in `U`, `ua`,
  uaβ, `uaArrow`, `isoToEquiv`. JSON rule 1. Tests: transport along
  `ua (r -> r + 3)` on `Fin 11`; the 6 loops of `U` at `Fin 3` give 6
  distinct transport tables.
- S4: HITs. Declarations, boundary checks, formal `hcomp`, eliminators,
  `transp` with parameters. `Trunc1`, `Trunc0`, set quotients, `S1`. JSON
  rule 4. Refusals: a missing path case, a path case with a wrong boundary,
  and an elimination of `Trunc0` into a type without an `isSet` proof.
- S5: the base span. `Fin n` and its ring operations, `finVal`, `finAll`,
  decidable equality with evidence, the removal of `Flag` and of the unused
  Nat operations. Library: `FinSet`, `El`, the `FinSet` of `Fin n`, of a
  Sigma, and of `Fin 2 * Fin 24`.
- S6: `Rand`, its monad laws as paths, `couple`, `count`, `Bag`, `match`
  and the fiber theorem in both directions. JSON rules 2 and 3 and the
  `Rand` rule.
- S7: the two HVZK examples of section 2 as instances: the real variable,
  the simulated variable and the path between them. Mutants: a wrong
  witness `w`, a wrong `s` and a non-bijective `s` must each give a check
  refusal.
- S8: `SPEC.md`, `probe/CAPABILITY.md`, `docs/STATUS.md`,
  `docs/VALIDATION.md` and `README.md`. Final gate.
- Gate: `make check` in this directory.

## 8. Open items

- O1: the name coin-lang.
- O2: the kernel route. Default: CCHM as in Cubical Agda, local to this
  repository. The agda/cubical library (`ua`, `uaArrow`, `isoToEquiv`,
  `FinSet`, the set truncation) is the reference for the guest prelude.
  Other routes: cartesian CTT with V types (redtt, cooltt); or a new
  lang-template kit (for example `hosts/tcc-json-cubical`). Closed in S1
  (USER ruling 2026-10-14): the default, CCHM, local to this repository.
- O3: `FinSet` as a library Sigma over `U` (default) or as a primitive
  universe of codes. A primitive universe still needs Glue for its Kan
  operations, so it saves no kernel work.
- O4: `bind` on `Rand` uses the Sigma of the coin spaces. This is the bag
  monad. Its distribution is the uniform one when each continuation has a
  coin space of the same size. A graded bind on the product coin space is
  an option.
- O5: the Schnorr group: the subgroup of order 11 mod 23 (default) or the
  additive group `Z_q` with generator 1.
- O6: "JSON deserialization" read as the kit JSON document.
- O7: `Trunc0` is in the language, because the design names it.
  strategy-lang O3 forbade truncation; this design asks for it.
- O8: the kit `Eq` goes in S1, or stays as a name for `Path`. Closed in
  S1 (ruling R3 (a)): `Eq` is gone. `Path A a b` is `PathP (fun _ => A) a
  b`.
- O9: graph isomorphism coins as `Fin 2 * Fin 24` with a permutation table
  (default), or as the loop type of `FinSet` at `Fin 4`. The second needs a
  `FinSet` proof for `Equiv (Fin 4) (Fin 4)` (a Lehmer code).
- O10: `match` needs decidable equality on `T`. S6 proves it for `T` made
  from `Fin`, products and Sigma over `Fin`.
- O11: the `J` rule on `refl` is a path (CCHM). It is not a conversion.
  Closed in S2a: `JRefl` in `domain/domain.lang` is a path by `transp`
  on a constant line. The check refusal `j-refl-conv` shows that `refl`
  does not prove it at a neutral motive.
- O12: fill `probe/CAPABILITY.md` (S8).
