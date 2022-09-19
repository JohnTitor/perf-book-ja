<!-- commit: https://github.com/nnethercote/perf-book/commit/60855e5d0007748ad316e17acd66171d9eb991fb -->

# ログとデバッグ

([原文](https://nnethercote.github.io/perf-book/logging-and-debugging.html))

時々、ログやデバッグのためのコードがプログラムの速度を著しく低下させることがあります。ログやデバッグのためのコード自体が遅いこともあれば、データをそのようなコードに送るためのデータコレクションコードが遅いこともあります。ログやデバッグを行わない場合には、そのような目的のためのコードが不必要に使われていないかを確かめてください。

- [**例 1**](https://github.com/rust-lang/rust/pull/50246/commits/2e4f66a86f7baa5644d18bb2adc07a8cd1c7409d)
- [**例 2**](https://github.com/rust-lang/rust/pull/75133/commits/eeb4b83289e09956e0dda174047729ca87c709fe)

[`assert!`] は常に実行されますが、[`debug_assert!`] はデバッグビルド時にのみ実行されることを覚えておいてください。頻繁に呼び出されるが安全性のために必要なわけではないアサーションについては、`debug_assert!` の使用を検討してください。

- [**例 1**](https://github.com/rust-lang/rust/pull/58210/commits/f7ed6e18160bc8fccf27a73c05f3935c9e8f672e)
- [**例 2**](https://github.com/rust-lang/rust/pull/90746/commits/580d357b5adef605fc731d295ca53ab8532e26fb)

[`assert!`]: https://doc.rust-lang.org/std/macro.assert.html
[`debug_assert!`]: https://doc.rust-lang.org/std/macro.debug_assert.html
