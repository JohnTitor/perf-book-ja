<!-- commit: https://github.com/nnethercote/perf-book/commit/bfb0a69ea4a0d3057e1fb3dac03dee249f6f208d -->

# 汎用的なアドバイス

([原文](https://nnethercote.github.io/perf-book/general-tips.html))

この本のこれより前のセクションでは Rust 固有のテクニックについて解説してきました。このセクションでは一般的なパフォーマンスの基礎事項の簡潔な概要をいくつか紹介していきます。

## Rust 自体のパフォーマンス

[リリースビルドを使わない]といったような明白な落とし穴を避ければ、Rust は一般的に良いパフォーマンスを発揮します。ときに、Python や Ruby のような動的型付け言語に慣れている場合にはこれが分かりやすいでしょう。

[リリースビルドを使わない]: build-configuration.md

## 最適化する部分を見極める

最適化されたコードはそうでないコードよりも複雑で書くのに労力を要することも多いです。このような理由からホットなコードのみを最適化していく方が良いでしょう。

最も大きなパフォーマンス改善というのは、低レベルでの最適化よりもアルゴリズムやデータ構造の変更などによってもたらされることが多いものです。

- [**例 1**](https://github.com/rust-lang/rust/pull/53383/commits/5745597e6195fe0591737f242d02350001b6c590)
- [**例 2**](https://github.com/rust-lang/rust/pull/54318/commits/154be2c98cf348de080ce951df3f73649e8bb1a6)

## 現代的なハードウェア上での最適化

現代的なハードウェアでうまく動作するコードを書くというのは常に簡単というわけではありませんが、やってみる価値はあります。例えば、キャッシュミスや分岐予測の失敗を出来る部分で最小化してみてください。

## 小さな最適化を積み重ねる

ほとんどの最適化は小さなスピード改善という結果になりがちです。単一の改善では目を引くものでなくとも、それを十分な数積み重ねていけば大きなものになります。

## いろんなプロファイラーを使ってみる

プロファイラーにはそれぞれ長所があります。複数使ってみるのが良いでしょう。

## ホットな関数の最適化

もしプロファイリングで関数がホットであると分かった場合には、スピードを改善できる2つの一般的なやり方があります。

- (a): 関数を高速化する
- (b): 可能な限り呼び出しを避ける

## 愚直で遅い実装を潰していく

賢いスピードアップを図るよりも愚直な実装で遅い部分を潰していく方が多くの場合簡単です。

## 必要なときだけ評価する

必要でない限り何かを計算するのは避けましょう。遅延評価/オンデマンド評価 (on-demand computations) は改善につながることが多いです。

- [**例 1**](https://github.com/rust-lang/rust/pull/36592/commits/80a44779f7a211e075da9ed0ff2763afa00f43dc)
- [**例 2**](https://github.com/rust-lang/rust/pull/50339/commits/989815d5670826078d9984a3515eeb68235a4687)

## 特殊・固有のケースについて処理する

複雑で一般的な実装は、共通する特別なケースに対する、よりシンプルなチェックにより避けられることが多いです。

Complex general cases can often be avoided by optimistically checking for
common special cases that are simpler.

- [**例 1**](https://github.com/rust-lang/rust/pull/68790/commits/d62b6f204733d255a3e943388ba99f14b053bf4a)
- [**例 2**](https://github.com/rust-lang/rust/pull/53733/commits/130e55665f8c9f078dec67a3e92467853f400250)
- [**例 3**](https://github.com/rust-lang/rust/pull/65260/commits/59e41edcc15ed07de604c61876ea091900f73649)

特に、小さなサイズが優位を占める場合、0～2個の要素を持つコレクションの処理はパフォーマンス改善につながることが多いです。

- [**例 1**](https://github.com/rust-lang/rust/pull/50932/commits/2ff632484cd8c2e3b123fbf52d9dd39b54a94505)
- [**例 2**](https://github.com/rust-lang/rust/pull/64627/commits/acf7d4dcdba4046917c61aab141c1dec25669ce9)
- [**例 3**](https://github.com/rust-lang/rust/pull/64949/commits/14192607d38f5501c75abea7a4a0e46349df5b5f)
- [**例 4**](https://github.com/rust-lang/rust/pull/64949/commits/d1a7bb36ad0a5932384eac03d3fb834efc0317e5)

同様に、繰り返しのあるデータを処理する場合、共通の値についてコンパクトな表現を用いてまれな値についてはサブテーブルへとフォールバックするような実装をすることで、単純な形式のデータ圧縮を使用できます。

- [**例 1**](https://github.com/rust-lang/rust/pull/54420/commits/b2f25e3c38ff29eebe6c8ce69b8c69243faa440d)
- [**例 2**](https://github.com/rust-lang/rust/pull/59693/commits/fd7f605365b27bfdd3cd6763124e81bddd61dd28)
- [**例 3**](https://github.com/rust-lang/rust/pull/65750/commits/eea6f23a0ed67fd8c6b8e1b02cda3628fee56b2f)

## 最もよくあるケースを優先する

コードが複数のケースに対処している場合、それぞれのケースの頻度を計測し、最もよくあるものを一番最初に処理しましょう。

局所性の高い探索 (lookup) を処理する場合、データ構造の先頭に小さなキャッシュを用意するとパフォーマンス改善につながることがあります。

## コメントを書く

最適化されたコードは明白でない構造をしていることも多く、それを説明する、特にプロファイリングの測定結果を示すようなコメントを書くことが重要になります。例えば、「このベクタは99%の割合で0か1を要素として持つのでそれらのケースを先に処理してください」といったコメントは役に立つでしょう。
