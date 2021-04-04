<!-- https://github.com/nnethercote/perf-book/commit/734f315e48e5f2d72777fc22f56b5404e2ece12e -->

# ヒープ割り当て

ヒープ割り当て (heap allocation) には中程度にコストがかかります。その詳細は使用するアロケータに依存しますが、それぞれの割り当て (そして割り当て解除) はグローバルロックの取得、いくつかの些細でないデータ構造の操作、そして (場合によっては) システムコールの実行を行います。小さな割り当ては大きなものよりコストがかからないとは限りません。割り当てを避けることは大きくパフォーマンスを改善できることがあるため、どの Rust のデータ構造と命令が割り当てを引き起こすのか理解する価値があります。

[Rust Container チートシート]は一般的な Rust の型を視覚化しており、続くセクションの理解を助けてくれます。

[rust container チートシート]: https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/

## プロファイリング

一般用途向けのプロファイラーが `malloc` や `free`、その他の関連する関数がホットであると示した場合には、割り当ての割合を減らし、代替となるアロケータを利用することを試してみる価値があることが多いでしょう。

[DHAT] は割り当ての割合を減らすときに使える素晴らしいプロファイラーです。Linux やその他の Unix システム上で動作します。ホットな割り当ての場所とその割合をかなり正確に特定してくれます。実際の結果は測定されたものと異なるでしょうが、rustc では、実行される 100 万命令あたり 10 の割り当てを減らすことで観測できるレベルのパフォーマンス向上 (~1%) が見られました。

[dhat]: https://www.valgrind.org/docs/manual/dh-manual.html

これは例となる DHAT の出力です:

```text
AP 1.1/25 (2 children) {
  Total:     54,533,440 bytes (4.02%, 2,714.28/Minstr) in 458,839 blocks (7.72%, 22.84/Minstr), avg size 118.85 bytes, avg lifetime 1,127,259,403.64 instrs (5.61% of program duration)
  At t-gmax: 0 bytes (0%) in 0 blocks (0%), avg size 0 bytes
  At t-end:  0 bytes (0%) in 0 blocks (0%), avg size 0 bytes
  Reads:     15,993,012 bytes (0.29%, 796.02/Minstr), 0.29/byte
  Writes:    20,974,752 bytes (1.03%, 1,043.97/Minstr), 0.38/byte
  Allocated at {
    #1: 0x95CACC9: alloc (alloc.rs:72)
    #2: 0x95CACC9: alloc (alloc.rs:148)
    #3: 0x95CACC9: reserve_internal<syntax::tokenstream::TokenStream,alloc::alloc::Global> (raw_vec.rs:669)
    #4: 0x95CACC9: reserve<syntax::tokenstream::TokenStream,alloc::alloc::Global> (raw_vec.rs:492)
    #5: 0x95CACC9: reserve<syntax::tokenstream::TokenStream> (vec.rs:460)
    #6: 0x95CACC9: push<syntax::tokenstream::TokenStream> (vec.rs:989)
    #7: 0x95CACC9: parse_token_trees_until_close_delim (tokentrees.rs:27)
    #8: 0x95CACC9: syntax::parse::lexer::tokentrees::<impl syntax::parse::lexer::StringReader<'a>>::parse_token_tree (tokentrees.rs:81)
  }
}
```

この例のすべてを説明することはこの本の範囲外ですが、DHAT が割り当てがどこで・どのくらいの頻度で起こっているか、どのくらいの大きさか、どのくらいの間有効なのか、そしてどのくらいの頻度でアクセスされるのかなど、割り当てについてたくさんの情報を提供してくれていることが分かるでしょう。

## `Box`

[`Box`] は最もシンプルなヒープに置かれる (heap-allocated) 型です。`Box<T>` の値はヒープに置かれた `T` の値です。

[`box`]: https://doc.rust-lang.org/std/boxed/struct.Box.html

`struct` や `enum` の 1 つあるいはそれ以上のフィールドを box 化することで、型を小さくできる場合があります (この詳細については [型のサイズ] というチャプターを参照してください)。

[型のサイズ]: type-sizes.md

それ以外では `Box` は簡潔で最適化の余地はあまりありません。

## `Rc`/`Arc`

