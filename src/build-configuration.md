<!-- commit: https://github.com/nnethercote/perf-book/commit/63f37c0c7e6a7363279dae1ae16ce83eed3208da -->

# ビルド設定

正しいビルド設定を行うことで、コードを変更することなく Rust プログラムのパフォーマンスを最大化することができます。

## リリースビルド

最も大切な Rust パフォーマンスについてのアドバイスはシンプルですが、[見落としがちなものです]。つまり、パフォーマンスを必要とするときにはデバッグビルドよりもリリースビルドを使うようにしてください。リリースビルドを行うには、`--release` フラグを Cargo に渡してください。

[見落としがちなものです]: https://users.rust-lang.org/t/why-my-rust-program-is-so-slow/47764/5

リリースビルドはデバッグビルドよりも **とても** 高速にプログラムを実行できます。デバッグビルドと比較して10-100倍高速化することもよくあります!

デバッグビルドはデフォルトの動作です。`cargo build` や `cargo run`、あるいは追加のフラグを渡さずに `rustc` を実行したときはデバッグビルドが行われます。デバッグビルドはデバッグには向いているのですが、最適化はなされていません。

次のような、`cargo build` 実行時の出力の最後の行について考えてみましょう:

```text
Finished dev [unoptimized + debuginfo] target(s) in 29.80s
```
`[unoptimized + debuginfo]` はデバッグビルドが行われたことを示します。コンパイルされたコードは `target/debug/` ディレクトリに置かれます。`cargo run` も同様にデバッグビルドを行います。

リリースビルドはデバッグビルドよりもコードを最適化します。また、デバッグアサーションや整数のオーバーフローチェックなどのいくつかのチェックを無効化します。リリースビルドは、`cargo build --release`、`cargo run --release`、あるいは `rustc -O` を実行することで行なえます (それ以外にも、`rustc` には `-C opt-level` のような最適化されたビルドのための複数のオプションがあります)。これはさらなる最適化のために、デバッグビルドよりもコンパイルに多くの時間がかかります。

次のような、`cargo build --release` 実行時の出力の最後の行について考えてみましょう:

```text
Finished release [optimized] target(s) in 1m 01s
```

`[optimized]` は、リリースビルドが行われたことを示します。コンパイルされたコードは `target/release/` ディレクトリに置かれます。`cargo run --release` も同様にリリースビルドを行います。

デバッグビルド (`dev` プロファイル) と リリースビルド (`release` プロファイル) のより詳細な違いについては、[Cargoのプロファイルについてのドキュメント]を参照してください。

[Cargoのプロファイルについてのドキュメント]: https://doc.rust-lang.org/cargo/reference/profiles.html

## リンク時最適化 (LTO)

リンク時最適化 (LTO) はビルド時間の増加を代償としてランタイムパフォーマンスを10-20%以上向上させる、プログラム全体に渡る最適化のテクニックです。任意の Rust プログラムでは、ランタイムとコンパイル時間のトレードオフに価値があるかを簡単に確かめることができます。

LTO を試してみる最もシンプルな方法は `Cargo.toml` に次の行を追記して、リリースビルドを行うことです:

```toml
[profile.release]
lto = true
```

これにより、依存関係にあるすべてのクレートに渡って最適化を行う、"fat" LTO を実行できます。

あるいは、`lto = "thin"` を `Cargo.toml` に追記することで "thin" LTO を行うことができます。これはビルド時間をあまり増やすことなく、"fat" LTO と大体同じように最適化を行うという、控えめな形の LTO です。

`lto` の設定と、異なるプロファイルに向けて特定の設定を有効化する方法については、[Cargo の LTO についてのドキュメント]を参照してください。

[Cargo の LTO についてのドキュメント]: https://doc.rust-lang.org/cargo/reference/profiles.html#lto

## Codegen Units

Rust コンパイラは、並列コンパイルとそれによる高速化のために、クレートを複数の [codegen unit] に分割します。しかし、これは潜在的な最適化を損なう原因になることがあります。コンパイルタイムを犠牲にして、そのような潜在的な最適化をも行ってランタイムパフォーマンスを改善したい場合は、codegen unit の数を`1`に設定することができます:

<!-- FIXME: codegen unit は訳すべき？ コード生成単位？ codegen 単位？ -->

```toml
[profile.release]
codegen-units = 1
```

- [**例**](https://likebike.com/posts/How_To_Write_Fast_Rust_Code.html#emit-asm)

[codegen unit]: https://doc.rust-lang.org/rustc/codegen-options/index.html#codegen-units

codegen unit の数はヒューリスティックであり、少なく設定することでかえってプログラムを遅くする場合もあることに注意してください。

## CPU 固有の命令を使う

もし旧式の、あるいは他の種類のプロセッサでの互換性についてあまり心配しなくてもいい場合には、[特定の CPU アーキテクチャ]固有の、最新の (そしておそらくは最速の) 命令を生成するようコンパイルに指示することができます。

[特定の CPU アーキテクチャ]: https://doc.rust-lang.org/1.41.1/rustc/codegen-options/index.html#target-cpu

例えば、rustc に `-C target-cpu=native` を渡すことで、現在使用している CPU について最適な命令を使うことができます:

```bash
$ RUSTFLAGS="-C target-cpu=native" cargo build --release
```

これは、特にコンパイラがコード内に[ベクトル化]の機会を見つけた場合には、大きな影響を与えます。

[ベクトル化]: https://ja.wikipedia.org/wiki/%E3%83%99%E3%82%AF%E3%83%88%E3%83%AB%E5%8C%96

## `panic!` 時に中断 (abort) する

パニックを捕捉したり巻き戻したりする必要がない場合には、パニック時には単に中断 (abort) するようコンパイラに指示することができます。これはバイナリサイズを削減しわずかにパフォーマンスを向上させることがあります:

```toml
[profile.release]
panic = "abort"
```

## プロファイルに基づく最適化 (PGO)

プロファイルに基づく最適化 (PGO) は、プログラムをコンパイルし、プロファイルデータを収集しながらサンプルデータをもとに実行し、プログラムの二度目のコンパイルをサポートするためにそのプロファイルデータを使用するという、コンパイルモデルです。

- [**例**](https://blog.rust-lang.org/inside-rust/2020/11/11/exploring-pgo-for-the-rust-compiler.html)

これはセットアップにある程度の労力を要する高度なテクニックですが、いくつかの状況では試す価値があります。詳細は[rustc の PGO に関するドキュメント]を参照してください。

[rustc の PGO に関するドキュメント]: https://doc.rust-lang.org/rustc/profile-guided-optimization.html
