# functional-design-rust

Rust 函数式声明式设计（FDD）skill：纯领域核心、eDSL + 解释器接缝、显式分层、可测试接口。

方法论提炼自 Alexander Granin《Functional Design and Architecture》（Haskell 原著），经由一套 17 讲 Rust 讲义整理。本 skill 只包含设计方法，不含原书正文。

## 目录

- `SKILL.md` — skill 本体：何时使用、十步设计工作流、硬规则、Haskell→Rust 速查
- `references/fdd-method.md` — 四支柱、分层表、自顶向下签名先行、HFM
- `references/rust-patterns.md` — 五种接口选型、Typed-Untyped、状态/并发/资源、KV/SQL 路线、错误域、累积校验、测试替身阶梯
- `references/chapters-index.md` — 十七讲路由表 + 测验模式

## 安装（Muse Code）

```sh
git clone https://github.com/iTZR1314/functional-design-rust.git
muse skills install functional-design-rust
```

或在 GitHub 页面点 Code → Download ZIP，解压后：

```sh
muse skills install <解压目录>
```

校验：

```sh
muse skills validate <目录> --json
```

## 使用示例

- “用 functional-design-rust 评审这个 crate 的分层”
- “按 FDD 把这段逻辑改成命令 enum + 双解释器（真实 + mock）”
- “按这个 skill 的错误域规则重排各层 error 类型”

## 说明

- 项目级用法：把本目录放进你的项目仓库 `.agents/skills/functional-design-rust/` 并提交，clone 即用，免安装。
- 讲义 docx 全文与闪卡 xlsx 不在包内（体积原因），需要时回原课程工作区查阅。
