<!-- commit: https://github.com/nnethercote/perf-book/commit/7cdbb476e0c1b40db89f394a40caafe65688933e -->

# 型のサイズ

しばしばインスタンス化される型を縮小する (shrink) ことは、パフォーマンスの向上を助けます。

例えば、もしメモリ使用量が高い場合、[DHAT] や [heaptrack] のようなヒーププロファイラーを使うと、ホットな割り当て位置 (allocation points) とそれに付随する型を特定できます。それらの型を縮小することはピーク時のメモリ使用量を減らすことができ、またメモリへの頻繁なアクセスやキャッシュへの負担を減らすことでパフォーマンスを改善できる可能性があります。

[DHAT]: https://www.valgrind.org/docs/manual/dh-manual.html
[heaptrack]: https://github.com/KDE/heaptrack

それに加え、128 bytes より大きい Rust の型はインライン化されず `memcpy` によってコピーされます。もしプロファイラー上で `memcpy` の使用量が無視できないほど大きいことが分かった場合、DHAT の "copy profiling" モードを使えば、ホットな `memcpy` 呼び出しの位置やそれに付随する型について正確に把握することができます。それらの型を 128 bytes 以下に縮小することで、`memcpy` の呼び出しを避けメモリへのアクセスを減らすことができ、コードの高速化が期待できます。

## 型のサイズを測定する

[`std::mem::size_of`] は byte 単位での型のサイズを教えてくれますが、しばしばその正確なレイアウトも知りたいことがあるでしょう。例えば、列挙型 (enum) は、他より大きい1つの列挙子により、驚くほど大きくなることがあります。

[`std::mem::size_of`]: https://doc.rust-lang.org/std/mem/fn.size_of.html

`-Zprint-type-sizes` オプションはそれを正確に確認できます。このオプションは stable Rust では使用できず、代わりに nightly Rust を使う必要があります。これは Cargo を使った呼び出しの一例です:

```bash
RUSTFLAGS=-Zprint-type-sizes cargo +nightly build --release
```

また、以下は rustc を使った呼び出しの一例です:

```bash
rustc +nightly -Zprint-type-sizes input.rs
```

これにより、使用中のすべての型のサイズ、レイアウト、及びアラインメントの詳細を見ることができます。例えば下記の型について考えます:

```rust
enum E {
    A,
    B(i32),
    C(u64, u8, u64, u8),
    D(Vec<u32>),
}
```

`-Zprint-type-sizes` オプションは下記に加え、いくつかの組み込み型についての情報も出力します:

```text
print-type-size type: `E`: 32 bytes, alignment: 8 bytes
print-type-size     discriminant: 1 bytes
print-type-size     variant `D`: 31 bytes
print-type-size         padding: 7 bytes
print-type-size         field `.0`: 24 bytes, alignment: 8 bytes
print-type-size     variant `C`: 23 bytes
print-type-size         field `.1`: 1 bytes
print-type-size         field `.3`: 1 bytes
print-type-size         padding: 5 bytes
print-type-size         field `.0`: 8 bytes, alignment: 8 bytes
print-type-size         field `.2`: 8 bytes
print-type-size     variant `B`: 7 bytes
print-type-size         padding: 3 bytes
print-type-size         field `.0`: 4 bytes, alignment: 4 bytes
print-type-size     variant `A`: 0 bytes
```

出力を見ると以下が分かります:
- 型のサイズとアラインメント
- (列挙型の場合) 判定式 (discriminant) のサイズ
- (列挙型の場合) サイズ降順にソートされた各列挙子のサイズ
- すべてのフィールドのサイズ、アラインメント、そして順序 (`E` のサイズを最小化するためにコンパイラが `C` 列挙子のフィールドを並び替えていることに注意してください)
- すべてのパディングのサイズと場所

ホットな型のレイアウトが分かったら、それを縮小するために複数を手法をとることができます。

## フィールドの順序

Rust コンパイラは、`#[repr(C)]` が指定されてない限り、サイズを最小化するために、自動的に構造体と列挙型のフィールドをソートします。そのため、フィールドの順序について心配する必要はありません。ホットな型のサイズを最小化する方法は他にもあります。

## より小さい列挙型

もし列挙型が大きな列挙子を持っている場合、1つ以上のフィールドをボックス化することを検討しましょう。例えば、この型について:

