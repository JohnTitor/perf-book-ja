<!-- commit: https://github.com/nnethercote/perf-book/commit/19db3a765030ed7c394a987eff5c09f639f0607d -->

# ログとデバッグ

時々、ログやデバッグのためのコードがプログラムの速度を著しく低下させることがあります。ログやデバッグのためのコード自体が遅いこともあれば、データをそのようなコードに送るためのデータコレクションコードが遅いこともあります。ログやデバッグを行わない場合には、そのような目的のためのコードが不必要に使われていないかを確かめてください。

- [**Example 1**](https://github.com/rust-lang/rust/pull/50246/commits/2e4f66a86f7baa5644d18bb2adc07a8cd1c7409d)
- [**Example 2**](https://github.com/rust-lang/rust/pull/75133/commits/eeb4b83289e09956e0dda174047729ca87c709fe)

[`assert!`] は常に実行されますが、[`debug_assert!`] はデバッグビルド時にのみ実行されることを覚えておいてください。頻繁に呼び出されるが安全性のために必要なわけではないアサーションについては、`debug_assert!` の使用を検討してください。

- [**Example**](https://github.com/rust-lang/rust/pull/58210/commits/f7ed6e18160bc8fccf27a73c05f3935c9e8f672e)

[`assert!`]: https://doc.rust-lang.org/std/macro.assert.html
[`debug_assert!`]: https://doc.rust-lang.org/std/macro.debug_assert.html