[`Rc`]/[`Arc`] は `Box` と似ていますが、ヒープ上の値は 2 つの参照カウントを持っています。これらの型は値の共有を可能にし、メモリ使用量を削減するための効果的な手段になり得ます。

[`rc`]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[`arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html

しかし、めったに共有されないような値に使われると、ヒープに置かれないような値までヒープに置くことで割り当ての割合を大きくしてしまう恐れがあります。

- [**例**](https://github.com/rust-lang/rust/pull/37373/commits/c440a7ae654fb641e68a9ee53b03bf3f7133c2fe)

`Box` と異なり、`Rc`/`Arc` 上で `clone` を呼び出しても割り当ては行われません。代わりに、参照カウントをインクリメントします。

## `Vec`

[`Vec`] はヒープに置かれる型で、割り当ての数を最適化したり、無駄なスペースの量を最小化したりする大きな余地があります。このためにはその要素がどのように保持されるかについて理解しなければなりません。

[`vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html

`Vec` には 3 つの要素があります。長さ、容量、そしてポインタです。ポインタは容量と要素の大きさが 0 でない場合にはヒープに置かれたメモリを指します。そうでない場合は、割り当てられたメモリを指すことはありません。

`Vec` 自体がヒープに置かれないとしても、その要素 (存在していてサイズが 0 でない場合) はいつもヒープに置かれます。もしサイズが 0 でない要素が存在する場合、それらの要素を保持するメモリは、追加の要素のためのスペースを開けておくことにより、必要より大きくなっている場合があります。現在ある要素の数が長さとなり、再度割り当てることなく要素を置ける数というのが容量になります。

ベクタが現在の容量を増やす必要が出てきた場合、その要素はより大きなヒープ割り当てにコピーされ、古いヒープ割り当ては解放されます。

### `Vec` の増大

一般的な方法で ([`vec![]`][vec macro]、[`Vec::new`] あるいは [`Vec::default`]) 作られた新しい空の `Vec` の長さと容量は 0 であり、ヒープ割り当てを必要としません。もし個々の要素をその `Vec` の最後に繰り返し追加した場合、定期的に再割り当てが発生します。増大させる方法 (strategy) は指定されていませんが、執筆時では quasi-doubling strategy が採用されています。この方法では 0、4、8、16、32、64…、のように容量が遷移していきます ([多くの割り当てを避ける]ため、これは 1 と 2 を飛ばして 0 から 4 へと容量を増やします)。ベクタが増大するにつれ、再割り当ての頻度は指数関数的に減っていきますが、無駄になる可能性のある余分な容量は指数関数的に増えていきます。

[vec macro]: https://doc.rust-lang.org/std/macro.vec.html
[`vec::new`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.new
[`vec::default`]: https://doc.rust-lang.org/std/default/trait.Default.html#tymethod.default
[多くの割り当てを避ける]: https://github.com/rust-lang/rust/pull/72227

この増大の仕組みは要素を増やしていけるデータ構造では一般的で、一般用途では理にかなっているものですが、事前にベクタの長さが分かっている場合には、より効率的なことを行えます。ホットなベクタの割り当て場所 (例えばホットな [`Vec::push`] の呼び出しなど) がある場合は、そこでベクタの長さを出力するために [`eprintln!`] を使い、その後長さの分布を見定めるために ([`counts`] などを使って) 後処理をする価値があります。例えば、たくさんの短いベクタがあったり、それよりすくない数のとても長いベクタがあったりするでしょう。割り当て場所を最適化する一番良い方法はその状況に大きく依存します。

[`vec::push`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.push
[`eprintln!`]: https://doc.rust-lang.org/std/macro.eprintln.html
[`counts`]: https://github.com/nnethercote/counts/

### 短い `Vec`

もし短いベクタがたくさんある場合には、[`smallvec`] クレートの `SmallVec` 型を使うことができます。`SmallVec<[T; N]>` は `Vec` の代替であり、`N` 個の要素を `SmallVec` 自身に保持できます。要素数がそれを越えた場合はヒープ割り当てに切り替わります (`SmallVec` を使う場合、`vec![]` を `smallvec![]` に置き換える必要があることに注意してください)。

- [**例 1**](https://github.com/rust-lang/rust/pull/50565/commits/78262e700dc6a7b57e376742f344e80115d2d3f2)
- [**例 2**](https://github.com/rust-lang/rust/pull/55383/commits/526dc1421b48e3ee8357d58d997e7a0f4bb26915)

[`smallvec`]: https://crates.io/crates/smallvec

`SmallVec` は適切に使えば割り当ての割合を確実に下げますが、その使用自体がパフォーマンスの向上を保証するものではありません。それは要素がヒープに置かれているかどうかを必ず確認するため、一般的な処理では `Vec` よりもわずかに遅いです。また、`N` や `T` が大きい場合、`SmallVec<[T; N]>` 自身は `Vec<T>` よりも大きくなることがあり、`SmallVec` のコピーは `Vec` より遅くなります。いつも通り、最適化に効果があるのか確認するためのベンチマークが必須です。

短いベクタがたくさんあり **加えて** その最大長を正確に把握している場合、[`arrayvec`] の `ArrayVec` 型は `SmallVec` よりも優れた選択肢です。ヒープ割り当てへのフォールバックを必要とせず、`SmallVec` よりもわずかに高速に動作します。

- [**例**](https://github.com/rust-lang/rust/pull/74310/commits/c492ca40a288d8a85353ba112c4d38fe87ef453e)

[`arrayvec`]: https://crates.io/crates/arrayvec

### 長い `Vec`

ベクタの最小の、あるいは実際の大きさをしっている場合、[`Vec::with_capacity`]、[`Vec::reserve`]、あるいは [`Vec::reserve_exact`] を使って特定の容量を確保できます。例えば、あるベクタが少なくとも 20 個の要素を持つことが分かっている場合、これらの関数は直ちに一回のの割り当てで少なくとも 20 の容量を持つベクタを生成します。反対に一度に 1 つずつアイテムを挿入した場合には 4 回の割り当てを行うことになります (容量について、4、8、16、そして 32)。

- [**例**](https://github.com/rust-lang/rust/pull/77990/commits/a7f2bb634308a5f05f2af716482b67ba43701681)

[`vec::with_capacity`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.with_capacity
[`vec::reserve`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.reserve
[`vec::reserve_exact`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.reserve_exact

ベクタの最大長を知っている場合、上記の関数は不必要に余分なスペースを割り当てないでいいようにしてくれます。同様に、[`Vec::shrink_to_fit`] は余分なスペースを最小化するために使えますが、再割り当てが必要となる場合があることに注意してください。

[`vec::shrink_to_fit`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.shrink_to_fit

## `String`

[`String`] はヒープに置かれるバイト列を持ちます。`String` の表現と操作は `Vec<u8>` によく似ています。増大と容量に関する多くの `Vec` メソッドと同等なものが `String` にもあります。例えば [`String::with_capacity`] などです。

[`string`]: https://doc.rust-lang.org/std/string/struct.String.html
[`string::with_capacity`]: https://doc.rust-lang.org/std/string/struct.String.html#method.with_capacity

[`smallstr`] クレートの `SmallString` 型は `SmallVec` 型に似ています。

[`smallstr`]: https://crates.io/crates/smallstr

[`smartstring`] クレートの `String` 型は std の `String` の代替であり、3 語分以下の文字列についてヒープ割り当てを回避します。64-bit プラットフォーム上では、これは、23 以下の ASCII 文字を合わせたすべての文字列を含む、24 bytes 以下の任意の文字列となります。

- [**例**](https://github.com/djc/topfew-rs/commit/803fd566e9b889b7ba452a2a294a3e4df76e6c4c)

[`smartstring`]: https://crates.io/crates/smartstring

`format!` マクロは `String` を生成する、つまり、割り当てを必要とすることに注意してください。文字列リテラルを使うことで `format!` の呼び出しを避けられる場合、この割り当てを回避できます。

- [**例**](https://github.com/rust-lang/rust/pull/55905/commits/c6862992d947331cd6556f765f6efbde0a709cf9)

## ハッシュテーブル

[`HashSet`] や [`HashMap`] はハッシュテーブルです。割り当てという文脈では、それらの表現や操作は `Vec` のそれに似ています。それらはキーや値を保持しつつ単一の継続したヒープ割り当てを行い、テーブルの拡大に必要となる限り再割り当てされます。増大と容量に関する多くの `Vec` のメソッドと同様のものが `HashSet`/`HashMap` にもあります。例えば、[`HashSet::with_capacity`] などです。

[`hashset`]: https://doc.rust-lang.org/std/collections/struct.HashSet.html
[`hashmap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[`hashset::with_capacity`]: https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.with_capacity

## `Cow`

時々、`&str` のような、何らかの借用されたデータを使いたい場合があります。これは殆どの場合読み取り専用ですが、たまに変更する必要が出てきます。毎回そのデータをクローンするのは無駄が多いです。その代わりに[`Cow`] 型を通して、借用/所有された両方のデータを表現できる、"clone-on-write" なセマンティクスを使うことができます。

[`cow`]: https://doc.rust-lang.org/std/borrow/enum.Cow.html

通常、借用された値 `x` から始めるときは、`Cow::Borrowed(x)` を使って `Cow` の中に `x` を包みます。`Cow` は [`Deref`] を実装しているので、`Cow` が持っているデータに対して不変 (non-mutating) なメソッドを直接呼び出せます。もし可変である必要がある場合には、[`Cow::to_mut`] によって、必要に応じてクローンしつつ、所有されている値への可変参照を取得できます。

[`deref`]: https://doc.rust-lang.org/std/ops/trait.Deref.html
[`cow::to_mut`]: https://doc.rust-lang.org/std/borrow/enum.Cow.html#method.to_mut

`Cow` をうまく動かすのは難しいかもしれませんが、試行錯誤する価値は十分あります。

- [**例 1**](https://github.com/rust-lang/rust/pull/37064/commits/b043e11de2eb2c60f7bfec5e15960f537b229e20)
- [**例 2**](https://github.com/rust-lang/rust/pull/50855/commits/ad471452ba6fbbf91ad566dc4bdf1033a7281811)
- [**例 3**](https://github.com/rust-lang/rust/pull/56336/commits/787959c20d062d396b97a5566e0a766d963af022)
- [**例 4**](https://github.com/rust-lang/rust/pull/68848/commits/67da45f5084f98eeb20cc6022d68788510dc832a)

## `clone`

ヒープに置かれたメモリを含む値に対して [`clone`] を呼び出すと、通常は追加の割り当てが発生します。例えば、空でない `Vec` に対して `clone` を呼び出すと、その要素の新しい割り当てが必要になります (ですが、新しい `Vec` の容量は元のものと異なる可能性があることに注意してください)。例外は `Rc`/`Arc` で、`clone` 呼び出しは参照カウントをインクリメントするだけです。

[`clone`]: https://doc.rust-lang.org/std/clone/trait.Clone.html#tymethod.clone

[`clone_from`] は `clone` の代替です。`a.clone_from(&b)` は `a = b.clone()` と同等ですが、不必要な割り当てを避ける場合があります。例えば既存の `Vec` をもとに `Vec` を一つクローンしたい場合、可能であれば既存の `Vec` のヒープ割り当てが再利用されます。以下の例で示します:

```rust
let mut v1: Vec<u32> = Vec::with_capacity(99);
let v2: Vec<u32> = vec![1, 2, 3];
v1.clone_from(&v2); // v1's allocation is reused
assert_eq!(v1.capacity(), 99);
```

`clone` は通常割り当てを発生させますが、多くの状況では利用することは理にかなっておりコードをシンプルにすることも多いです。プロファイリングデータを使ってどの `clone` の呼び出しがホットで避けるために工夫する価値があるか確認してください。

[`clone_from`]: https://doc.rust-lang.org/std/clone/trait.Clone.html#method.clone_from

時々、Rust コードには、(a) プログラマーのミス、(b) コードの変更により以前必要だった `clone` 呼び出しが不必要になった、などの理由で、不必要な `clone` 呼び出しが含まれることがあります。もし必要そうでないホットな `clone` の呼び出しを見つけた場合には、単純に削除できることもあります。

- [**例 1**](https://github.com/rust-lang/rust/pull/37318/commits/e382267cfb9133ef12d59b66a2935ee45b546a61)
- [**例 2**](https://github.com/rust-lang/rust/pull/37705/commits/11c1126688bab32f76dbe1a973906c7586da143f)
- [**例 3**](https://github.com/rust-lang/rust/pull/64302/commits/36b37e22de92b584b9cf4464ed1d4ad317b798be)

## `to_owned`

[`ToOwned::to_owned`] は多くの一般的な型に対して実装されています。これは借用されたデータから所有されたデータを作成します。一般的にはクローンが必要で、それによりしばしばヒープ割り当てが発生します。例えば、これは `&str` から `String` を作成するときに使われます。

[`toowned::to_owned`]: https://doc.rust-lang.org/std/borrow/trait.ToOwned.html#tymethod.to_owned

時々、所有されたコピーでなく構造体にある借用されたデータへの参照を保持することで、`to_owned` 呼び出しを回避できることがあります。これは構造体にライフタイム注釈を必要とし、コードを複雑にします。そのため、プロファイリングやベンチマークでそうする価値があると分かった場合にのみ適用されるべきです。

- [**例**](https://github.com/rust-lang/rust/pull/50855/commits/6872377357dbbf373cfd2aae352cb74cfcc66f34)

## コレクションを再利用する

`Vec` のようなコレクションを段階的に作成しなければならない場合があります。この場合、複数の `Vec` をつくってそれらを組み合わせるよりも、一つの `Vec` を変更していく方が、一般により良いやり方です。

例えば、複数回呼び出される可能性のある `Vec` を生成する `do_stuff` という関数があります:

```rust
fn do_stuff(x: u32, y: u32) -> Vec<u32> {
    vec![x, y]
}
```

この時、渡された `Vec` を変更した方が良いことがあります:

```rust
fn do_stuff(x: u32, y: u32, vec: &mut Vec<u32>) {
    vec.push(x);
    vec.push(y);
}
```

再利用できる "workhorse" なコレクションを保持しておく価値がある場合もあります。例えば、ループの各イテレーションのために `Vec` が必要な場合、ループの外で `Vec` を宣言し、ループのボディ内でそれを使用し、それからループボディの最後に [`clear`] (容量を変えずに `Vec` を空にする) を呼び出すことができるでしょう。これは、各イテレーションでの `Vec` の使用がその他のイテレーションに関係がないということが分かりづらくなることと引き換えに、割り当てを避けることができます。

- [**例 1**](https://github.com/rust-lang/rust/pull/77990/commits/45faeb43aecdc98c9e3f2b24edf2ecc71f39d323)
- [**例 2**](https://github.com/rust-lang/rust/pull/51870/commits/b0c78120e3ecae5f4043781f7a3f79e2277293e7)

[`clear`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.clear

同様に、構造体の中に "workhorse" なコレクションを保持しておく価値がある場合もあります。これは繰り返し呼び出される 1 つ以上のメソッドの中で再利用されます。

## 代替となるアロケータを使用する

割り当ての多い Rust プログラムのパフォーマンスを向上させる別の手段として、デフォルトの (システム) アロケータを代替のものに置き換えるというものがあります。正確な影響は個々のプログラムと選ばれるアロケータに依存します。また、各プラットフォームのシステムアロケータはそれぞれ長所短所を持っているので、プラットフォームによってもその影響は異なります。異なるアロケータの使用はバイナリサイズにも影響します。

人気のある代替アロケータの 1 つに [`jemallocator`] クレートを通して使える [jemalloc] があります。使用するには依存関係を `Cargo.toml` に追記します:

```toml
[dependencies]
jemallocator = "0.3.2"
```

それから次の行を Rust コードのどこかに追記します:

```rust,ignore
#[global_allocator]
static GLOBAL: jemallocator::Jemalloc = jemallocator::Jemalloc;
```

もう 1 つの代替となるアロケータは [`mimalloc`] を通して使える [mimalloc] です。

[jemalloc]: https://github.com/jemalloc/jemalloc
[`jemallocator`]: https://crates.io/crates/jemallocator
[mimalloc]: https://github.com/microsoft/mimalloc
[`mimalloc`]: https://docs.rs/mimalloc/0.1.22/mimalloc/
