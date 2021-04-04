<!-- commit: https://github.com/nnethercote/perf-book/commit/19db3a765030ed7c394a987eff5c09f639f0607d -->

# インライン化

([原文](https://nnethercote.github.io/perf-book/inlining.html))

ホットでインライン化されていない関数の呼び出しはしばしば実行時間の無視できない部分を占めることがあります。それらの関数をインライン化することで、小さいですが簡単な高速化を図ることができます。

Rust の関数向けには4種類のインライン属性があります:

- **なし**: コンパイラ自身がその関数がインライン化されるべきなのかを判断します。これは最適化レベルや関数サイズなどに依存します。もしリンク時最適化を使っていない場合には関数がクレートを跨いでインライン化されることはありません。
- **`#[inline]`**: これはクレートの垣根を越えて関数がインライン化されるべきであることを示します。
- **`#[inline(always)]`**: これはクレートの垣根を越えて関数がインライン化されるべきであることを **強く** 示します。
- **`#[inline(never)]`**: これは関数がインライン化されるべきでないことを **強く** 示します。

インライン属性は関数がインライン化される/されないことを保証するものではありません。しかし実際には、`#[inline(always)]` は例外的な場合を除きすべての場合においてインライン化を行います。

## シンプルなケース

インライン化するのに最適な候補は (a) とても小さい、あるいは (b) コールサイト (call site) が一つである、といったような関数です。コンパイラはインライン属性がなくともそれらの関数を自身の判断でインライン化することがしばしばあります。ですが、コンパイラはいつも最適な選択をするわけではないので、属性の付与が時々必要となります。

- [**例 1**](https://github.com/rust-lang/rust/pull/37083/commits/6a4bb35b70862f33ac2491ffe6c55fb210c8490d)
- [**例 2**](https://github.com/rust-lang/rust/pull/50407/commits/e740b97be699c9445b8a1a7af6348ca2d4c460ce)
- [**例 3**](https://github.com/rust-lang/rust/pull/50564/commits/77c40f8c6f8cc472f6438f7724d60bf3b7718a0c)
- [**例 4**](https://github.com/rust-lang/rust/pull/57719/commits/92fd6f9d30d0b6b4ecbcf01534809fb66393f139)
- [**例 5**](https://github.com/rust-lang/rust/pull/69256/commits/e761f3af904b3c275bdebc73bb29ffc45384945d)

Cachegrind は関数がインライン化されているかどうかを確かめるのに適したプロファイラーです。Cachegrind の出力を見てみてください。最初と最後の行がイベントカウントとしてマーク **されていない** 場合にのみ、関数がインライン化されていることが分かります。

例:

```text
      .  #[inline(always)]
      .  fn inlined(x: u32, y: u32) -> u32 {
700,000      eprintln!("inlined: {} + {}", x, y);
200,000      x + y
      .  }
      .  
      .  #[inline(never)]
400,000  fn not_inlined(x: u32, y: u32) -> u32 {
700,000      eprintln!("not_inlined: {} + {}", x, y);
200,000      x + y
200,000  }
```

影響が予測できないこともあるため、インライン属性を追加した後には測定し直すべきです。時々、以前はインライン化されていた近くの関数がインライン化されなくなったことで、なんの効果もないことがあります。また、コードが遅くなることもあります。インライン化はコンパイル時間にも影響します。特に、関数の内部表現を複製する、クレートを跨ぐようなインライン化の場合には影響があることが多いです。

## 難しいケース

時々、巨大で複数のコールサイトを持つが、一つのコールサイトだけがホットな関数が生まれます。そのような場合、速度向上のためホットなコールサイトをインライン化しつつ、不要なコードの肥大化をさけるためにコールドな (ホットの逆で、稀にしか実行されない) コールサイトはインライン化したくないものです。これを処理する方法として、関数を、常にインライン化される (always-inlined) ものと決してインライン化されない (never-inlined) ものに分け、後者から前者を呼ぶというものがあります。

例えば、この関数は:

```rust
# fn one() {};
# fn two() {};
# fn three() {};
fn my_function() {
    one();
    two();
    three();
}
```

下のように2つの関数になり:

```rust
# fn one() {};
# fn two() {};
# fn three() {};
// ホットなコールサイトではこれを使う
#[inline(always)]
fn inlined_my_function() {
    one();
    two();
    three();
}

// コールドなコールサイト (ホットでないコールサイト) にはこれを使う
#[inline(never)]
fn uninlined_my_function() {
    inlined_my_function();
}
```

- [**例 1**](https://github.com/rust-lang/rust/pull/53513/commits/b73843f9422fb487b2d26ac2d65f79f73a4c9ae3)
- [**例 2**](https://github.com/rust-lang/rust/pull/64420/commits/a2261ad66400c3145f96ebff0d9b75e910fa89dd)
