# coin-lang domain design

> Paste the domain design below the line, verbatim. Do not edit it.

This file is the only source of the core types and the core operations of
coin-lang. `SPEC.md` sections 5 and 6 cite its sections. Do not add a core
type or a core operation that this file does not give.

When the design changes, paste the new version. Then record the change as an
open item in `SPEC.md` section 9.

---

I want to build a programming language with the following features:

For type constructors, we have products, coproducts, monads (map/bind), algebras and coalgebras (fold/unfold), and paths/loops/quotients (as in Cubical TT).  For base types and operations, restrict to the minimum span required to model the following:

Perfect HVZK as an equivalence of coin spaces. This is what I would build
     first.
     - Model a uniform random variable as Σ (Ω : FinSet) (Ω → T). By
       univalence, a path from (Ω, f) to (Ω', g) is an equivalence σ : Ω ≃ Ω'
       with g ∘ σ = f, which is a bijective coupling.
     - For uniform distributions on sample spaces of the same size, equal
       distributions hold exactly when such a σ exists, because the fibers
       have equal sizes.
     - Perfect HVZK for Schnorr or for graph isomorphism is then a path from
       the real transcript variable to the simulated one. The masking step z =
       r + e·w is the equivalence r ↦ r + e·w on Z_q.
     - The set truncation of this groupoid is the type of finite multisets.

In order to enable non-trivial loops/quotients, proof irrelevance is prohibited.

The host language shall be TinyCC, and the compilation target shall be a JSON deserialization of the final state of each type instantiated in the program.
