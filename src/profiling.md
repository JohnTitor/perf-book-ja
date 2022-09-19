<!-- commit: https://github.com/nnethercote/perf-book/commit/91e4ea974ea4e3594a595dc9b94e70aa85e162ee -->

# プロファイリング

([原文](https://nnethercote.github.io/perf-book/profiling.html))

プログラムを最適化するときには、プログラムのどの部分がホット、つまりランタイムパフォーマンスに影響するほど頻繁に実行されていて、変更する価値があるかを見定める方法が必要です。これにはプロファイリングが最適です。

## プロファイラー

（訳注：それぞれのプロファイラーについて特徴を把握しきれておらず、訳が伝わりにくいものになっているものがあります。適宜[原文]を参照してください）

多くのプロファイラーを利用できますが、それぞれに得意・不得意があります。以下のプロファイラーは Rust プログラム上でうまく動作します：

- [perf] はハードウェアパフォーマンスカウンターを利用した、一般用途向けのプロファイラーです。[Hotspot] や [Firefox Profiler] は perf が記録したデータを閲覧するのに適しています。perf は Linux 上で動作します。
- [AMD μProf] は一般用途向けのプロファイラーです。 Windows 及び Linux 上で動作します。
- [flamegraph] はコードのプロファイルに perf または DTrace を使用し、フレームグラフの形式でその結果を表示する Cargo コマンドです。Linux および DTraceがサポートするすべてのプラットフォーム (macOS、FreeBSD、NetBSD、そしておそらく Windows) 上で動作します。
- [Cachegrind] 及び [Callgrind] はグローバル、関数ごと、あるいはソースコード行別の命令数カウントとシミュレートされたキャッシュ、そして分岐予測データを提供します。Linux といくつかの Unix システム上で動作します。
- [DHAT] はコードのどの部分が多くのアロケーションを起こしているか見つけたり、ピーク時のメモリ使用状況について把握したりすることに適しています。これはまた `memcpy` の頻繁な呼び出しを特定するためにも使われます。Linux 及びその他いくつかの Unix システム上で動作します。[dhat-rs] は、機能がやや貧弱で Rust コードに少し手を加える必要がありますが、すべてのプラットフォーム上で動作する実験的な代替クレートです。
- [heaptrack] 及び [bytehound] はヒーププロファイリングツールです。Linux 上で動作します。
- [`counts`] はアドホックなプロファイリングをサポートしています。これは `eprintln!` 文の使用と周波数ベースの後処理を組み合わせたもので、コードの一部についてドメイン固有な情報を把握するのに適しています。すべてのプラットフォームで動作します。
- [Coz] は潜在的な最適化を測定するための **簡略化された (casual)** プロファイリングを行います。[coz-rs] により Rust をサポートしています。 Linux 上で動作します。

[原文]: https://nnethercote.github.io/perf-book/profiling.html
[perf]: https://perf.wiki.kernel.org/index.php/Main_Page
[Hotspot]: https://github.com/KDAB/hotspot
[Firefox Profiler]: https://profiler.firefox.com/
[AMD μProf]: https://developer.amd.com/amd-uprof/
[flamegraph]: https://github.com/flamegraph-rs/flamegraph
[Cachegrind]: https://www.valgrind.org/docs/manual/cg-manual.html
[Callgrind]: https://www.valgrind.org/docs/manual/cl-manual.html
[DHAT]: https://www.valgrind.org/docs/manual/dh-manual.html
[dhat-rs]: https://github.com/nnethercote/dhat-rs/
[heaptrack]: https://github.com/KDE/heaptrack
[bytehound]: https://github.com/koute/bytehound
[`counts`]: https://github.com/nnethercote/counts/
[Coz]: https://github.com/plasma-umass/coz
[coz-rs]: https://github.com/plasma-umass/coz/tree/master/rust

## Debug Info

リリースビルドを効果的にプロファイルするには、ソースコード行のデバッグ情報 (debug info) を有効化しなければならないことがあります。有効化するには、以下の行を `Cargo.toml` に追記してください：

```toml
[profile.release]
debug = 1
```

`debug` 設定についての詳細な内容は [Cargo のドキュメント]を参照してください。

[Cargo のドキュメント]: https://doc.rust-lang.org/cargo/reference/profiles.html#debug

残念ながら、上記の手順を踏んでも標準ライブラリの詳細なプロファイリングデータを得ることはできません。これはリリースされている標準ライブラリはデバッグ情報を含んでいないためです。この問題を解決するには、コンパイラと標準ライブラリを自分でビルドする必要があります。以下を `config.toml` に追記して、[こちら](https://github.com/rust-lang/rust)に記載の手順に従ってください：

```toml
[rust]
debuginfo-level = 1
```

これは面倒ですが、場合によってはやってみる価値があるでしょう。

## シンボルデマングリング

<!-- textlint-disable ja-technical-writing/no-doubled-joshi -->
Rust はコンパイルされたコード中に関数名をエンコードするためのマングリングスキーマを持っています。もしプロファイラがこれに対応していない場合、出力に `_ZN3foo3barE` や `_ZN28_$u7b$$u7b$closure$u7d$$u7d$E`、`_RMCsno73SFvQKx_1cINtB0_3StrKRe616263_E` のような、`_ZN` や `_R` から始まるシンボル名が含まれる可能性があります。これらは [`rustfilt`] を使って手ずからデマングリングできます。
<!-- textlint-enable ja-technical-writing/no-doubled-joshi -->

[`rustfilt`]: https://crates.io/crates/rustfilt
