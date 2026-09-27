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

## 三张图：从印象到路线图（ch 3）

约束从松到紧，顺序不能倒：允许错 → 允许乱 → 要求定。

1. 必要性图：回答"做什么"。概念图：图里每个实体对应用都是必要的。
   第一版只求画出印象，允许不准确——目标是照亮不确定的区域。
   进阶：在中心放一条互操作总线，让组件不直连，把交互逻辑抽象成架构解。
2. 元素图：回答"怎么做"。三张里最自由：无中心对象、元素平等、无内容
   限制（概念、术语、库、层、对象皆可）、允许不连接。给元素打标签的过程
   就是确认它位置的过程。可以迭代多张。
3. 架构图：树状、无环、无孤立元素。包含重要子系统、关系与具体决策
   （库/接口/模块）。一个"汉堡块" = 组件 + 它的实现。关系只有两种：
   "交互"（双向）与"使用"（单向）。

架构图是路线图，不是契约：原书第 9 章会推翻第 3 章的一个架构决策，
这是正常的——修改理由是工程判断（复杂度不划算），而不是图没人看。
Rust 落点：架构图的 Cargo 版就是 workspace 布局；互操作总线落为
trait 边界或消息通道。

## 端到端清理清单（ch 4）

先还债，再加功能。顺序如下：

1. 修依赖方向：承载接口的范围不依赖任何实现；没有任何东西依赖 Impl。
   检查方法：画出模块依赖图，凡指向实现的箭头都是错的。
2. 修命名：消灭魔法常数与不一致前缀；厂商/来源各自命名空间；
   声明式定义加 Def 后缀；与 Hdl 强耦合的类型就和 Hdl 待在一起。
3. 加访问器：不透明类型配"知道如何处理它"的函数；接口纪律先于测试。
4. 用测试暴露问题，再用 trait 解决：先写出"我够不到句柄"的测试，
   再引入 trait——这个顺序本身就是端到端设计。
5. Service Handle 只做子层解耦，不撑整个架构（与自由单子/ReaderT/
   final tagless 的横向比较见 `rust-patterns.md` 接口表）。
6. mock 无需框架：替换实现就是换解释器。可配置 mock = 可区分身份
   （名字/GUID）+ 调用计数（单线程 `Rc<RefCell<..>>`，跨线程
   `Arc<Mutex<..>>`）。Rust 细节：结构体字段里的闭包调用要写 `(h.f)(..)`；
   字段必须是 `Box<dyn Fn(..)>` 而非 FnOnce（见 `skeletons.md` 的 E0382
   反例）；性能敏感路径选 trait（零成本单态化），否则每个字段都是一次
   堆分配加间接调用。
