---
title: "Wasmを使った美味しいネイティブバイナリの作り方"
emoji: "🍳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['rust', 'webassembly', 'wasm']
published: false
---
# Wasmを使った美味しい実行バイナリの作り方
最近Wasmランタイムの1つである`Wasmer`のバージョン`3.0.0`がリリースされ、色々と機能が追加された。
https://wasmer.io/posts/announcing-wasmer-3.0

その中でも個人的に特に気になったのが、`Support for creating native executables for any platform`という記載。([ニュース](https://www.publickey1.jp/blog/22/wasmwinmaclinuxwasmer_30.html)でも取り上げられていた)
どうやら、WasmバイナリからOSのネイティブバイナリを生成できるようになったらしい。
そもそもどうやって生成してる？とか、CやRustのコンパイラから直接生成したときと比べてパフォーマンスはどうなる？バイナリサイズは？などなど、色々と気になった。

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
[WebAssembly 入門してみた](https://zenn.dev/0kate/articles/f3c38767dd62eb)の時に使ったスパゲッティをそのまま使う。(スパゲッティとは)
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
- Wasmバイナリからオブジェクトファイル(よく`.o`とか`.obj`とかがついているやつ)を生成
⬇
- オブジェクトファイルと紐付けるCのヘッダーファイルを生成
⬇
- Cのコンパイラ/リンカでネイティブの実行ファイルを組み立て

さらにその後にもう少し細かい記載がある。
> 1. First, we adapted the engine, allowing Wasmer load code directly from native objects-symbols that are linked at runtime. The Engine first generates a native object file for a given .wasm file (.o in Linux / macOS or .obj in Windows).
> 2. Once the object file is generated, we generate a header file that links its contents to certain variables at compilation time and plugs them into the Engine with Engine::deserialize_object.
> 3. And once that happens, we just need to use the Wasm-C-API that we all love to interact with this Wasm file!

雑に訳すとこんな感じ。
1. 実際に変換を行うエンジン(以下、変換エンジン)がプラットフォームごとに合わせたオブジェクトファイルを生成する。
   (Linux/macOSの場合は`.o`、Windowsの場合は`.obj`の拡張子)
2. Cのヘッダーファイルを生成して、変換エンジンに読み込ませる。
3. あとは良い感じにやる

コードを見てみたほうが手っ取り早そうなので、実際に`Wasmer`のコードを見ていく。
`create-exe`コマンドの本体は[`wasmer/lib/cli/src/commands/create_exe.rs`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs)にある。

たしかに[`create_exe.rs:287`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs#L287)辺りで、オブジェクトファイルとヘッダーを生成していそうなコードが見える。
```rust
...
let (module_info, obj, metadata_length, symbol_registry) =
    Artifact::generate_object(
        compiler, &data, prefixer, &target, tunables, features,
    )?;

let header_file_src = crate::c_gen::staticlib_header::generate_header_file(
    &module_info,
    &*symbol_registry,
    metadata_length,
);
...
```

リンクしているのは[`create_exe.rs:319`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs#L319)辺り。
```rust
...
self.link(
    output_path,
    object_file_path,
    std::path::Path::new("static_defs.h").into(),
    &[],
    None,
    None,
)?;
...
```

それぞれもう少し深く見ていく。

### オブジェクトファイルの作り方 (`generate_object`)
```rust
...
let (module_info, obj, metadata_length, symbol_registry) =
    Artifact::generate_object(
        compiler, &data, prefixer, &target, tunables, features,
    )?;
...
```
先の`generate_object`部分がオブジェクトファイルを生成しているであろうコードなわけだが、その実態は[`wasmer/lib/compiler/src/engine/artifact.rs:472`](https://github.com/wasmerio/wasmer/blob/46328d5b2212d4ff438d7ba0114100200e89451a/lib/compiler/src/engine/artifact.rs#L472)にある。

その中でも割と重要そうな部分がこの辺。
```rust
...
#[allow(dead_code)]
let (compile_info, function_body_inputs, data_initializers, module_translation) =
    Self::generate_metadata(data, compiler, tunables, features)?;  // メタデータを生成してる？
...
let compilation: wasmer_types::compilation::function::Compilation = compiler
    .compile_module(  // コンパイルしてる？
        target,
        &metadata.compile_info,
        module_translation.as_ref().unwrap(),
        function_body_inputs,
    )?;
let mut obj = get_object_for_target(target_triple).map_err(to_compile_error)?;

// なんかemitしてる
emit_data(&mut obj, WASMER_METADATA_SYMBOL, &metadata_binary, 1)
    .map_err(to_compile_error)?;

// なんかemitしてる
emit_compilation(&mut obj, compilation, &symbol_registry, target_triple)
    .map_err(to_compile_error)?;
...
```
なんか生成しているそれっぽい雰囲気はめちゃくちゃ出ているが、それぞれ深ぼっていく前に`compiler`とか「**コンパイル**」とかが出てきたので、これについて少しだけ見ていく。

ここで言うコンパイラというのは、その名の通り**Wasmバイナリを何かしらの方法でプラットフォームごとのバイナリに変換する**役割を持つやつ。
`create-exe`の時に何が使われているのかは、実行時の出力で確認できる。
```shell
Compiler: cranelift  # <= これ
Target: x86_64-unknown-linux-gnu
Format: Symbols
```
`cranelift`ってなんぞ。

#### `Wasmer`のコンパイラ事情
ここまでの通り、Wasmからネイティブバイナリを生成するためにコンパイラを挟むことになるわけだが、現在`Wasmer`では以下の3種類のコンパイラに対応している。
これらのコンパイラを目的に応じて使い分けるのが推奨される。
- [Singlepass](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-singlepass)
- [Cranelift](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-cranelift)
- [LLVM](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-llvm)

##### [`Siglepass`](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-singlepass)
こちらはおそらく、直接Wasmバイナリからネイティブの実行ファイルに変換するコンパイラ。
([`One-pass Compiler`](https://en.wikipedia.org/wiki/One-pass_compiler)と置き換えてもだいたい同義？)
このコンパイラのメリットとしては、**コンパイルにかける手間が比較的少ないためコンパイルにかかる時間が推測しやすい**というところ。
ただ、コンパイル時間は後述の2つより圧倒的に速いものの、実行時のパフォーマンスは遅いらしい。
こういった一貫したコンパイル時間は、ブロックチェーンの分野などで有効とのこと。(ブロックチェーンそこまで詳しくないのでちょっとイメージついていない)

##### [`Crafnelift`](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-cranelift)
こちらは、`Cranelift IR`という専用の中間表現を任意のマシンコードに変換するコンパイラ。(デフォルト使われるのもこれ)
`Wasmtime`とかを持ってる[Bytecode Alliance](https://bytecodealliance.org/)のプロジェクトの一つ。
このコンパイラの使いどころとしては特に書いてなかったが、開発時のみの使用が推奨らしい。本番では後述のLLVMを使ったほうが良いとのこと。
> We recommend using this compiler crate only for development proposes.

##### [`LLVM`](https://github.com/wasmerio/wasmer/tree/master/lib/compiler-llvm)
こちらは言わずとしれた、`Clang`とかの裏側でバックエンドコンパイラとして使われていたりする超有名コンパイラ。
`LLVM`専用の中間表現からプラットフォーム固有のネイティブバイナリに変換する。
本番環境でWasmを利用する際はこちらが推奨とのこと。(ネイティブコード並みにパフォーマンスも最大化できるため)
> We recommend using LLVM as the default compiler when running WebAssembly files on any production system, as it offers maximum peformance near to native speeds.

実績も豊富なコンパイラだしこれは納得。

「コンパイラ」とか急に出てきてビビり散らかしていたのがスッキリした所で、先程の処理に戻って少しだけ深く調べていく。

#### `generate_metadata`
まずは先程抜粋したコードで最初に呼ばれている`generate_metadata`。
```rust
#[allow(dead_code)]
let (compile_info, function_body_inputs, data_initializers, module_translation) =
    Self::generate_metadata(data, compiler, tunables, features)?;
```
Wasmのバイナリを解析して、以下のメタデータを取得するのがここの処理。
- テーブルセクション (ELFで言うところの関数やらのシンボル情報に相当？)
- データセクション (ELFで言うところの静的データを書いてある所に相当？)
- コードセクション (ELFで言うところの`.text`とかにある各関数のコードへの参照に相当？)

ちなみにこれらのWasmバイナリがどういう情報なのかに関しては、[Wasm(バイナリ)を読む](https://zenn.dev/0kate/articles/7716f37f7fc327)を読むべし。

#### `compiler.compile_module`
次に呼ばれている`compiler.compile_module`。
先程のコンパイラのいずれかから指定されたものの`compile_module`が呼び出される。
```rust
let compilation: wasmer_types::compilation::function::Compilation = compiler
    .compile_module(
        target,
        &metadata.compile_info,
        module_translation.as_ref().unwrap(),
        function_body_inputs,
    )?;
```

`compile_module`の中身をいろいろデフォルメするとこんな感じ。
`Cranelift`の中間表現を介して、バイナリにコンパイルしている。(`Cranelift`の変換処理については、`Wasmer`から逸れるので今回は割愛)
```rust
/// Compile the module using Cranelift, producing a compilation result with
/// associated relocations.
fn compile_module(
    ...
) -> Result<Compilation, CompileError> {
    ...

    let mut func_translator = FuncTranslator::new();
    let (functions, fdes): (Vec<CompiledFunction>, Vec<_>) = function_body_inputs
    .map(|(i, input)| {
        ...
        // WasmのバイナリからCranelift IRに変換する
        func_translator.translate(
            module_translation_state,
            &mut reader,
            &mut context.func,
            &mut func_env,
            i,
        )?;

        // ここで変換したCranelift IRからバイナリを生成
        let mut code_buf: Vec<u8> = Vec::new();
        context
            .compile_and_emit(&*isa, &mut code_buf)
            .map_err(|error| CompileError::Codegen(pretty_error(&context.func, error)))?;
        ...

    // function call trampolines (only for local functions, by signature)
    // ここではWasmモジュール内の関数シグネチャを生成 (出たなトランポリン)
    let mut cx = FunctionBuilderContext::new();
    let function_call_trampolines = module
        ...

    // dynamic function trampolines (only for imported functions)
    // ここでは外部から差し込まれる関数への参照を生成 (出たなトランポリン)
    let mut cx = FunctionBuilderContext::new();
    let dynamic_function_trampolines = module
        ...
```
以前遭遇したトランポリンがまた出てきたが、これは別でまとめたい。(継続やトランポリン実行に関する文献？を教えていただいたので読んでみた)

#### `emit_data`
その次は、なにやら`emit`している`emit_data`。
```rust
emit_data(&mut obj, WASMER_METADATA_SYMBOL, &metadata_binary, 1)
    .map_err(to_compile_error)?;
```

ざっくり言うとここは、`generate_metadata`で生成したメタデータをプラットフォームごとの実行形式に合わせていくところ。
中身はこんな感じ。
```rust
pub fn emit_data(
    obj: &mut Object,
    name: &[u8],
    data: &[u8],
    align: u64,
) -> Result<(), ObjectError> {
    let symbol_id = obj.add_symbol(ObjSymbol {
        name: name.to_vec(),
        value: 0,
        size: 0,
        kind: SymbolKind::Data,
        scope: SymbolScope::Dynamic,
        weak: false,
        section: SymbolSection::Undefined,
        flags: SymbolFlags::None,
    });
    let section_id = obj.section_id(StandardSection::Data);
    obj.add_symbol_data(symbol_id, section_id, data, align);

    Ok(())
}
```
`obj`は[`object`](https://github.com/gimli-rs/object)クレートのもので、各プラットフォームの実行形式に合わせて読み書きできるインターフェースを提供する構造体。
`add_symbol`とか`add_symbol_data`とかで、シンボル情報を追加していっているのが見える。

#### `emit_compilation`
最後は、ここもなにやら`emit`している`emit_compilation`。
```rust
emit_compilation(&mut obj, compilation, &symbol_registry, target_triple)
    .map_err(to_compile_error)?;
```

ここも先程の`emit_data`同様、生成した実際のコードに相当するバイナリをプラットフォームごとの実行形式に合わせていくところ。
```rust
pub fn emit_compilation(
    obj: &mut Object,
    compilation: Compilation,
    symbol_registry: &impl SymbolRegistry,
    triple: &Triple,
) -> Result<(), ObjectError> {
    ...
    // Add sections
    let custom_section_ids = compilation
        ...

    // Add functions
    let function_symbol_ids = function_bodies
        ...


    // Add function call trampolines
    for (signature_index, function) in compilation.function_call_trampolines.into_iter() {
        ...
```
こちらも`object`クレートを使っている。

こういった処理が行われた後`generate_object`に戻り、最後は生成したオブジェクトコードを`function.o`というファイルに書き込んでいる。
```rust
...
let (module_info, obj, metadata_length, symbol_registry) =  // ここでオブジェクトコード`obj`を取得
    Artifact::generate_object(
        compiler, &data, prefixer, &target, tunables, features,
    )?;

...

// Write object file with functions
let object_file_path: std::path::PathBuf =
    std::path::Path::new("functions.o").into();                         // `function.o`
    let mut writer = BufWriter::new(File::create(&object_file_path)?);  // ファイルの作成 & writerの作成
    obj.write_stream(&mut writer)                                       // ここでオブジェクトコードを書き込み
        .map_err(|err| anyhow::anyhow!(err.to_string()))?;
    writer.flush()?;
...
```

### ヘッダーファイルの作り方 (`generate_header_file`)
次はヘッダーファイルの生成を見ていく。
```rust
let header_file_src = crate::c_gen::staticlib_header::generate_header_file(
    &module_info,
    &*symbol_registry,
    metadata_length,
);
```
この`generate_header_file`の実装は、[`wasmer/lib/cli/src/c_gen/staticlib_header.rs:72`](https://github.com/wasmerio/wasmer/blob/ec959640a301733360d2c1b91516e1aa47be9de3/lib/cli/src/c_gen/staticlib_header.rs#L72)にある。
この関数の役割は、**Wasmバイナリから生成したオブジェクトファイルのシンボル情報**をもとに**ヘッダーファイルに書かれるべきCのソースコード**を列挙、実際にヘッダーファイルの生成を行う`generate_c`関数に引き渡すこと。
```rust
pub fn generate_header_file(
    module_info: &ModuleInfo,
    symbol_registry: &dyn SymbolRegistry,
    metadata_length: usize,
) -> String {
    // c_statementsにCのソースコード文を集めている
    let mut c_statements = vec![
        CStatement::LiteralConstant {
            value: "#include \"wasmer.h\"\n#include <stdlib.h>\n#include <string.h>\n\n"
                .to_string(),
        },
        CStatement::LiteralConstant {
            value: "#ifdef __cplusplus\nextern \"C\" {\n#endif\n\n".to_string(),
        },

    ...(長いので省略)

    c_statements.push(CStatement::LiteralConstant {
        value: "\n#ifdef __cplusplus\n}\n#endif\n\n".to_string(),
    });

    generate_c(&c_statements)
}
```
たしかに、Cのソースコードを文字列として含んだデータ型が`push`されまくっている。
実際に生成している`generate_c`は、[`wasmer/lib/cli/src/c_gen/mod.rs:346`](https://github.com/wasmerio/wasmer/blob/ec959640a301733360d2c1b91516e1aa47be9de3/lib/cli/src/c_gen/mod.rs#L346)に書いてある。
```rust
pub fn generate_c(statements: &[CStatement]) -> String {
    let mut out = String::new();
    for statement in statements {
        statement.generate_c(&mut out);
    }
    out
}
```
ここは思っていたよりかなり薄く作られており、直前で列挙された「ヘッダーファイルに書き出されるべきソースコード文」の一覧に対して一つずつ`generate_c`メソッドを呼び出しているだけみたい。
引数で渡している文字列に、文ごとの文字列を末尾に追加してもらっている。

それぞれのソースコード文の`generate_c`は各文の特性に応じて処理が分岐しており、例えば宣言文の場合はこんな感じ。
```rust
impl CStatement {
    /// Generate C source code for the given CStatement.
    fn generate_c(&self, w: &mut String) {
        match &self {
            Self::Declaration {
                name,
                is_extern,
                is_const,
                ctype,
                definition,
            } => {
                if *is_const {
                    w.push_str("const ");
                }
                if *is_extern {
                    w.push_str("extern ");
                }
                ctype.generate_c_with_name(name, w);
                if let Some(def) = definition {
                    w.push_str(" = ");
                    def.generate_c(w);
                }
                w.push(';');
                w.push('\n');
            }
    ...
```
なるほど。。めちゃくちゃCのソースコードを文字列で作っている。
ここでまた呼び出し元のに戻り、作られたソースコードは最終的に`static_defs.h`というファイルに書き込まれる。
```rust
let header_file_src = crate::c_gen::staticlib_header::generate_header_file(  // ここでヘッダーファイルのソースコードを生成
    &module_info,
    &*symbol_registry,
    metadata_length,
);
...
// Write down header file that includes pointer arrays and the deserialize function
let mut writer = BufWriter::new(File::create("static_defs.h")?);  // `static_defs.h`
writer.write_all(header_file_src.as_bytes())?;                    // 生成したソースコードを書き込み
writer.flush()?;
...
```

### リンク (`link`)
最後に、ここまで作ってきた`funciton.o`と`static_defs.h`を使って実行ファイルにリンクいく。
```rust
self.link(
    output_path,                                   // 出力先のパス
    object_file_path,                              // 生成されたオブジェクトファイルのパス (`function.o`)
    std::path::Path::new("static_defs.h").into(),  // 生成されたヘッダーファイルのパス (`static_defs.h`)
    &[],
    None,
    None,
)?;
```
実際に`link`していく処理が書かれているのは[`wasmer/lib/cli/src/commands/create_exe.rs:1041`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/create_exe.rs#L1041)。
ここでやっていることは、ざっくり書くとこんな感じ。
- ビルド用の一時ディレクトリとして`wasmer-static-compile`ディレクトリを作成
⬇
- 作成した一時ディレクトリに、エントリポイントとなるCのソースファイル`wasmer_main.c`を作成
⬇
- ビルド時に必要なライブラリ(`libwasmer.a`)の探索と、ヘッダファイル(`wasmer.h`, `wasm.h`)の一時ディレクトリへのコピー
⬇
- 一時ディレクトリ内で`cc`コマンドを実行、`main_obj.obj`というオブジェクトファイルを作成
⬇
- OSごとのリンク処理を呼び出して、最終的なネイティブの実行バイナリを生成

それぞれ少しずつ深く見ていく。

#### ビルド用の一時ディレクトリを作成
ここは一時ディレクトリを作っているだけなのでサラッと流す。
```rust
let tempdir = tempdir::TempDir::new("wasmer-static-compile")?;  // `wasmer-static-compile`
let tempdir_path = tempdir.path();
```

#### エントリポイントとなるソースファイルを作成
作成したビルド用の一時ディレクトリに`wasmer_main.c`というファイルを作成して、エントリポイントとなるコードを書き込んでいる。
```rust
...

let c_src_path = tempdir_path.join("wasmer_main.c");  // `wasmer_main.c`
...

// ここでソースファイルを作成して、エントリポイントとなるコードを書き込み
if let Some(entrypoint) = pirita_main_atom.as_ref() {
    let c_code = Self::generate_pirita_wasmer_main_c_static(pirita_atoms, entrypoint);
    std::fs::write(&c_src_path, c_code)?;
} else {
    std::fs::write(&c_src_path, WASMER_STATIC_MAIN_C_SOURCE)?;
}
...
```
ちょっと長いので割愛するが、実際に書き込んでいるコードは[`wasmer/lib/cli/src/commands/wasmer_create_exe_main.c`](https://github.com/wasmerio/wasmer/blob/master/lib/cli/src/commands/wasmer_create_exe_main.c)にある。

#### ビルドに必要なライブラリの探索と、ヘッダーファイルのコピー
まずは共有ライブラリ`libwasmer.a`の探索。`WASMER_DIR`(デフォルトで`$HOME/.wasmer`)から探している。
```rust
...

// `libwasmer.a`の探索 (見つからなかったらエラー)
let mut libwasmer_path = get_libwasmer_path()?
    .canonicalize()
    .context("Failed to find libwasmer")?;

let lib_filename = libwasmer_path
    .file_name()
    .unwrap()
    .to_str()
    .unwrap()
    .to_string();
libwasmer_path.pop();

...
```

次にヘッダーファイル`wasmer.h`, `wasm.h`のコピー。これも`WASMER_DIR`にある。
```rust
...

// `wasmer.h`の探索 (見つからなかったらエラー)
let wasmer_include_dir = get_wasmer_include_directory()?;
let wasmer_h_path = wasmer_include_dir.join("wasmer.h");
if !wasmer_h_path.exists() {
    return Err(anyhow::anyhow!(
        "Could not find wasmer.h in {}",
        wasmer_include_dir.display()
    ));
}

// `wasm.h`の探索 (見つからなかったらエラー)
let wasm_h_path = wasmer_include_dir.join("wasm.h");
if !wasm_h_path.exists() {
    return Err(anyhow::anyhow!(
        "Could not find wasm.h in {}",
        wasmer_include_dir.display()
    ));
}

// コピー
std::fs::copy(wasmer_h_path, header_code_path.join("wasmer.h"))?;
std::fs::copy(wasm_h_path, header_code_path.join("wasm.h"))?;
...
```

#### `cc`コマンドを実行して`main_obj.obj`を生成
これまで生成したファイルを合体させて、`main_obj.obj`というオブジェクトファイルを生成。
```rust
...

let compilation = {
    Command::new("cc")
        .arg("-c")
        .arg(&c_src_path)
        .arg(if linkcode.optimization_flag.is_empty() {
            "-O2"
        } else {
            linkcode.optimization_flag.as_str()
        })
        .arg(&format!("-L{}", libwasmer_path.display()))  // `libwasmer.a`があるディレクトリ
        .arg(&format!("-l:{}", lib_filename))             // 探索するライブラリの名前 (`libwasmer.a`にマッチ)
        //.arg("-lwasmer")
        // Add libraries required per platform.
        // We need userenv, sockets (Ws2_32), advapi32 for some system calls and bcrypt for random numbers.
        //#[cfg(windows)]
        //    .arg("-luserenv")
        //    .arg("-lWs2_32")
        //    .arg("-ladvapi32")
        //    .arg("-lbcrypt")
        // On unix we need dlopen-related symbols, libmath for a few things, and pthreads.
        //#[cfg(not(windows))]
        .arg("-ldl")
        .arg("-lm")
        .arg("-pthread")
        .arg(&format!("-I{}", header_code_path.display()))  // `static_defs.h`とか`wasm.h`とかのヘッダーファイルがあるディレクトリ
        .arg("-v")
        .arg("-o")
        .arg("main_obj.obj")  // 出力先
        .output()?
};
...
```

#### OSごとのリンク処理を呼び出してネイティブバイナリを生成
最後に、OSごとのリンク処理が実行されて実行ファイルに仕上げられる。
```rust
...

let linkcode = LinkCode {
    object_paths,
    output_path,
    ..Default::default()
};
...

linkcode.run().context("Failed to link objects together")?;
...
```

`linkcode.run`の実装は、[`wasmer/lib/cli/src/commands/create_exe.rs:1320`](https://github.com/wasmerio/wasmer/blob/b3ea89b573a18ae2acf7939a1a40957161c5c363/lib/cli/src/commands/create_exe.rs#L1320)にある。
```rust
impl LinkCode {
    fn run(&self) -> anyhow::Result<()> {
        // `libwasmer.a`
        let libwasmer_path = self
            .libwasmer_path
            .canonicalize()
            .context("Failed to find libwasmer")?;
        println!(
            "Using path `{}` as libwasmer path.",
            libwasmer_path.display()
        );

        // リンカのコマンドを実行
        let mut command = Command::new(&self.linker_path);  // Windowsなら`clang`、それ以外なら`cc`
        let command = command
            .arg("-Wall")
            .arg(&self.optimization_flag)
            .args(
                self.object_paths
                    .iter()
                    .map(|path| path.canonicalize().unwrap()),
            )
            .arg(&libwasmer_path);
        let command = if let Some(target) = &self.target {
            command.arg("-target").arg(format!("{}", target))
        } else {
            command
        };
        // Add libraries required per platform.
        // We need userenv, sockets (Ws2_32), advapi32 for some system calls and bcrypt for random numbers.
        #[cfg(windows)]
        let command = command
            .arg("-luserenv")
            .arg("-lWs2_32")
            .arg("-ladvapi32")
            .arg("-lbcrypt");
        // On unix we need dlopen-related symbols, libmath for a few things, and pthreads.
        #[cfg(not(windows))]
        let command = command.arg("-ldl").arg("-lm").arg("-pthread");
        let link_against_extra_libs = self
            .additional_libraries
            .iter()
            .map(|lib| format!("-l{}", lib));
        let command = command.args(link_against_extra_libs);
        let command = command.arg("-o").arg(&self.output_path);
        let output = command.output()?;

        if !output.status.success() {
            bail!(
                "linking failed with: stdout: {}\n\nstderr: {}",
                std::str::from_utf8(&output.stdout)
                    .expect("stdout is not utf8! need to handle arbitrary bytes"),
                std::str::from_utf8(&output.stderr)
                    .expect("stderr is not utf8! need to handle arbitrary bytes")
            );
        }
        Ok(())
    }
}
```
こうやって最終的に、指定した出力先にネイティブバイナリが吐き出される。
ヘッダーはソースコードを文字列で書き込んで生成したり、外部のコマンドを呼び出したりと、「思ったより力技なんだな、、」といった雑感だがこれはこれで面白い。
生成方法がなんとなくわかった所で、あと少しだけ気になったところを調べていく。

## バイナリサイズは？
バイナリサイズの所。
CやRustなどのコンパイラで直接生成したネイティブバイナリと比べてサイズはどれくらい違うのか？
もちろん「言語やコンパイラによって違う」と言ってしまえばそれまでなのだが、雰囲気だけでも知りたいのでちょっと調べてみる。

コードはあいも変わらずこのスパゲッティ。(スパゲッティとは)
```rust
fn main() {
    println!("Hello, Wasm!");
}
```

リリースビルドで、`Singlepass`・`Cranelift`・`LLVM`とそれぞれのコンパイラで試してみる。
```shell
# まずはRustコンパイラから直接ビルドしたネイティブバイナリ
$ cargo build --release
$ du -h ./target/release/hello-wasm
3.8M	./target/release/hello-wasm

# ちなみにWasmバイナリのサイズ
$ cargo build --release --target wasm32-wasi
$ du -h ./target/wasm32-wasi/release/hello-wasm.wasm
2.0M	./target/wasm32-wasi/release/hello-wasm.wasm

# Wasmバイナリから生成したバイナリ (Singlepass)
$ wasmer create-exe ./target/wasm32-wasi/release/hello-wasm.wasm -o ./target/wasm32-wasi-release/hello-wasm-singlepass --singlepass
$ du -h ./target/wasm32-wasi/release/hello-wasm-singlepass
29M	./target/wasm32-wasi/release/hello-wasm-singlepass

# Wasmバイナリから生成したバイナリ (Cranelift)
$ wasmer create-exe ./target/wasm32-wasi/release/hello-wasm.wasm -o ./target/wasm32-wasi-release/hello-wasm-cranelift
$ du -h ./target/wasm32-wasi/release/hello-wasm-cranelift
29M	./target/wasm32-wasi/release/hello-wasm-cranelift

# Wasmバイナリから生成したバイナリ (LLVM)
$ wasmer create-exe ./target/wasm32-wasi/release/hello-wasm.wasm -o ./target/wasm32-wasi-release/hello-wasm-llvm --llvm
$ du -h ./target/wasm32-wasi/release/hello-wasm-llvm
29M	./target/wasm32-wasi/release/hello-wasm-llvm
```
Rustのバイナリは大きめと言われるが、Wasmから生成したバイナリはそれよりもさらに大きい。
「こんなに変わるのか...」と思いつつ、これはどう最適化していけるのか興味深い所。
裏側で使うコンパイラの違いは特にバイナリサイズには影響しない？

## パフォーマンスは？
パフォーマンスについても気になる。
こちらもCやRustなどのコンパイラで直接生成したネイティブバイナリと比べてどうなのか。
バイナリサイズと同じく、コンパイラによって違うと言ってしまえばそうなのだがとりあえずやっていく。

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

ビルドして計測。
こちらもリリースビルドで、`Singlepass`・`Cranelift`・`LLVM`とそれぞれのコンパイラで試してみる。(ビルドコマンドはほぼ一緒なので割愛)
```shell
# Rustで直接生成したバイナリ
$ time -p  target/release/fib > /dev/null
real 3.61
user 3.56
sys 0.00

# ちなみにWasmをそのまま実行した場合
$ time -p wasmer ./target/wasm32-wasi/release/fib.wasm > /dev/null
real 5.13
user 5.17
sys 0.04

# Wasmから生成したバイナリ (Singlepass)
$ time -p ./target/wasm32-wasi/release/fib-singlepass > /dev/null
real 12.30
user 10.71
sys 0.08

# Wasmから生成したバイナリ (Cranelift)
$ time -p ./target/wasm32-wasi/release/fib > /dev/null
real 4.77
user 4.70
sys 0.02

# Wasmから生成したバイナリ (LLVM)
$ time -p ./target/wasm32-wasi/release/fib-llvm > /dev/null
real 2.95
user 2.92
sys 0.02
```
Wasmから生成した場合だと少しだけ遅くなるみたい。
だがWasmバイナリを実行した場合よりは、変換後のネイティブバイナリの方が速くなるのは言うまでもない。
`Singlepass`によるコンパイルの場合は、公式の記載通りこの中ではダントツで遅い。
`LLVM`の場合はダントツで速く、ここはさすがの安定感を感じる。(もちろん実行環境の条件にもよるところもあるだろうが)

## コンパイル速度は？
ついでにコンパイル時間も気になったので、それぞれのコンパイラで計測してみた。
```shell
# Singlepassでのコンパイル
$ time -p wasmer create-exe ./target/wasm32-wasi/release/hello-wasm.wasm -o ./target/wasm32-wasi/release/hello-wasm-singlepass --singlepass > /dev/null
real 3.06
user 2.44
sys 0.63

# Craneliftでのコンパイル
$ time -p wasmer create-exe ./target/wasm32-wasi/release/hello-wasm.wasm -o ./target/wasm32-wasi/release/hello-wasm-cranelift
real 3.08
user 2.57
sys 0.62

# LLVMでのコンパイル
$ time -p wasmer create-exe ./target/wasm32-wasi/release/hello-wasm.wasm -o ./target/wasm32-wasi/release/hello-wasm-llvm --llvm
real 4.13
user 6.64
sys 0.61
```
これもだいたい公式の記載どおりで、一番単純な`Singlepass`でのコンパイルが最もコンパイルが速い。
`LLVM`はこの中では最もコンパイルに時間がかかるが、実行時のパフォーマンスはダントツな所も本番向きというのは納得。

# クロスコンパイルもできるらしい
なんとクロスコンパイルもできるらしく、例えば「Linuxホスト上でWasmバイナリからexe形式のバイナリを生成する」みたいなこともできるみたい。
> In Wasmer 3.0 we used the power of Zig for doing cross-compilation from the C glue code into other machines.

クロスコンパイルには[Zig](https://ziglang.org/)を使っており、たしかにところどころコードにも`Zig`の記載が見える。
(最近`Zig`の名前を聞くことも増えてきた気がする。)

# 最後に
ここまで`Wasmer`のWasmバイナリからネイティブバイナリの生成を追ってきて、どのように生成しているのかなんとなく把握できてとても楽しめた。(実際「美味しいネイティブバイナリ」が作れたのかという所ではあるが)
ヘッダーファイルを生成するところとか、思ったより力技でこれはこれで良い。
よくよく考えたら、WasmによっていろいろとWebブラウザに移植されどんどん集約されていっている潮流がある中で、個人的にある意味逆行しているとも捉えられそうな気もする。(自分がわかってないだけかもしれないが)
実際[SQLite3 WASM/JS](https://www.publickey1.jp/blog/22/sqlite3_wasmjssqlite_340websqlite.html)で、SQLite3とかもWebブラウザ対応が進められていたりというのもあるし。
この辺について`Wasmer`の技術戦略的なところももう少し真面目に調べてみたいと思った。

# Appendix
- [`Wasmer 3.0`のアナウンス](https://wasmer.io/posts/announcing-wasmer-3.0)
- [`Zig`](https://ziglang.org/)(クロスコンパイルに使われている)
- [SQLite3 WASM/JS](https://www.publickey1.jp/blog/22/sqlite3_wasmjssqlite_340websqlite.html)
