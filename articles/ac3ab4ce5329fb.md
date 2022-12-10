---
title: "Wasmを使った美味しい実行バイナリの作り方"
emoji: "🍳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['rust', 'webassembly', 'wasm']
published: false
---
# Wasmを使った美味しい実行バイナリの作り方
最近、Wasmランタイムの1つである`Wasmer`のバージョン`3.0.0`がリリースされ、色々と機能が追加された。
https://wasmer.io/posts/announcing-wasmer-3.0

その中でも個人的に特に気になったのが、`Support for creating native executables for any platform`という記載。
どうやら、`Wasm`バイナリからOSの実行バイナリを生成できるようになったらしい。
そもそもどうやって生成してる？CやRustのコンパイラから直接生成したときと比べてパフォーマンスはどうなる？バイナリサイズは？など色々と気になった。

これは調べるしかないということで、今回は**Wamserを使った実行バイナリの美味しい作り方**を深堀りしていこうと思う。

# とりあえず使ってみる
まずは使ってみないと何もわからないので、とりあえず`Wasmer`の`3.0`をインストールして使ってみる。

## Wasmer 3.0をインストール
```shell
# これを実行すれば最新のWasmerがインストールされる
$ curl https://get.wasmer.io -sSfL | sh

# Done
$ wasmer --version
wasmer 3.0.2
```

## Wasmバイナリを用意
[Wasm(バイナリ)を読む](https://zenn.dev/0kate/articles/7716f37f7fc327)の時に使ったスパゲッティコードをそのまま使う。
```rust
fn main() {
    println!("Hello, Wasm!");
}
```
```shell
$ cargo build --target wasm32-wasi
$ wasmer target/wasm32-wasi/debug/hello-wasm.wasm
Hello, Wasm!
```

## Wasmバイナリから実行バイナリを生成
ここから新しく追加された`create-exe`サブコマンドで実行バイナリを生成していく。

[公式のブログ](https://wasmer.io/posts/wasm-as-universal-binary-format-part-1-native-executables)では`wasm2wat`を変換しているが、そこはよしなに置き換える。
筆者はLinux上で試しているため、ELF形式のバイナリが生成されるはず。
```shell
# まずは入力となるファイルがWasmバイナリであることを確認
$ file ./hello-wasm.wasm
./hello-wasm.wasm: WebAssembly (wasm) binary module version 0x1 (MVP)

# create-exeを実行
# -o で出力ファイルを指定
$ wasmer create-exe hello-wasm.wasm -o ./hello-wasm
Compiler: cranelift
Target: x86_64-unknown-linux-gnu
Format: Symbols
Using path `path/to/.wasmer/lib/libwasmer.a` as libwasmer path.
✔ Native executable compiled successfully to `./hello-wasm`.

# ELFが吐き出されている！
$ file ./hello-wasm
./hello-wasm: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e711e868f565ae2700255306b13be52fa929990c, for GNU/Linux 3.2.0, with debug_info, not stripped

# もちろん実行できるしちゃんと動く
$ ./hello-wasm
Hello, Wasm!
```
本当にできてしまった。(それはそう)

# 調べていく
実際に生成できることが確認できた所で、もう少し深く調べていく。

## どうやって生成してる？
最初に気になったのは「そもそもどうやってWasmバイナリから実行ファイルを生成しているのか」という所。
`Wasmer`のコードを見ていく。

## バイナリサイズは？

## パフォーマンスは？

# 最後に

# Appendix
- [`Wasmer 3.0`のアナウンス](https://wasmer.io/posts/announcing-wasmer-3.0)
