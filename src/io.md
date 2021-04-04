<!-- commit: https://github.com/nnethercote/perf-book/commit/05d4e28000de22fbf30652ab650fe29ce6ebe9a4 -->

# I/O

([原文](https://nnethercote.github.io/perf-book/io.html))

## ロック

Rust の [`print!`] 及び [`println!`] マクロは呼び出しごとに標準出力をロックします。これらのマクロを繰り返し呼び出すときには、手ずから標準出力をロックした方が良い場合があります。

[`print!`]: https://doc.rust-lang.org/std/macro.print.html
[`println!`]: https://doc.rust-lang.org/std/macro.println.html

例えばこのコードを:

```rust
# let lines = vec!["one", "two", "three"];
for line in lines {
    println!("{}", line);
}
```

このように変更した方が良い場合があります:

```rust
# fn blah() -> Result<(), std::io::Error> {
# let lines = vec!["one", "two", "three"];
use std::io::Write;
let mut stdout = std::io::stdout();
let mut lock = stdout.lock();
for line in lines {
    writeln!(lock, "{}", line)?;
}
// `lock` がドロップする際に標準出力はアンロックされる
# Ok(())
# }
```

このような繰り返し操作を行う場合には、標準入力や標準エラー出力も同様にロックできます。

## バッファリング

Rust のファイル I/O はデフォルトではバッファリングされていません。ファイルやネットワークソケットへの、小さいが繰り返し行われる読み書き処理を大量に行う場合は、[`BufReader`] または [`BufWriter`] を使ってください。これらは必要となるシステムコールの数を最小化しつつ入出力についてのインメモリバッファを管理します。

[`BufReader`]: https://doc.rust-lang.org/std/io/struct.BufReader.html
[`BufWriter`]: https://doc.rust-lang.org/std/io/struct.BufWriter.html

例えば、バッファされていない出力のあることのコードを:
```rust
# fn blah() -> Result<(), std::io::Error> {
# let lines = vec!["one", "two", "three"];
use std::io::Write;
let mut out = std::fs::File::create("test.txt").unwrap();
for line in lines {
    writeln!(out, "{}", line)?;
}
# Ok(())
# }
```

このように変更できます:

```rust
# fn blah() -> Result<(), std::io::Error> {
# let lines = vec!["one", "two", "three"];
use std::io::{BufWriter, Write};
let mut out = std::fs::File::create("test.txt")?;
let mut buf = BufWriter::new(out);
for line in lines {
    writeln!(buf, "{}", line)?;
}
buf.flush()?;
# Ok(())
# }
```

`buf` がドロップした場合自動で強制的な出力 (flush) が行われるため、明示的な [`flush`] の呼び出しは絶対に必要というわけではありません。しかし、暗黙的な呼び出しにおいては、強制的な出力で起きたエラーは無視されることになります。明示的な呼び出しではエラーは無視されません。

[`flush`]: https://doc.rust-lang.org/std/io/trait.Write.html#tymethod.flush

バッファリングは標準出力ともうまく動作するので、標準出力に対して多くの書き込みを行う際には手ずからロック **及び** バッファリングを組み合わせて行うと良いでしょう。

## 入力を生バイト列 (raw bytes) として読み込む

組み込みの [`String`] は内部で UTF-8 を使っています。そのため、入力を読み込む際には UTF-8 バリデーションのために小さいが0ではないオーバーヘッドが発生します。もし、例えば ASCII 文字を処理するときのように、UTF-8について何も考えなくていいようなバイト列入力だけを処理したい場合には、[`BufRead::read_until`] を使用できます。

[`String`]: https://doc.rust-lang.org/std/string/struct.String.html
[`BufRead::read_until`]: https://doc.rust-lang.org/std/io/trait.BufRead.html#method.read_until

また、[バイト指向のデータ行]を読み込んだり、[バイト文字列]を処理したりするための専用のクレートがあります。

[バイト指向のデータ行]: https://github.com/Freaky/rust-linereader
[バイト文字列]: https://github.com/BurntSushi/bstr
