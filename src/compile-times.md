<!-- commit: https://github.com/nnethercote/perf-book/commit/fe8f8ba8e19b79708f623421cf707e4184897c76 -->

# コンパイル時間

([原文](https://nnethercote.github.io/perf-book/compile-times.html))

この本は主に Rust プログラムのパフォーマンスを向上させることを対象としていますが、ここでは Rust プログラムのコンパイル時間を削減することを取り扱います。それは、多くの人が関心を持っているトピックであるためです。

## リンク

コンパイル時間の大部分は実際にはリンク時間です。特に小さな変更を施した後にコンパイルし直すときは顕著です。Linux や Windows 上では、デフォルトのものよりずっと高速な lld をリンカとして選択できます。

コマンドラインから lld を指定するには、ビルドコマンドの先頭に `RUSTFLAGS="-C link-arg=-fuse-ld=lld"` を付け加えます。

（複数のプロジェクトのために）[config.toml] ファイルから lld を指定するには、次の行を追記します：

```toml
[build]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
```

[config.toml]: https://doc.rust-lang.org/cargo/reference/config.html

Rust は lld の利用を完全にサポートしているわけではありませんが、Linux や Windows 上での殆どのユースケースで動作するはずです。lld の完全なサポートについてはこの [GitHub Issue] を参照してください。

[github issue]: https://github.com/rust-lang/rust/issues/39915#issuecomment-618726211

## インクリメンタルコンパイル

Rust コンパイラは[インクリメンタルコンパイル]をサポートしています。これはクレートをコンパイルし直すときの重複した作業を避けるものです。代償として、生成される実行可能ファイルの動作が少しだけ遅くなる場合があります。そのため、これはデフォルトではデバッグビルドでのみ有効になっています。リリースビルドでも同様に有効化したい場合には `Cargo.toml` に次の行を追記してください：

```toml
[profile.release]
incremental = true
```

`incremental` 設定や異なるプロファイラについて特定の設定を有効化する方法など、詳細については [Cargo のドキュメント]を参照してください。

[インクリメンタルコンパイル]: https://blog.rust-lang.org/2016/09/08/incremental.html
[cargo のドキュメント]: https://doc.rust-lang.org/cargo/reference/profiles.html#incremental

## 視覚化

Cargo はプログラムのコンパイルを視覚化する機能を持っています。`-Ztimings` を渡すことで有効化できます：

```bash
cargo +nightly build -Ztimings
```

完了すると、HTML ファイルの名前が表示されます。そのファイルの web ブラウザで開いてください。HTML ファイルはプログラムに使われている様々なクレート間での依存関係を示すガントチャートを持っています。これはクレートグラフ中にどのくらいの並行性があるかを示し、コンパイルを直列化している大きなクレート群を分割すべきかを教えてくれます。詳細なグラフの読み方については[このドキュメント][z-timings]を参照してください。

[ガントチャート]: https://en.wikipedia.org/wiki/Gantt_chart
[z-timings]: https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#timings

## LLVM IR

Rust はバックエンドに LLVM を採用しています。LLVM の実行は時としてコンパイル時間の大部分を占めることがあります。特に、Rust コンパイラのフロントエンドが[中間表現 (IR)][ir] を大量に生成し LLVM がそれを最適化するのに時間を要している場合には顕著です。

[llvm]: https://llvm.org/
[ir]: https://ja.wikipedia.org/wiki/%E4%B8%AD%E9%96%93%E8%A1%A8%E7%8F%BE

<!-- textlint-disable ja-technical-writing/arabic-kanji-numbers -->
これらの問題は [`cargo llvm-lines`] により診断できます。このコマンドはどの Rust 関数が最も LLVM IR を生成しているかを表示するものです。巨大なプログラム中で何十回、あるいは何百回とインスタンス化されるため、ジェネリックな関数が重要なものとなることが多いです。
<!-- textlint-enable ja-technical-writing/arabic-kanji-numbers -->

[`cargo llvm-lines`]: https://github.com/dtolnay/cargo-llvm-lines/

もしジェネリックな関数が中間表現を膨大にしている場合、いくつかの修正方法があります。もっともシンプルなものは関数を小さくすることです。

- [**例**](https://github.com/rust-lang/rust/pull/72166/commits/5a0ac0552e05c079f252482cfcdaab3c4b39d614)

もう1つの方法は関数のジェネリックでない部分を、一度しかインスタンス化されない個別のジェネリックでない関数に移動することです。これが可能かどうかはジェネリックな関数の実装詳細に依存します。コード中での露出を最小化するため、ジェネリックでない関数はジェネリックな関数のインナー関数として書かれることが多いです。

- [**例**](https://github.com/rust-lang/rust/pull/72013/commits/68b75033ad78d88872450a81745cacfc11e58178).

時々、[`Option::map`] や [`Result::map_err`] のようなよくあるユーティリティ関数が何度もインスタンス化されることがあります。そのような場合、同等の `match` 式に置き換えることでコンパイル時間を削減できます。

[`option::map`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.map
[`result::map_err`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.map_err

コンパイル時間における、上記のような変更の効果は通常小さいものですが、場合によっては大きな改善/改悪につながることもあります。

- [**例**](https://github.com/servo/servo/issues/26585)
