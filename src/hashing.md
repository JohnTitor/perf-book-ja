<!-- commit: https://github.com/nnethercote/perf-book/commit/d0ba7e92f194dfcdc374eb32801317b918a90d22 -->

# ハッシュ化

`HashSet` と `HashMap` は広く使われる2つの型です。デフォルトのハッシュアルゴリズムは指定されていませんが、執筆時点では [SipHash 1-3] と呼ばれるアルゴリズムがデフォルトとなっています。このアルゴリズムは高い品質を持っており、衝突に対する高い保護も提供しますが、その一方で、特に整数のような短いキーについては、遅いという特徴があります。

[SipHash 1-3]: https://en.wikipedia.org/wiki/SipHash

もしプロファイリングによりハッシュ化がホットであると判明し、[HashDoS attacks] がアプリケーションにおいて懸念にならない場合には、より高速なハッシュアルゴリズムを持つハッシュテーブルの使用により、大きな速度の改善を行うことができます:

- [`rustc-hash`] は `HashSet` や `HashMap` の代替となる `FxHashSet` や `FxHashMap` を提供しています。ハッシュアルゴリズムは低品質ですが、特に整数キーの場合にはとても高速で、rustc 内の他のどのハッシュアルゴリズムよりも性能が優れています ([`fxhash`] は同じアルゴリズムと型の実装を持ちますが、`rustc-hash` に比べると古くあまりメンテナンスされていません)。
- [`fnv`] は `FnvHashSet` 及び `FnvHashMap` 型を提供しています。ハッシュアルゴリズムは `fxhash` よりも高品質ですが、その分速度はやや劣ります。
- [`ahash`] は `AHashSet` 及び `AHashMap` を提供しています。`ahash` のハッシュアルゴリズムはいくつかのプロセッサで利用可能な AES 命令をサポートしています。

[HashDoS attacks]: https://en.wikipedia.org/wiki/Collision_attack
[`rustc-hash`]: https://crates.io/crates/rustc-hash
[`fxhash`]: https://crates.io/crates/fxhash
[`fnv`]: https://crates.io/crates/fnv
[`ahash`]: https://crates.io/crates/ahash

ハッシュ化のパフォーマンスがプログラムにおいて重要であれば、これらの代替案をいくつか試す価値があるでしょう。例えば、rustc では次の結果が得られました:

- `fnv` から `fxhash` への移行は[最大6%の高速化][fnv2fx]をもたらしました。
- `fxhash` から `ahash` への移行の試みは [1-4%の速度低下][fx2a]という結果でした。
- `fxhash` からデフォルトのハッシュアルゴリズムに戻す試みは[4-84%の速度低下][fx2default]という結果でした!

[fnv2fx]: https://github.com/rust-lang/rust/pull/37229/commits/00e48affde2d349e3b3bfbd3d0f6afb5d76282a7
[fx2a]: https://github.com/rust-lang/rust/issues/69153#issuecomment-589504301
[fx2default]: https://github.com/rust-lang/rust/issues/69153#issuecomment-589338446

`FxHashSet` や `FxHashMap` のような代替案を一般に採用することを決めた場合、ある場所で `HashSet` や `HashMap` を誤って使ってしまうということが容易に起こり得ます。プロファイルに `SipHasher13` というコードの存在があることがそれを物語っています。

ハッシュ関数の設計は複雑なトピックであり、この本の範囲外です。[`ahash` のドキュメント]には参考になる情報が載っています。

[`ahash` のドキュメント]: https://github.com/tkaitchuck/aHash/blob/master/compare/readme.md
