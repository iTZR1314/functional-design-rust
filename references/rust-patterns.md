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

## Actor 协议：从"能通信"到"通信不错乱"（ch 8）

- 第一性原理：把"共享 + 加锁"换成"独占 + 传消息"。actor 状态是被
  move 进 actor 线程的局部变量，其他线程在类型层面就够不着——没有
  忘记加锁的可能。
- 请求—响应模式：两个单向通道合成一个双向 Pipe（请求通道 + 响应通道）。
  Rust 对 Haskell MVar 版的一处真实改进：响应方线程结束（正常或 panic）
  会 drop 它持有的 Sender，请求方 `recv()` 立刻得到 `Err` 而不是永远挂起。
  但这防不住所有死锁——两个线程互等对方的锁照样死。
- 不安全的协议长什么样：请求 enum 与响应 enum 是两个独立类型，
  `match req { Square(n) => Reversed(..) }` 这种错配能通过编译。
  "想要 Square 10 却收到 Reversed"是任何通信协议的共有问题。
- 修法：把回信通道绑进请求——`Ask<Q, A> { question: Q, reply: SyncSender<A> }`，
  答案类型由问题决定。把 `ask.answer` 的参数写错类型，看编译器说什么——
  这个五秒实验就是"类型安全的协议"的全部含义。
- 持有规则：需要向谁发消息，就持有发往它信箱的 Sender；回信走 Ask 自带的
  一次性 reply 通道，不另存。父持有发往子 actor 的 Sender，子只带走自己的
  Ask reply 加发往父/模拟器的 Sender。
- spawn 铁律：`thread::spawn` 闭包是 `'static`，借用的栈变量传不进去——
  move 所有权进去，共享用 `Arc`。
- MVar 模型的上限（原书总结）：并发数据模型变大变复杂时 MVar 力不从心，
  大型 MVar 模型趋向过度复杂，届时转向 STM。在 Rust 对应：通道 + 小临界区
  （见"状态"一节）。线程成本与 Haskell 绿色线程不同，默认粗粒度 actor 起步。

## 录制—重放实操（附录 D）

机制三句话：框架里烘焙序列化能力 → 录制模式跑场景，每步追加
`{step, entry, mode}` 进 JSON（含输入、输出、参数）→ 重放模式逐步比对，
不匹配精确报出第几步。这是第 15 章手写测试替身的自然演化：业务逻辑泛型
于 `trait Lang`，录制器（干真活 + 追加记录）与重放器（组装实际调用 →
比对 → 返回录制值）各实现一次，逻辑"意识不到魔法"。

serde 形状：`#[serde(tag = "tag")]` 的 Entry enum（每变体合同不同：
问答类比对输入输出，Tell 类只比 text）；`#[serde(default)]` 让 mode 可省，
录制文件可手改——小改直接改 JSON，不必重录。

五种模式（Normal 为默认）：

| 模式 | 行为 | 用于 |
|---|---|---|
| Normal | 比对 + 用录制值顶替 | 默认 |
| NoVerify | 放行本步差异 | 时间戳/UUID/随机数步骤；或把时钟做成可注入 |
| NoMock | 本步调真实实现 | 混合真实调用（DB/日志器） |
| GlobalNoVerify / GlobalNoMocking | 整份录制套用 | 大面积放行/联调 |
| GlobalSkip | 跳过 | 废弃场景 |

耗时阈值：`Recorded` 留 `time_threshold_ms: Option<f64>`（带
skip_serializing_if，不设不落盘），重放计时超限即报。但耗时断言只进独立
性能套件（criterion + `#[ignore]`），不进正确性测试——CI 负载抖动会让它变红。

Rust 生态对照：日常用 insta 快照固定"输出长什么样"（零成本）；只有要
"精确到第几步/中途注错/混合真实调用"才自搭录制—重放。

黄金纪律（最危险误读）：录制只固定行为，不判断对错。批准一份快照/录制 =
签字"这个行为是对的"。正确工作流：跑录制 → 人工逐条读（顺序、参数对吗）
→ 确认再提交。省掉中间那步，固定住的可能是一个 bug。
