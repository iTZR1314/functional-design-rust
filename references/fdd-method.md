# FDD Method

Four pillars, layering, and the top-down flow. Distilled from chapters 0–2
(讲义 A/B/C); details live in the course docx, not here.

## Four pillars

1. Domain as eDSL: actions of the domain become constructors (enum variants),
   not scattered strings or method calls. "万物皆 eDSL" — but only where it
   pays (see rule below).
2. Description vs execution split: build pure values (the script), run them
   with an interpreter (the machine). Same script, many interpreters: real,
   mock, dry-run, audit-log.
3. Constraints in types: illegal states unrepresentable — newtype over String,
   enum over bool-pairs, phantom tags only against real key mix-ups.
4. Layered architecture for effects: pure domain core in the center; impurity
   lives in exactly one interpreting layer per seam.

Payoff test: eDSL is worth it at the second interpreter. One execution mode
-> plain functions.

## Layers (outer to inner)

- Application: `main.rs`/app crate — args (clap), logging init (tracing),
  runtime startup, config. May own threads.
- Service interfaces: traits only, no implementations. One crate (or module)
  holding the seams.
- Domain core: structs/enums + pure fns. `Cargo.toml` has no IO deps
  (no fs/net/time/rand). Depends on nothing impure.
- Persistence: `trait Repository` (pure interface) + sqlx/redis/file impls
  (impure). DB rows map to domain entities explicitly.
- Presentation: CLI/GUI/HTTP handlers (ratatui/egui/axum). Thin: parse,
  call service, render.

Dependency rule: outer depends on inner; inner never imports outer. Enforce
with Cargo workspace layout (cyclic deps fail the build — use that).

A `utils` bag of string helpers is a drawer, not a layer: layers carry design
responsibilities and appear on the architecture sketch.

## Top-down, signature-first

1. Start from one top-level function signature per use case.
2. Stub everything with `todo!()` so the skeleton compiles.
3. Decompose: each stub's type mismatch (E0308) is a design question.
4. Iterate in small compiler-checked steps — top-down, not waterfall.

```rust
type Fahrenheit = f64;
type Celsius = f64;

fn read_sensor(_id: &str) -> Fahrenheit { todo!() }
fn to_celsius(_f: Fahrenheit) -> Celsius { todo!() }
fn report(_c: Celsius) -> String { todo!() }

fn pipeline(id: &str) -> String {
    report(to_celsius(read_sensor(id)))
}
```

## Requirements to types

- Separate functional (what it computes) from non-functional (speed, volume,
  operability) requirements first.
- For unclear vocabulary: association mind map of domain concepts, then turn
  nouns into types and verbs into eDSL commands or pure functions.
- Architecture sketch in two forms: a diagram AND the Cargo workspace that
  implements it.

## HFM in one paragraph

Hierarchical free monads = nested languages. Upper-layer command enum holds
lower-layer commands as variant payloads; the upper interpreter delegates to
the lower interpreter. Subsystem boundaries sit between eDSL languages and
meet only in business logic.
