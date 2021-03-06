<!-- commit: https://github.com/nnethercote/perf-book/commit/62fa60e3ca2a18b3f4d0cf91922d83e75ed57b42 -->

# ベンチマーク

([原文](https://nnethercote.github.io/perf-book/benchmarking.html))

ベンチマークとは、同じ働きを持つ 2 つ以上のプログラムのパフォーマンスを比較するものです。これは、例えば Firefox vs. Safari vs. Chrome のように、複数の異なるプログラムを比較することを指す場合もたまにあります。また、同じプログラムの異なる 2 つのバージョンを比較することを指す場合もあります。最後のケースは「この変更は速度に影響を与えたのか？」といった質問に答えるための根拠となり得ます。

ベンチマークは複雑なトピックであり、1 から 10 まで説明することはこの本では行わず、基本的な内容についてのみ紹介していきます。

初めに、測定するための負荷 (workload) が必要です。理想的には、プログラムの実際の使われ方を模した様々な負荷を用意したいところです。実際の入力を使った負荷が最善ですが、[マイクロベンチマーク][microbenchmarks]や[ストレステスト]もある程度は役に立つでしょう。

[microbenchmarks]: https://stackoverflow.com/questions/2842695/what-is-microbenchmarking
[ストレステスト]: https://ja.wikipedia.org/wiki/%E3%82%B9%E3%83%88%E3%83%AC%E3%82%B9%E3%83%86%E3%82%B9%E3%83%88

次に、実際に負荷をかける方法が必要です。これは使用される測定基準にも影響します。Rust の組み込みの[ベンチマークテスト]はシンプルですが、不安定な機能を使っているため Nightly Rust でしか動作しません。[`bencher`] クレートはそれを stable Rust でも動作するようにしたものです（訳注：`bencher` は 2021 年 4 月現在、2018 年 1 月に公開された v0.1.5 が最新版となっており、積極的に開発されているものではないことに注意してください）。[Criterion] はそれらの代替となる、より洗練されたクレートです。また、カスタムベンチマークハーネスも利用できます。例えば、[rustc-perf] は Rust コンパイラのベンチマークに使われているハーネスです。

[ベンチマークテスト]: https://doc.rust-lang.org/1.16.0/book/benchmark-tests.html
[`bencher`]: https://crates.io/crates/bencher
[criterion]: https://github.com/bheisler/criterion.rs
[rustc-perf]: https://github.com/rust-lang/rustc-perf/

測定基準については、多くの選択肢があり、どのようなものが適切かは測定されるプログラムの性質に依存します。例えば、バッチ処理を行うプログラムに適切な測定基準は、インタラクティブなプログラムについて適切なものではないでしょう。実測時間はユーザーが体験するものと一致することがほとんどなので、明白な選択肢となります。しかし、実測時間は同時に大きなばらつきを持つものでもあります。特にメモリレイアウトの些細な変更が、一時的ではあるが影響の大きいパフォーマンス低下を起こすことがあります。そのため、サイクル数や命令数などの、ばらつきの少ない他の測定基準も合理的な代替案になり得ます。

複数の負荷の測定結果をまとめることも難しい課題です。様々な方法がありますが、これが絶対的に正解！という方法はありません。

良いベンチマークというのは難しいものです。そう言いつつ、特にプログラムの最適化を始めるときに、完璧なベンチマークをセットアップすることについてあまり気負うことはありません。普通のベンチマークでも何もしないよりは遥かに役に立ちます。また、測定しているものやその結果について先入観を持たないようにしてください。そして、プログラムのパフォーマンス特性について理解することで、時間をかけてベンチマーク結果を改善していくことができます。
