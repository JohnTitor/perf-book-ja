<!-- commit: https://github.com/nnethercote/perf-book/commit/19db3a765030ed7c394a987eff5c09f639f0607d -->

# ラッパー型

([原文](https://nnethercote.github.io/perf-book/wrapper-types.html))

Rust は [`RefCell`] や [`Mutex`] のような、値に対して特別な動作を行う様々なラッパー型を提供しています。そのような値へのアクセスは無視できない回数になることもあります。もしそのような複数の値が同時にアクセスされる場合には、それらを単一のラッパーに包んだ方が良いでしょう。

[`refcell`]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[`mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html

例えば以下のような構造体は:

```rust
# use std::sync::{Arc, Mutex};
struct S {
    x: Arc<Mutex<u32>>,
    y: Arc<Mutex<u32>>,
}
```

このように表現した方が良いでしょう:

```rust
# use std::sync::{Arc, Mutex};
struct S {
    xy: Arc<Mutex<(u32, u32)>>,
}
```

これがパフォーマンスの向上につながるかは、値への実際のアクセスパターンに依存します。

- [**例**](https://github.com/rust-lang/rust/pull/68694/commits/7426853ba255940b880f2e7f8026d60b94b42404)
