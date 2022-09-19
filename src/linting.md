<!-- commit: https://github.com/nnethercote/perf-book/commit/c512cbdbd3fa7e8f509272544be54b86d5f0647e -->

# Linting

([原文](https://nnethercote.github.io/perf-book/linting.html))

[Clippy] は Rust コード内のよくある間違いを捕捉する lint のコレクションです。これは、一般の Rust コードで実行するには素晴らしいツールです。また、パフォーマンスの最適化を損なう原因となるコードパターンに関する様々な lint を持っています。そのため、パフォーマンスについても手助けをしてくれます。

[Clippy]: https://github.com/rust-lang/rust-clippy

## Clippy の基本

インストールできたら、実行するのは簡単です：

```bash
cargo clippy
```

パフォーマンスに関する lint のリストは [この list][lint list] から "Perf" 以外のすべてのグループを除外することで確認できます。

[lint list]: https://rust-lang.github.io/rust-clippy/master/

コードを高速化するだけでなく、パフォーマンスに関する lint は一般によりシンプルで慣用的なコードを提案してくれます。そのため、頻繁に実行されないようなコードでもその提案に従う価値があります。

## 型を禁止する

後続の章では、標準ライブラリにある特定の型を避け高速化をもたらす代替の型を使用した方がよい場合の説明があります。しかしそれら代替の型を使用する際、誤って標準ライブラリの型をどこかで使用してしまう、というケースが発生し得るでしょう。

Rust 1.55 で Clippy に追加された [`disallowed_type`] lint を使用すれば、この問題を回避できます。例えば、標準のハッシュテーブルの使用を禁止する場合（理由については[ハッシュ化]の章で説明されています）、`clippy.toml` ファイルを用意し、以下の行を追記してください：

```toml
disallowed-types = ["std::collections::HashMap", "std::collections::HashSet"]
```

[ハッシュ化]: ./hashing.md
[`disallowed_type`]: https://rust-lang.github.io/rust-clippy/master/index.html#disallowed_type

そして、Rust コード上で以下を宣言します：

```rust,no_run
#![warn(clippy::disallowed_type)]
```

執筆時点で `disallowed_type` が "nursery"（開発中）というグループにあるため上記の手順が必要となっています。グループは将来変更される可能性があります。
