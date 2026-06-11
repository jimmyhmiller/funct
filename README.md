# Funct

I created funct for use in my editor [Jim](https://github.com/jimmyhmiller/jim). I originally start with rhai but found the lack of being able to pause execution too limiting. I use funct for all my widgets and want to make sure an infinite loop doesn't cause my frames to drop. This feature right now is still definitely not fully tested, but it was crucial for my future plans so it felt worth making this.

This software is **Pre-Alpha**. I will make breaking changes and it is very untested. (This library has been vibe coded.)

**Docs:** [Language Guide](docs/guide.md) · [Standard Library](docs/stdlib.md) · [Spec](docs/funct-spec.md)

---

## A taste of the language

```rust
type Shape = Circle { radius } | Square { side }

fn area(s) = match s {
    Circle { radius } => 3.14159 * radius * radius,
    Square { side }   => side * side,
}

let shapes = [Circle { radius: 1.0 }, Square { side: 2.0 }]
let total  = shapes |> map(area) |> sum

let hits = atom(0)
swap!(hits, n => n + 1)

fn parse_pair(a, b) {
    let x = parse_int(a)?
    let y = parse_int(b)?
    Ok((x, y))
}

match parse_pair("3", "4") {
    Ok((x, y)) => println("sum = ${x + y}"),
    Err(msg)   => println("oops: ${msg}"),
}
```

---

## Embedding in Rust

```rust
use funct::{Funct, vals};

let mut vm = Funct::new();

// expose Rust functions — arguments and return values convert automatically
vm.register1("double", |x: i64| x * 2);
vm.register2("add", |a: i64, b: i64| a + b);

// or expose a whole Rust type, with constructors, fields, and methods
vm.register_type::<Player>("Player")
    .ctor2("new_player", |name: String, hp: i64| Player { name, hp })
    .field("hp", |p| p.hp)
    .method1("damage", |p, n: i64| { p.hp -= n; p.hp });

// or bundle functions into a module the script can import
vm.register3("lerp_impl", |a: f64, b: f64, t: f64| a + (b - a) * t);
let lerp = vm.native_fn("lerp_impl").unwrap();
vm.register_module("math", vec![("lerp", lerp)]);

vm.eval(r#"
    import { lerp } from "math"
    let p = new_player("hero", 10)
    p.damage(3)
    lerp(0.0, 10.0, 0.5)
"#)?;

// and call back into the script from Rust, with a typed return
let result: i64 = vm.call_typed("compute", vals![20, 2])?;
```

---

## Built for live editing: pause, snapshot, resume

```rust
use funct::{Funct, Value, StopWhen, RunResult, Cause};
use std::time::Duration;

let mut vm = Funct::new();
vm.eval(/* a long-running function `work` */)?;

// run with a deterministic instruction budget instead of to completion
let mut st = vm.start("work", vec![Value::Int(100)])?;

match vm.run(&mut st, StopWhen::Fuel(500)) {
    RunResult::Paused(Cause::FuelExhausted) => {
        let json = vm.save_state(&st)?;
        std::fs::write("state.json", json)?;
    }
    RunResult::Done(value)  => { /* finished */ }
    RunResult::Faulted(f)   => { /* runtime fault, as a value */ }
    _ => {}
}

let mut vm = Funct::new();
let mut st = vm.restore_state(&std::fs::read_to_string("state.json")?)?;
vm.run(&mut st, StopWhen::Never);
```

---

## License

MIT — see [LICENSE](LICENSE).
