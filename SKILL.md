---
name: functional-design-rust
description: Guide Rust functional-declarative design with layered architecture, eDSL modeling, and testable interfaces.
---

# Functional Design Rust

Design Rust systems with Functional Declarative Design (FDD): pure domain
core, eDSL + interpreter seams, explicit layering, and testable interfaces.

Source distilled: Alexander Granin "Functional Design and Architecture"
(Haskell original), via a 17-part Rust reading course (docx讲义 A–Q +
per-chapter flashcards). This skill contains the method, not the book text.

## When to use

Use when the user is designing or refactoring a Rust system and any of these
apply: choosing module/crate layering, modeling a domain with enums/traits,
deciding a dependency-injection shape, handling state/concurrency/resources,
drawing a persistence or error-handling boundary, or making code testable.

Do not use for pure language questions (borrow checker, lifetimes, async
syntax) with no design component — answer those directly.

## Design workflow

Follow these steps in order. Skip a step only by saying why.

1. Requirements first: restate functional vs non-functional needs in one
   paragraph. If the domain vocabulary is unclear, build a small concept map
   (mind map) before writing types. See `references/fdd-method.md`.
2. Fix the layers: assign every piece to application / service-interface /
   domain-core (pure) / persistence / presentation. Pure core crate must not
   depend on any IO crate. Dependency direction points inward toward the
   domain. Cargo workspace structure is the executable architecture diagram.
3. Signature-first skeleton: write public function/trait signatures with
   `todo!()` bodies so the whole skeleton compiles, then fill in. Let E0308
   type mismatches act as design review.
4. Model the domain as data: use `enum` + `struct` (ADT). Make illegal states
   unrepresentable (newtypes, enums instead of strings/bools, phantom tags
   only when a real mix-up exists). If behavior varies by execution context
   (real/mock/dry-run/audit), model actions as a command `enum` (eDSL) plus
   one interpreter function per context — never scatter execution inside the
   model.
5. One interpreter per context: same script runs against real, mock,
   dry-run, and logging interpreters. eDSL pays off at the second
   interpreter; with only one execution mode, plain functions are better.
6. Choose the interface shape deliberately (see `references/rust-patterns.md`):
   Service Handle (struct of closures/fns), `AppContext` reader param, command
   enum + interpreter, request enum with per-variant response type, or
   trait + generic param (final-tagless style). Default: trait for stable
   boundaries + Service Handle for easily swappable behavior.
7. Push impurity to the seam: pure core returns values; exactly one layer
   interprets effects (IO, threads, channels, DB). Separate typed facade from
   untyped runtime table (Typed-Untyped) where handles cross the boundary.
8. Errors as values with domains: one error enum per layer/crate (`thiserror`
   for libraries), `anyhow`/boxed errors only at the app edge. Convert foreign
   errors at the interpreter boundary; never leak `Stringly` errors inward.
   Decide the panic boundary explicitly (panic = bug, `Result` = expected).
9. Resources via RAII: prefer `Drop` guards and scoped ownership over manual
   bracket calls; check the four pitfalls (early return, forget/cancel, thread
   boundary, async drop).
10. Test for the seam: property tests (`proptest`, strategies over filters)
    for pure logic; trait-based doubles (fixed/configurable/programmable/
    record-replay) for interpreters; record-replay snapshot tests only as
    change detectors, never as correctness proofs.

## Rules

- Pure stays pure: no `std::fs`, `std::net`, time, rand, or global mutable
  state inside the domain core. If it is needed, it becomes a command in the
  eDSL and is interpreted outside.
- No stringly domain: bare `String`/`bool` pairs describing domain facts must
  become newtypes or enum variants.
- One execution mode = no eDSL: do not wrap single-context calls in a command
  enum.
- API types and domain types are separate structs, with an explicit mapping
  function at the boundary. Same for DB rows vs domain entities.
- Every `trait` boundary gets at least a mock/doubles story before it ships.

## Rust pattern quick map

- Haskell ADT -> Rust `enum` + `struct`, exhaustive `match`.
- `Maybe`/`Either` -> `Option`/`Result` (note reversed type params).
- Free monad -> command `enum` + interpreter fn; nesting = upper enum variant
  holds lower enum value (HFM: upper interpreter calls lower interpreter).
- Typeclass -> `trait` + associated types; per-request response ~ associated
  type or generic param; no HKT — use the three SQL routes in the reference.
- `ReaderT` -> pass `&AppContext` (zero cost); `StateT` -> explicit state
  param or a state command pair in the eDSL.
- STM -> `Mutex`/`RwLock`/channels; keep transactions small and
  non-blocking; split read-compute-write races explicitly.
- Bracket -> `Drop` guard.
- Servant type-level routing -> value-level router table (axum/actix).
- Lens -> usually unnecessary (`&mut` suffices).

## References

- `references/fdd-method.md` — pillars, layers, top-down flow.
- `references/rust-patterns.md` — Haskell→Rust pattern choices with costs.
- `references/chapters-index.md` — which chapter/method applies to a problem,
  plus the quiz mode.

When a decision matches a reference section, follow it and cite the file.
When the references do not cover the case, fall back to the workflow above
and state the assumption.
