<!-- commit: https://github.com/nnethercote/perf-book/commit/b6d4c85f8fa59def307b835da94f058b1c1fd7c6 -->

# イテレータ

## `collect` と `extend`

[`Iterator::collect`] はイテレータを `Vec` のようなコレクション型に変換します。これは通常割り当てを必要とします。もしコレクション型がその後再びイテレートされるだけなのであれば、`collect` の呼び出しは避けるべきです。

[`iterator::collect`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect

上記の理由から、`Vec<T>` よりも `impl Iterator<Item=T>` といったイテレータ型を関数から返した方が良い場合が多いです。ただ、[この記事]で説明されているように、そういった返り値には追加のライフタイム注釈が必要となる場合があることに注意してください。

- [**例**](https://github.com/rust-lang/rust/pull/77990/commits/660d8a6550a126797aa66a417137e39a5639451b)

[この記事]: https://blog.katona.me/2019/12/29/Rust-Lifetimes-and-Iterators/

同様に、イテレータを `Vec` にして [`append`] を使うよりも、イテレータを使って既存のコレクション型 (`Vec` など) を拡張するために [`extend`] を使った方が良いでしょう。

[`extend`]: https://doc.rust-lang.org/std/iter/trait.Extend.html#tymethod.extend
[`append`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.append

最後に、イテレータを書くときは、可能であれば [`Iterator::size_hint`] か [`ExactSizeIterator::len`] メソッドを実装した方がいい場合が多いです。その理由として、イテレータによって生成される要素数についての事前情報が手に入るため、イテレータを使う `collect` や `extend` といった呼び出しにおいてより小さな割り当てを行える、というものがあります。

[`iterator::size_hint`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.size_hint
[`exactsizeiterator::len`]: https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html#method.len

## チェイン

[`chain`] はとても便利ですが、単一のイテレータより遅くなることがあります。可能であればホットなイテレータに対しては使用を避けるべきでしょう。

- [**例**](https://github.com/rust-lang/rust/pull/64801/commits/5ca99b750e455e9b5e13e83d0d7886486231e48a)

同様に、[`map`] に続けて [`filter`] を使うよりも、[`filter_map`] を使用した方が高速になることがあります。

[`chain`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain
[`filter_map`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map
[`filter`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter
[`map`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map