```rust
type LargeType = [u8; 100];
enum A {
    X,
    Y(i32),
    Z(i32, LargeType),
}
```

こうすることができます:

```rust
# type LargeType = [u8; 100];
enum A {
    X,
    Y(i32),
    Z(Box<(i32, LargeType)>),
}
```

これにより、`A::Z` 列挙子について余分なヒープ割り当てを必要とする代わりに、型のサイズを抑えることができます。`A::Z` が比較的使われない場合には、パフォーマンスの向上をより期待できます。`Box` は、特に `match` パターンにおいて、`A::Z` をやや扱いづらくすることにも注意してください。

- [**Example 1**](https://github.com/rust-lang/rust/pull/37445/commits/a920e355ea837a950b484b5791051337cd371f5d)
- [**Example 2**](https://github.com/rust-lang/rust/pull/55346/commits/38d9277a77e982e49df07725b62b21c423b6428e)
- [**Example 3**](https://github.com/rust-lang/rust/pull/64302/commits/b972ac818c98373b6d045956b049dc34932c41be)
- [**Example 4**](https://github.com/rust-lang/rust/pull/64374/commits/2fcd870711ce267c79408ec631f7eba8e0afcdf6)
- [**Example 5**](https://github.com/rust-lang/rust/pull/64394/commits/7f0637da5144c7435e88ea3805021882f077d50c)
- [**Example 6**](https://github.com/rust-lang/rust/pull/71942/commits/27ae2f0d60d9201133e1f9ec7a04c05c8e55e665)

## より小さい整数

より小さい整数型を使用することで、型のサイズを縮小できる可能性は結構あります。例えば、インデックスに `usize` を使うというのはよくあることですが、それを `u32`、`u16`、あるいは `u8` 型で保持しておいて、使用するときに `usize` に型強制 (coerce) してやるというのはそこそこな場面で合理的です。

- [**Example 1**](https://github.com/rust-lang/rust/pull/49993/commits/4d34bfd00a57f8a8bdb60ec3f908c5d4256f8a9a)
- [**Example 2**](https://github.com/rust-lang/rust/pull/50981/commits/8d0fad5d3832c6c1f14542ea0be038274e454524)

## ボックス化されたスライス

Rust のベクタには3つの要素があります。長さ、容量、そしてポインタです。もしあるベクタが将来変更されなさそうな場合、[`Vec::into_boxed_slice`] を使って **ボックス化されたスライス (boxed slice)** に変換できます。ボックス化されたスライスには2つの要素のみあります。長さとポインタです。余っている容量を解放するため、再割り当てが発生し得ます。

```rust
# use std::mem::{size_of, size_of_val};
let v: Vec<u32> = vec![1, 2, 3];
assert_eq!(size_of_val(&v), 3 * size_of::<usize>());

let bs: Box<[u32]> = v.into_boxed_slice();
assert_eq!(size_of_val(&bs), 2 * size_of::<usize>());
```

ボックス化されたスライスは、[`slice::into_vec`] を使ってクローンや再割り当てをすることなくベクタに戻すことができます。

The boxed slice can be converted back to a vector with [`slice::into_vec`]
without any cloning or a reallocation.

[`Vec::into_boxed_slice`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.into_boxed_slice
[`slice::into_vec`]: https://doc.rust-lang.org/std/primitive.slice.html#method.into_vec

## リグレッションを避ける

そのサイズがパフォーマンスに影響を与えるほど、ある型がホットである場合は、それが誤ってリグレッションしないことを保証するために、静的なアサーションを使うことをおすすめします。次の例は [`static_assertions`] クレートのマクロを使っています:

```rust,ignore
  // この型は頻繁に使われているので、意図せず型のサイズが大きくならないことを確かめる。
  #[cfg(target_arch = "x86_64")]
  static_assertions::assert_eq_size!(HotType, [u8; 64]);
```

ここでの `cfg` 属性は重要です。なぜならプラットフォームによって型のサイズが異なることがあるためです。アサーションを `x86_64` (最も広く使われているプラットフォーム) に限定することは、実際のリグレッションを防ぐ上で十分役に立つでしょう。

[`static_assertions`]: https://crates.io/crates/static_assertions
