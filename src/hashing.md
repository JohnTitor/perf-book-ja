<!-- commit: https://github.com/nnethercote/perf-book/commit/c30dcc00f96e3538d1cc1d01dadc861a71ffee5f -->

# ハッシュ化

([原文](https://nnethercote.github.io/perf-book/hashing.html))

`HashSet` と `HashMap` は広く使われる2つの型です。デフォルトのハッシュアルゴリズムは指定されていませんが、執筆時点では [SipHash 1-3] と呼ばれるアルゴリズムがデフォルトとなっています。このアルゴリズムは高い品質を持っており衝突に対する高い保護性能を持ちますが、その一方で特に整数のような短いキーについては遅いという特徴があります。

[SipHash 1-3]: https://en.wikipedia.org/wiki/SipHash

もしプロファイリングによりハッシュ化がホットであると判明し、[HashDoS attacks] がアプリケーションにおいて懸念にならない場合には、より高速なハッシュアルゴリズムを持つハッシュテーブルの使用により、大きな速度の改善ができます。

<!-- textlint-disable ja-technical-writing/no-doubled-conjunctive-particle-ga -->
- [`rustc-hash`] は `HashSet` や `HashMap` の代替となる `FxHashSet` や `FxHashMap` を提供しています。ハッシュアルゴリズムは低品質ですが、特に整数キーの場合にはとても高速で、rustc 内の他のどのハッシュアルゴリズムよりも性能が優れています（[`fxhash`] は同じアルゴリズムと型の実装を持ちますが、`rustc-hash` に比べると古くあまりメンテナンスされていません）
- [`fnv`] は `FnvHashSet` 及び `FnvHashMap` 型を提供しています。ハッシュアルゴリズムは `rustc-hash` よりも高品質ですが、その分速度はやや劣ります
- [`ahash`] は `AHashSet` 及び `AHashMap` を提供しています。`ahash` のハッシュアルゴリズムはいくつかのプロセッサで利用可能な AES 命令をサポートしています
<!-- textlint-enable ja-technical-writing/no-doubled-conjunctive-particle-ga -->

[HashDoS attacks]: https://en.wikipedia.org/wiki/Collision_attack
[`rustc-hash`]: https://crates.io/crates/rustc-hash
[`fxhash`]: https://crates.io/crates/fxhash
[`fnv`]: https://crates.io/crates/fnv
[`ahash`]: https://crates.io/crates/ahash

ハッシュ化のパフォーマンスがプログラムにおいて重要であれば、これらの代替案をいくつか試す価値があるでしょう。例えば、rustc では次の結果が得られました：

- `fnv` から `fxhash` への移行は[最大6%の高速化][fnv2fx]をもたらしました
- `fxhash` から `ahash` への移行の試みは [1-4%の速度低下][fx2a]という結果でした
- `fxhash` からデフォルトのハッシュアルゴリズムに戻す試みは[4-84%の速度低下][fx2default]という結果でした！

[fnv2fx]: https://github.com/rust-lang/rust/pull/37229/commits/00e48affde2d349e3b3bfbd3d0f6afb5d76282a7
[fx2a]: https://github.com/rust-lang/rust/issues/69153#issuecomment-589504301
[fx2default]: https://github.com/rust-lang/rust/issues/69153#issuecomment-589338446

`FxHashSet` や `FxHashMap` のような代替案を一般に採用することを決めた場合、ある場所で `HashSet` や `HashMap` を誤って使ってしまうということが容易に起こり得ます。[`clippy` を使用する]とこの問題を回避できます。

[`clippy` を使用する]: ./linting.md#型を禁止する

いくつかの型はハッシュ化を必要としません。例えば、整数型を包んだ [newtype] があり、その値はランダムまたはランダムに近いものであるとします。そのような型の場合、ハッシュ化された値の分布は値自体のそれとさほど変わらないでしょう。こういうケースでは [`nohash_hasher`] が役立つこともあります。

[newtype]: https://doc.rust-lang.org/rust-by-example/generics/new_types.html
[`nohash_hasher`]: https://crates.io/crates/nohash-hasher

ハッシュ関数の設計は複雑なトピックであり、この本の範囲外です。[`ahash` のドキュメント]には参考になる情報が載っています。

[`ahash` のドキュメント]: https://github.com/tkaitchuck/aHash/blob/master/compare/readme.md
