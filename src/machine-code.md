<!-- commit: https://github.com/nnethercote/perf-book/commit/42a0631a9254d257a7c8777e739ff6eea51a3b9b -->

# マシンコード

([原文](https://nnethercote.github.io/perf-book/machine-code.html))

頻繁にアクセスされる小さなコード片がある場合には、非効率な部分がないか生成されたマシンコードを調べる価値があるでしょう。[Compiler Explorer] というウェブサイトでは、そのような調査をするための素晴らしい環境が整っています。

[compiler explorer]: https://godbolt.org/

関連して、[`core::arch`] モジュールはアーキテクチャ固有の命令へのアクセスを提供しています。その多くは SIMD 命令に関するものです。

[`core::arch`]: https://doc.rust-lang.org/core/arch/index.html

インデックス変数の範囲に対してアサーションを追加することで、ループ内の境界チェックを避けられる場合があります。これは高度なテクニックで、境界チェックが本当に取り除かれているか生成されたコードを確かめる必要があります。

- [**例 1**](https://github.com/rust-random/rand/pull/960/commits/de9dfdd86851032d942eb583d8d438e06085867b)
- [**例 2**](https://github.com/image-rs/jpeg-decoder/pull/167/files)
