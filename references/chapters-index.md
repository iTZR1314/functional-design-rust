# Chapters Index + Quiz Mode

17 course files: A–Q (导论, ch 1–15, 附录). Use this table to route a
problem to the right method. Full prose stays in the workspace docx; this is
the router, not a copy.

| File | Topic | Reach for it when |
|---|---|---|
| A 导论 | FDD claims, 4 pillars, Haskell→Rust map, roadmap | onboarding; "what is FDD / why Rust fits" |
| B ch1 | design = risk/complexity management; coupling/cohesion; IoC | justifying an abstraction; reviewing a tangled module |
| C ch2 | architecture levels, layer table, first eDSL, HFM preview | choosing layers; first command-enum design |
| D ch3 | MVP sketch: Andromeda; necessity/element/architecture diagrams; module privacy | starting a new service; drawing the three diagrams |
| E ch4 | end-to-end cleanup; Service Handle; pure vs impure methods; mock discipline | fixing dependency direction; writing first trait mocks |
| F ch5 | eDSL via ADT; deep vs shallow embedding; parameterized ADT; interpreters | domain has 2+ execution contexts |
| G ch6 | free monad as interface; why no generic `Free` in Rust; layered eDSL | deciding command-enum vs trait; nesting languages |
| H ch7 | state taxonomy; State monad → Rust advice; Brainfuck/IORef lessons | state placement; `Mutex` vs param vs eDSL-state |
| I ch8 | actors; MVar request–reply; channel choice; thread vs async | device/sim/concurrent conversations; deadlock triage |
| J ch9 | threads accounting; STM ideas; `Send`/`Sync`; effect containment | thread pool design; data-race vs race-condition bugs |
| K ch10 | logging; Typed–Untyped; traceable state; Bracket → `Drop` + 4 pitfalls | logger design; global state; resource guards |
| L ch11 | KV store; `DBEntity`; phantom keys; expression problem / orphan rule | typed KV boundary; key-mixup bugs |
| M ch12 | relational model; HKD 3 routes; GAT limits; SQL-ecosystem stance | table modeling; choosing sqlx/diesel/sea-orm hiding |
| N ch13 | error domains; 3 conversion points; 5 DI shapes compared | error-enum layout; `thiserror` vs `anyhow`; DI choice |
| O ch14 | CLI/HTTP client+server; 3-model split; accumulating validation | API boundary; validator with breadcrumb paths |
| P ch15 | AAA; property tests; doubles ladder; whitebox fragility | test plan; flaky/mock-heavy suites |
| Q 附录 | monads-in-Rust; word-count; record-replay rig; 5 replay modes | snapshot/auto-whitebox testing; "where did monads go" |

## Quiz mode (flashcards live in workspace `闪卡/`)

The workspace ships per-chapter decks (`闪卡/*.xlsx`, plus CSV for A/B).
They are the source project's study material and are NOT bundled into this
skill. To drill: pick a chapter row above, ask one question at a time in the
style of that chapter's deck (single/best-answer choice, answer with the
letter + one-sentence reason), wait for the user's answer, then explain by
citing the method file and the pattern file — never reveal two questions at
once, and never grade on text similarity, only on the design reason.
