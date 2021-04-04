<!-- https://github.com/nnethercote/perf-book/commit/030ab92fbee6ec877c9fcba08237cbeee40cc996 -->

# 標準ライブラリの型

[`Vec`]、[`Option`]、[`Result`]、そして [`Rc`]/[`Arc`] のような、よく使われる標準ライブラリの型のドキュメントを読むことは、パフォーマンスを向上させるのに使える面白い関数を見つけることに繋がります。

[`vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[`option`]: https://doc.rust-lang.org/std/option/enum.Option.html
[`result`]: https://doc.rust-lang.org/std/result/enum.Result.html
[`rc`]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[`arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html

また、[`Mutex`]、[`RwLock`]、[`Condvar`]、そして [`Once`] のような、パフォーマンス向上という視点で他の標準ライブラリの型の代替となり得る型についても知っておくべきでしょう。

[`mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`rwlock`]: https://doc.rust-lang.org/std/sync/struct.RwLock.html
[`condvar`]: https://doc.rust-lang.org/std/sync/struct.Condvar.html
[`once`]: https://doc.rust-lang.org/std/sync/struct.Once.html

## `Vec`

[`Vec::remove`] は特定のインデックスの要素を削除し後ろにある要素を 1 つずつ左にシフトさせます。計算量は O(n) です。対して、[`Vec::swap_remove`] は特定のインデックスの要素を最後の要素と入れ替えつつ削除します。これは順序を保持しませんが、計算量は O(1) です。

[`Vec::retain`] は `Vec` から複数のアイテムを効果的に削除します。`String`、`HashSet`、そして `HashMap` のような他のコレクション型にも同様のメソッドがあります。

[`vec::remove`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove
[`vec::swap_remove`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.swap_remove
[`vec::retain`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain

## `Option` と `Result`

[`Option::ok_or`] は `Option` を `Result` に変換します。もし `Option` の値が `None` だった場合には、`ok_or` の引数が `err` のパラメータとして渡されます。`err` は先行評価されます。もしそのコストが大きい場合には代わりに [`Option::ok_or_else`] を使用して、クロージャを通してそのエラーの値を遅延評価するようにすべきです。例えばこれは:

```rust
# fn expensive() {}
# let o: Option<u32> = None;
let r = o.ok_or(expensive()); // 常に `expensive()` として評価される
```

このように変更すべきです:

```rust
# fn expensive() {}
# let o: Option<u32> = None;
let r = o.ok_or_else(|| expensive()); // 必要なときだけ `expensive()` として評価される
```

- [**例**](https://github.com/rust-lang/rust/pull/50051/commits/5070dea2366104fb0b5c344ce7f2a5cf8af176b0)

[`option::ok_or`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or
[`option::ok_or_else`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or_else

[`Option::map_or`]、[`Option::unwrap_or`]、[`Result::or`]、[`Result::map_or`]、そして [`Result::unwrap_or`] のような、似たような類似メソッドが他にもあります。

[`option::map_or`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.map_or
[`option::unwrap_or`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_or
[`result::or`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.or
[`result::map_or`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.map_or
[`result::unwrap_or`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or

## `Rc`/`Arc`

[`Rc::make_mut`]/[`Arc::make_mut`] は "clone-on-write" なセマンティクスを提供し、`Rc`/`Arc` への可変参照を作成します。もし参照カウントが 1 より大きい場合には、所有権が一意であることを確保するために中の値をクローンします。そうでなければ、つまり参照カウントが 1 であれば、元の値を変更します。これらのメソッドは頻繁には必要になりませんが、場合によってはとても役に立ちます。

- [**例 1**](https://github.com/rust-lang/rust/pull/65198/commits/3832a634d3aa6a7c60448906e6656a22f7e35628)
- [**例 2**](https://github.com/rust-lang/rust/pull/65198/commits/75e0078a1703448a19e25eac85daaa5a4e6e68ac)

[`rc::make_mut`]: https://doc.rust-lang.org/std/rc/struct.Rc.html#method.make_mut
[`arc::make_mut`]: https://doc.rust-lang.org/std/sync/struct.Arc.html#method.make_mut

## `Mutex`、`RwLock`、`Condvar`、そして `Once`

[`parking_lot`] クレートは上記の標準ライブラリの同期的な型よりも小さく、高速で、フレキシブルな代替実装を提供します。`parking_lot` にある型の API とセマンティクスは標準ライブラリのそれらと似ていますが同じものではありません。

[`parking_lot`]: https://crates.io/crates/parking_lot
