<!-- commit: https://github.com/nnethercote/perf-book/commit/e4a66445b16293eaf691435eca099fc31e348ecb -->

# 並列処理

Rustは安全な並列プログラミングのために素晴らしいサポートを提供しています。そのような並列処理は大きなパフォーマンス向上をもたらすことがあります。並列処理をプログラムに実装するには様々な方法がありますが、任意のプログラムについての最適な方法というのはそのプログラムの設計に大きく依存します。

並列処理についての詳細な説明はこの本の範囲外です。もしこのトピックに興味があれば、[`rayon`] や [`crossbeam`] のドキュメントは良い開始点となるでしょう。

[`rayon`]: https://crates.io/crates/rayon
[`crossbeam`]: https://crates.io/crates/crossbeam
