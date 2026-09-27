# Skeletons

三套可直接抄的最小模板（std only，用 `rustc --edition 2021 --test` 验证通过）。
完整上下文见讲义 F（eDSL）、E 4.2.4（mock）、O 14.2.4（校验）。

## 1. 命令 enum + 三解释器（同一份脚本，真跑 / mock / 干跑）

```rust
use std::cell::RefCell;
use std::collections::HashMap;

/// 命令 enum：领域动作 = 数据。脚本 = Vec<Cmd>。
#[derive(Debug, Clone, PartialEq)]
pub enum Cmd {
    SetupSensor { name: String, addr: u16 },
    ReadSensor { name: String },
    Report { message: String },
}

pub type Script = Vec<Cmd>;

/// 解释器是普通函数：吃掉命令，产出效果。纯逻辑只组装 Script。
pub trait Runtime {
    fn setup(&mut self, name: &str, addr: u16);
    fn read(&mut self, name: &str) -> f64;
    fn report(&mut self, msg: String);
}

/// 真解释器：读写真实世界（此处用 HashMap 代表硬件表）。
#[derive(Default)]
pub struct Real {
    sensors: HashMap<String, f64>,
}

impl Runtime for Real {
    fn setup(&mut self, name: &str, addr: u16) {
        self.sensors.insert(name.to_string(), addr as f64);
    }
    fn read(&mut self, name: &str) -> f64 {
        *self.sensors.get(name).unwrap_or(&f64::NAN)
    }
    fn report(&mut self, msg: String) {
        println!("{msg}");
    }
}

/// Mock 解释器：可编程脚本 + 调用记录。
#[derive(Default)]
pub struct Mock {
    pub readings: HashMap<String, f64>,
    pub log: RefCell<Vec<String>>,
}

impl Runtime for Mock {
    fn setup(&mut self, name: &str, addr: u16) {
        self.readings.insert(name.to_string(), addr as f64);
    }
    fn read(&mut self, name: &str) -> f64 {
        self.log.borrow_mut().push(format!("read {name}"));
        *self.readings.get(name).unwrap_or(&0.0)
    }
    fn report(&mut self, msg: String) {
        self.log.borrow_mut().push(msg);
    }
}

/// 干跑解释器：只校验，不执行。
#[derive(Default)]
pub struct DryRun {
    pub errors: Vec<String>,
}

impl Runtime for DryRun {
    fn setup(&mut self, name: &str, _addr: u16) {
        if name.is_empty() {
            self.errors.push("empty sensor name".to_string());
        }
    }
    fn read(&mut self, name: &str) -> f64 {
        if name.is_empty() {
            self.errors.push("read with empty name".to_string());
        }
        0.0
    }
    fn report(&mut self, _msg: String) {}
}

/// 跑脚本：同一个 run，换解释器即换执行方式。
pub fn run(rt: &mut impl Runtime, script: &[Cmd]) {
    for cmd in script {
        match cmd {
            Cmd::SetupSensor { name, addr } => rt.setup(name, *addr),
            Cmd::ReadSensor { name } => {
                let v = rt.read(name);
                rt.report(format!("{name} = {v}"));
            }
            Cmd::Report { message } => rt.report(message.clone()),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn demo_script() -> Script {
        vec![
            Cmd::SetupSensor { name: "t1".into(), addr: 0x48 },
            Cmd::ReadSensor { name: "t1".into() },
        ]
    }

    #[test]
    fn same_script_runs_against_mock() {
        let mut mock = Mock::default();
        run(&mut mock, &demo_script());
        assert_eq!(
            *mock.log.borrow(),
            vec!["read t1".to_string(), "t1 = 72".to_string()]
        );
    }

    #[test]
    fn dry_run_catches_bad_script() {
        let mut dry = DryRun::default();
        run(&mut dry, &[Cmd::SetupSensor { name: "".into(), addr: 1 }]);
        assert!(!dry.errors.is_empty());
    }
}
```

判据重申：只有当第二个解释器出现（mock、干跑、审计三者任一），
这套样板才回本；永远单上下文，直接写函数。

## 2. Service Handle + 可配置 mock（含调用计数）

```rust
use std::cell::RefCell;
use std::rc::Rc;
// 跨线程把 Rc 换成 Arc，RefCell 换成 Mutex，形状不变。

pub struct HwHandle {
    pub read_temp: Box<dyn Fn() -> f64>,
}

/// 闭包交出的是副本/共享引用，原件留在环境里 → 保持 Fn，可反复调用。
/// 若写成 `move || name`（交出捕获值所有权），闭包只实现 FnOnce，
/// 第二次调用即 E0382。这就是 mock 句柄必须用 Rc<RefCell<..>> 的原因。
pub fn configurable_mock(first: f64, calls: &Rc<RefCell<u32>>) -> HwHandle {
    let n = Rc::clone(calls);
    HwHandle {
        read_temp: Box::new(move || {
            *n.borrow_mut() += 1;
            first
        }),
    }
}

// 调用字段里的闭包要多一对括号：(h.read_temp)()
#[test]
fn mock_counts_calls() {
    let calls = Rc::new(RefCell::new(0));
    let h = configurable_mock(25.0, &calls);
    assert_eq!((h.read_temp)(), 25.0);
    assert_eq!((h.read_temp)(), 25.0);
    assert_eq!(*calls.borrow(), 2);
}
```

无需 mock 框架：替换实现就是 Service Handle 的基本操作。
多个可区分的 mock 再给身份加名字/GUID 即可（讲义 E 4.2.4.2）。

## 3. 累积校验 Validated（面包屑路径）

```rust
#[derive(Debug, PartialEq)]
pub enum Validated<T> {
    Ok(T),
    Err(Vec<String>),
}

impl<T> Validated<T> {
    /// 面包屑路径：给本层所有错误加前缀，如 "address.zip: 为空"。
    pub fn at(self, path: &str) -> Self {
        match self {
            Validated::Ok(v) => Validated::Ok(v),
            Validated::Err(es) => Validated::Err(
                es.into_iter().map(|e| format!("{path}: {e}")).collect(),
            ),
        }
    }

    /// 应用式组合：两边错误累积，而不是 `?` 那样首错即停。
    pub fn zip<U, V>(self, other: Validated<U>, f: impl FnOnce(T, U) -> V) -> Validated<V> {
        match (self, other) {
            (Validated::Ok(a), Validated::Ok(b)) => Validated::Ok(f(a, b)),
            (Validated::Ok(_), Validated::Err(e))
            | (Validated::Err(e), Validated::Ok(_)) => Validated::Err(e),
            (Validated::Err(mut e1), Validated::Err(e2)) => {
                e1.extend(e2);
                Validated::Err(e1)
            }
        }
    }
}

#[test]
fn validation_accumulates() {
    let a: Validated<u32> = Validated::Err(vec!["为空".into()]);
    let b: Validated<u32> = Validated::Err(vec!["超长".into()]);
    assert_eq!(
        a.at("name").zip(b.at("addr"), |x, y| (x, y)),
        Validated::Err(vec!["name: 为空".into(), "addr: 超长".into()])
    );
}
```

校验放边界层（API 入口 / 落库前），领域核心只收已校验的值。
