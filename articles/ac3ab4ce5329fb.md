---
title: "Wasmを使った美味しいネイティブバイナリの作り方"
emoji: "🍳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['rust', 'webassembly', 'wasm']
published: false
---
# Wasmを使った美味しい実行バイナリの作り方
最近、Wasmランタイムの1つである`Wasmer`のバージョン`3.0.0`がリリースされ、色々と機能が追加された。
https://wasmer.io/posts/announcing-wasmer-3.0

その中でも個人的に特に気になったのが、`Support for creating native executables for any platform`という記載。([ニュース](https://www.publickey1.jp/blog/22/wasmwinmaclinuxwasmer_30.html)でも取り上げられていた)
どうやら、`Wasm`バイナリからOSのネイティブバイナリを生成できるようになったらしい。
そもそもどうやって生成してる？とか、CやRustのコンパイラから直接生成したときと比べてパフォーマンスはどうなる？バイナリサイズは？Wasmからネイティブバイナリが生成できてどんな所が嬉しいのか？などなど、色々と気になった。

これは調べるしかないということで、今回は**Wamserを使った美味しいネイティブバイナリの作り方**を深堀りしていこうと思う。

# とりあえず作ってみる
まずはやってみないと何もわからないので、とりあえず`Wasmer`の`3.0.0`をインストールして使ってみる。

## Wasmer 3.0.0をビルド
```shell
# wasmerをクローン(クローン済みの場合はスキップ)
$ git clone https://github.com/wasmerio/wasmer.git
$ cd path/to/wasmer

# 3.0.0のタグに切り替えてビルド(ちょっとだけ待つ)
$ git checkout tags/v3.0.0
$ make build-wasmer

# できあがり
$ ./target/release/wasmer --version
wasmer 3.0.0
```
あとはお好みで、実行しやすいように適当にパスを通しておく。

## Wasmバイナリを用意
[Wasm(バイナリ)を読む](https://zenn.dev/0kate/articles/7716f37f7fc327)の時に使ったスパゲッティをそのまま使う。(スパゲッティとは)
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
ここまでで下準備は終わり。

## Wasmバイナリからネイティブバイナリを生成
ここから新しく追加された`create-exe`サブコマンドでネイティブバイナリを生成していく。

[公式のブログ](https://wasmer.io/posts/wasm-as-universal-binary-format-part-1-native-executables)では`wasm2wat`を変換しているが、そこはよしなに置き換える。
筆者はLinux上で試しているため、ELF形式のバイナリが生成されるはず。
```shell
# まずは入力となるファイルがWasmバイナリであることを確認
$ file ./hello-wasm.wasm
./hello-wasm.wasm: WebAssembly (wasm) binary module version 0x1 (MVP)

# `create-exe`を実行
# `-o`で出力ファイルを指定
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

# いろいろ調べていく
実際に生成できることが確認できた所で、もう少し深くいろいろと調べていく。

## どうやって生成してる？
最初に気になったのは「そもそもどうやってWasmバイナリからネイティブバイナリを生成しているのか」という所。
[公式のブログ](https://wasmer.io/posts/wasm-as-universal-binary-format-part-1-native-executables)にはこんな記載がある。

> In a nutshell, this is what happens under the hood when calling wasmer create-exe: we convert the Wasm to a static object file, generating a C header file to help the linker link the Wasm exported functions with the compiled object file symbols, and then we use a C compiler/linker file to join everything together: the static object (generated from the Wasm file), a minimal libwasmer.a (headless, with no compilers) and the WASI glue code.

ざっくりいうとこういうフローで生成しているらしい。
- `Wasm`バイナリをオブジェクトファイルに変換
⬇
- オブジェクトファイルを取り込むCのヘッダーファイルを生成
⬇
- Cのコンパイラ/リンカでネイティブの実行バイナリを組み立て

さらにその直後にもう少し細かい記載がある。
> 1. First, we adapted the engine, allowing Wasmer load code directly from native objects-symbols that are linked at runtime. The Engine first generates a native object file for a given .wasm file (.o in Linux / macOS or .obj in Windows).
> 2. Once the object file is generated, we generate a header file that links its contents to certain variables at compilation time and plugs them into the Engine with Engine::deserialize_object.
> 3. And once that happens, we just need to use the Wasm-C-API that we all love to interact with this Wasm file!

雑に訳すとこんな感じ。
1. 実際に変換を行うエンジン(以下、変換エンジン)がプラットフォームごとに合わせたオブジェクトファイルを生成する。
   (Linux/macOSの場合は`.o`、Windowsの場合は`.obj`の拡張子)
2. Cのヘッダーファイルを生成して、`Engine::deserialize_object`で変換エンジンに読み込ませる。
3. aaa

実際に`Wasmer`のコードを見ていく。
`create-exe`コマンドの本体は[`wasmer/lib/cli/src/commands/create_exe.rs`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs)にある。

たしかに[`create_exe.rs:287`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs#L287)辺りで、オブジェクトファイルとヘッダーを生成していそうなコードが確認できる。
```rust
let (module_info, obj, metadata_length, symbol_registry) =
    Artifact::generate_object(
        compiler, &data, prefixer, &target, tunables, features,
    )?;

let header_file_src = crate::c_gen::staticlib_header::generate_header_file(
    &module_info,
    &*symbol_registry,
    metadata_length,
);
```

リンクしているのは[`create_exe.rs:319`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs#L319)らへん。
```rust
self.link(
    output_path,
    object_file_path,
    std::path::Path::new("static_defs.h").into(),
    &[],
    None,
    None,
)?;
```

まだまだ全然浅いのでもう少し深く見ていく。

### オブジェクトファイルの作り方 (`generate_object`)
```rust
let (module_info, obj, metadata_length, symbol_registry) =
    Artifact::generate_object(
        compiler, &data, prefixer, &target, tunables, features,
    )?;
```
先の`generate_object`部分がオブジェクトファイルを生成しているであろう場所なわけだが、その実態は[`wasmer/lib/compiler/src/engine/artifact.rs:472`](https://github.com/wasmerio/wasmer/blob/46328d5b2212d4ff438d7ba0114100200e89451a/lib/compiler/src/engine/artifact.rs#L472)にある。
その中でも割と重要そうな部分がこの辺。
```rust
        let compilation: wasmer_types::compilation::function::Compilation = compiler
            .compile_module(
                target,
                &metadata.compile_info,
                module_translation.as_ref().unwrap(),
                function_body_inputs,
            )?;
        let mut obj = get_object_for_target(target_triple).map_err(to_compile_error)?;

        emit_data(&mut obj, WASMER_METADATA_SYMBOL, &metadata_binary, 1)
            .map_err(to_compile_error)?;

        emit_compilation(&mut obj, compilation, &symbol_registry, target_triple)
            .map_err(to_compile_error)?;
```

ここで`compiler`というものが出てくるがこれは呼び出し元から渡されているもので、`create-exe`を実行したときに何が使われているのか出力されている。
```shell
Compiler: cranelift  # <= これ
Target: x86_64-unknown-linux-gnu
Format: Symbols
```
どうやらこのコンパイラが実際のオブジェクトコードを生成している様子。

#### `Wasmer`のコンパイラ事情
ここまでの通り、Wasmからネイティブバイナリを生成するためにコンパイラを挟むことになるわけだが、現在`Wasmer`では以下の3種類のコンパイラに対応している。
これらのコンパイラを目的に応じて使い分けることで、また違う恩恵を受けることができる。
- [Singlepass](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-singlepass)
- [Cranelift](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-cranelift)
- [LLVM](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-llvm)

##### [`Siglepass`](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-singlepass)
こちらはおそらく、直接Wasmバイナリからネイティブの実行ファイルに変換するコンパイラ。
([`One-pass Compiler`](https://en.wikipedia.org/wiki/One-pass_compiler)と置き換えてもだいたい同義？)
このコンパイラのメリットとしては、**コンパイルにかける手間が比較的少ないためコンパイルにかかる時間が推測しやすい**というところ。
こういった一貫したコンパイル時間は、ブロックチェーンの分野などで有効らしい。(ブロックチェーンそこまで詳しくないのでちょっとイメージついていない)

##### [`Crafnelift`](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-cranelift)
こちらは`Cranelift IR`(`Cranelift`特有の中間表現)を任意のマシンコードに変換できるコンパイラ。(デフォルト使われるのもこれ)
`Wasmtime`とかを持ってる[Bytecode Alliance](https://bytecodealliance.org/)のプロジェクトの一つ。
このコンパイラの使いどころとしては特に書いてなかったが、開発時のみの使用が推奨らしい。本番では後述のLLVMを使ったほうが良いとのこと。
ちょっと考えてみたけど、Rustのクレートとして提供されるため再現性が用意である所？くらいしか思いつかない。

##### [`LLVM`](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-llvm)
言わずとしれた、`Clang`とかの裏側でバックエンドコンパイラとして使われていたりする超有名なコンパイラ。
`LLVM`専用の中間表現からプラットフォーム固有のネイティブバイナリに変換する。
本番環境でWasmを利用する際は、ネイティブコード並みにパフォーマンスも最大化できる点からこのコンパイラが推奨とのこと。(実績も豊富なコンパイラだしこれは納得)

#### `Cranelift`でのコンパイル
とりあえず「デフォルトで使われるから」という理由だけではあるが、`Cranelift`でのコンパイルをもう少し見ていこうと思う。
オブジェクトファイルを生成する際に先のコンパイラの`compile_module`というメソッドが呼ばれるわけだが、`Cranelift`でのコンパイルの場合は[`wasmer/lib/compiler-cranelift/src/compiler.rs:62`](https://github.com/wasmerio/wasmer/blob/master/lib/compiler-cranelift/src/compiler.rs#L62)に実装がある。

### ヘッダーファイルの作り方 (`generate_header_file`)
```rust
let header_file_src = crate::c_gen::staticlib_header::generate_header_file(
    &module_info,
    &*symbol_registry,
    metadata_length,
);
```
次はヘッダーファイルが生成される所を見ていく。ここのそれらしい関数は`generate_header_file`で、実態は[`wasmer/lib/cli/src/c_gen/staticlib_header.rs:72`](https://github.com/wasmerio/wasmer/blob/ec959640a301733360d2c1b91516e1aa47be9de3/lib/cli/src/c_gen/staticlib_header.rs#L72)にある。

### リンク (`link`)

## バイナリサイズは？
次はバイナリサイズの所。
CやRustなどのコンパイラで直接生成したネイティブバイナリと比べてサイズはどれくらい違うのか？
もちろん「言語やコンパイラによって違う」と言ってしまえばそれまでなのだが、雰囲気だけでも知りたいのでちょっと調べてみる。

コードはあいも変わらずこのスパゲッティ。(スパゲッティとは)
```rust
fn main() {
    println!("Hello, Wasm!");
}
```

ビルドしてサイズを見てみる。(ビルド手順は割愛)
```shell
# まずはRustコンパイラから直接ビルドしたネイティブバイナリ
$ ./target/debug/hello-wasm
Hello, Wasm!
$ du -h ./target/debug/hello-wasm
3.8M	./target/debug/hello-wasm

# 次にWasmバイナリから生成したネイティブバイナリ
$ ./target/wasm32-wasi/debug/hello-wasm
Hello, Wasm!
$ du -h ./target/wasm32-wasi/debug/hello-wasm
29M	./target/wasm32-wasi/debug/hello-wasm

# ちなみにWasmバイナリのサイズ
$ wasmer ./target/wasm32-wasi/debug/hello-wasm.wasm
Hello, Wasm!
$ du -h ./target/wasm32-wasi/debug/hello-wasm.wasm
2.0M	./target/wasm32-wasi/debug/hello-wasm.wasm
```
Rustのバイナリは大きめと言われるが、それよりもさらに大きい。
想像してたよりもデカいし「こんなに変わるのか...」と思いつつ、これはどう最適化していけるのか興味深い所。

## パフォーマンスは？
パフォーマンスについても気になる。
こちらもCやRustなどのコンパイラで直接生成したネイティブバイナリと比べてどうなのか。
これもコンパイラによって違うと言ってしまえばそうなのだがやっていく。

速度比較でよくある、`n`番目のフィボナッチ数を計算させてみる。
(とりあえず`n = 45`でやってみる)
```rust
fn fibonacci(n: i64) -> i64 {
    match n {
        1 | 2 => 1,
        _ => fibonacci(n - 2) + fibonacci(n - 1),
    }
}

fn main() {
    println!("{}", fibonacci(45));
}
```

ビルドして計測。(ビルド手順は割愛)
```shell
# Rustで直接生成したバイナリ
$ time -p ./target/debug/fib > /dev/null
real 9.98
user 9.97
sys 0.00

# Wasmから生成したバイナリ
$ time -p ./target/wasm32-wasi/debug/fib > /dev/null
real 12.71
user 12.68
sys 0.02

# ちなみにWasmのバイナリを実行した場合
$ time -p wasmer ./target/wasm32-wasi/debug/fib.wasm > /dev/null
real 13.55
user 13.51
sys 0.02
```
Wasmバイナリから生成した場合だと少しばかり遅くなるみたい。
だがWasmバイナリを実行した場合よりは、変換後のネイティブバイナリの方が速くなるのは言うまでもない。
# クロスコンパイルもできるらしい
なんとクロスコンパイルもできるらしく、例えば「Linuxホスト上でWasmバイナリからexe形式のバイナリを生成する」みたいなこともできるみたい。
> In Wasmer 3.0 we used the power of Zig for doing cross-compilation from the C glue code into other machines.

クロスコンパイルには[Zig](https://ziglang.org/)を使っており、たしかにところどころコードにも`Zig`の記載が見える。
(最近`Zig`の名前を聞くことも増えてきた気がする。)

# 何が嬉しそう？
Wasmバイナリのままで済めばそのまま実行すれば良いだけの話なので、「Wasmのポータビリティを享受しつつ、速度面でネイティブバイナリの恩恵も受けたい」というタイミングなら嬉しいかも？
(そもそもWasm自体がネイティブ並の実行速度を目指している側面もあるが)

## デスクトップアプリとか？
全然Wasm入門したてで界隈に詳しくないド素人の激浅見解だが、まずは処理量が多いデスクトップアプリとかどうだろう？
PCゲームや動画編集ソフトなど、処理が複雑でパフォーマンス的にWasmバイナリの実行では物足りないケースとか。
Linuxをメインで使っていると対応してないソフトウェア(特にPCゲーム)とかもあるが、一度Wasmバイナリとして吐き出すようにすればどのプラットフォームにも対応できるようにはなるし、パフォーマンス的にもネイティブの恩恵を受けられそう。(そもそもLinux上でゲームがしたいのだろうかという点があるが)
Linux上でWindowsアプリを動かすために`Wine`など使うこともあるが、その代わりにならないだろうか。

# 最後に
よくよく考えたら、WasmによってあらゆるソフトウェアがWebブラウザに移植されどんどん集約されていく世界線の中で、ある意味逆行しているとも捉えることもできそうな今回の新機能。(実際[SQLite3 WASM/JS](https://www.publickey1.jp/blog/22/sqlite3_wasmjssqlite_340websqlite.html)で、SQLite3とかもWebブラウザ対応が進められていたり)
この辺について`Wasmer`の技術戦略的なところも気になるきっかけになった。

# Appendix
- [`Wasmer 3.0`のアナウンス](https://wasmer.io/posts/announcing-wasmer-3.0)
- [`Zig`](https://ziglang.org/)(クロスコンパイルに使われている)
- [SQLite3 WASM/JS](https://www.publickey1.jp/blog/22/sqlite3_wasmjssqlite_340websqlite.html)
