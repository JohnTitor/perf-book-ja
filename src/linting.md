<!-- commit: https://github.com/nnethercote/perf-book/commit/9ad40f937b6cedf261ebb2fde38dba662f4761dc -->

# Linting

([原文](https://nnethercote.github.io/perf-book/linting.html))

[Clippy] は Rust コード内のよくある間違いを捕捉する lint のコレクションです。これは、一般の Rust コードで実行するには素晴らしいツールです。また、パフォーマンスの最適化を損なう原因となるコードパターンに関する様々な lint を持っています。そのため、パフォーマンスについても手助けをしてくれます。

[Clippy]: https://github.com/rust-lang/rust-clippy

インストールできたら、実行するのは簡単です:

```bash
cargo clippy
```

パフォーマンスに関する lint のリストは [この list][lint list] から "Perf" 以外のすべてのグループを除外することで確認できます。

[lint list]: https://rust-lang.github.io/rust-clippy/master/

コードを高速化するだけでなく、パフォーマンスに関する lint は一般によりシンプルで慣用的なコードを提案してくれます。そのため、頻繁に実行されないようなコードでもその提案に従う価値があります。
