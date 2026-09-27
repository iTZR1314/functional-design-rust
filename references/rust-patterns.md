# Rust Patterns (Haskell → Rust)

Choice guide with costs. Distilled from chapters 5–15 (讲义 F–P + 附录 Q).

## Interface shapes (ch 4 / 13)

| Shape | Rust form | Use when | Cost |
|---|---|---|---|
| Service Handle | struct holding `fn`/closures or a small trait object | behavior swapped per test/env; mock-heavy seam | closure `FnOnce`/`FnMut` friction; keep handles `Clone` or `Arc` |
| Reader / AppContext | `fn run(ctx: &AppCtx, ...)` | config/clock/shared handles threaded through logic | param noise; zero runtime cost |
| Command enum + interpreter | `enum Cmd {…}` + `fn interpret(cmd) -> …` | 2+ execution modes (real/mock/dry-run/audit) | boilerplate per variant; best testability |
| Per-request response type | enum where each variant carries its own output type (GADT-like) | heterogeneous request/response protocol | needs associated types or generics; keep protocol small |
| Final-tagless / trait-generic | `trait Lang { … }`, code generic over `L: Lang` | multiple backends with static dispatch | monomorphization + orphan-rule limits; avoid for tiny seams |

Default: trait for stable subsystem boundaries, Service Handle for
easily-swapped behavior, command enum where audit/dry-run matters.
Compare all five before locking an axis the system will pivot on.

## Typed–Untyped split (ch 9 / 10)

Typed facade outside, untyped table inside: handles (`VarId`, connection
keys) cross the boundary; bodies live in a runtime map. Applies to thread
registries, loggers, STM-style state. Benefit: stable typed API over a
flexible store. Cost: handle→row mismatch must be an explicit error, and the
abstraction should be skippable where it charges per call.

Global mutable logger: never `static mut` / `unsafePerformIO` equivalent.
Use `OnceLock<Arc<dyn Logger>>` (or `tracing`) — safe, standard.

## State (ch 7)

- Prefer: explicit state params → state command pair inside the eDSL →
  `Mutex`/`RwLock` in one owning layer → channels/actors across threads.
- `RefCell`/`Cell` for single-thread interior mutability only; `IORef`-style
  sharing breaks under concurrency.
- Concurrency: channels (`std::sync::mpsc`, tokio/flume) + actor per device
  (ch 8). Choose thread vs async by lifetime: short fan-out = scoped threads
  / pool; long-lived conversations = async tasks. Never borrow stack locals
  into `spawn` — move owned data or `Arc`.
- `Send`/`Sync` are the compiler's thread-safety proof: `Send` = movable,
  `Sync` = shareable. Passing the compiler solves data races, not
  race conditions (read-compute-write still needs a transaction/critical
  section). No STM in std: keep critical sections tiny and non-blocking.

## Resources (ch 10)

Bracket = `Drop` guard (auto-return tokens, file/connection guards). Four
pitfalls: early `return`/`?` skipping cleanup (guard fixes it), `mem::forget`
or leaked `Arc` suppressing `Drop`, guards crossing thread boundaries, and
`async drop` (no async `Drop` — expose an explicit `close().await`).

## Persistence

- KV (ch 11): untyped string store trait + typed `DBEntity` trait with
  associated `Key`/`Value` types; phantom tags against key mix-ups. Respect
  the expression problem + orphan rule: new backends via newtypes/new crates,
  not by editing the core trait per table.
- SQL (ch 12), three routes: (1) plain generic param per shape — simplest,
  some duplication; (2) open trait set per backend — extensible without
  touching existing code; (3) GAT-based columnar — closest to Haskell HKD,
  `derive` support is weak. Whichever ORM/query builder is chosen (sqlx /
  diesel / sea-orm), hide it behind `trait Repository` at the crate boundary
  and map rows ↔ domain explicitly. Never unify domain and DB structs.

## Errors (ch 13)

Errors are values, not events. One error enum per layer (`thiserror` for
libraries); `anyhow`/boxed errors only at the binary edge. Convert foreign
exceptions at the interpreter boundary (recommended over converting at the
language layer or discarding). In Rust the "free-monad exception" debate
collapses to `?` + typed errors. Panic boundary: panic = bug (fail fast),
`Result::Err` = expected failure (handle). `?` in `main` returns are fine
only after logging/telemetry is set up.

## Business logic (ch 14)

Three separate models with mapping functions: API/DTO types, domain types,
DB types. Validation is applicative and accumulating (collect ALL field
errors with breadcrumb paths like `address.zip`), not fail-fast `?` — write
a small `Validated<T>` accumulator. Routers are value-level tables (axum),
not type-level DSLs. Handlers stay thin: parse → call service → render.

## Tests (ch 15 + appendix)

- AAA per test; name the property, not the example.
- Property tests with `proptest`: express invariants, use strategies (not
  filters) to generate valid inputs, keep the minimal-counterexample habit.
- Doubles as a ladder: fixed value → configurable → programmable script →
  record-replay. One `trait`, several fakes.
- Record-replay (appendix): record interpreter traffic, replay and diff;
  five replay modes (strict/order-free/whitelist/timing-threshold/migrate).
  Change detector only — it never proves correctness. Snapshot goldens get
  the same warning.
- Coverage and mocks are not goals: cover seams and properties, not lines;
  every mock must correspond to a real interpreter contract test.
