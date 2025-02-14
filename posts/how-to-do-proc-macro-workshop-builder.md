# proc-macro-workshopのbuilderをどう進めたらよいか

Rustの手続き型マクロを学ぶのに[Rust Latam: procedural macros workshop](https://github.com/dtolnay/proc-macro-workshop)のbuilderから始めるのが良いと知ったが、builderの説明を読んでもどう進めてよいか全くわからなかった私のような人に向けて文章を残します。私自身もマクロの初心者なので参考程度にしてください。

**目次**
- [はじめに](#はじめに)
  - [手続き型マクロのライブラリの作成](#手続き型マクロのライブラリの作成)
  - [手続き型マクロの仕組み](#手続き型マクロの仕組み)
  - [戻り値のTokenStreamの作成](#戻り値のtokenstreamの作成)
  - [入力のTokenStreamの解析](#入力のtokenstreamの解析)
  - [エラーハンドリング](#エラーハンドリング)
- [builder](#builder)
  - [01-parse](#01-parse)
  - [02-create-builders](#02-create-builders)
  - [03-call-setters](#03-call-setters)
  - [04-call-build](#04-call-build)
  - [05-method-chaining](#05-method-chaining)
  - [06-optional-field](#06-optional-field)
  - [07-repeated-field](#07-repeated-field)
  - [08-unrecognized-attribute](#08-unrecognized-attribute)
  - [09-redefined-prelude-types](#09-redefined-prelude-types)
  - [私が書いたコード](#私が書いたコード)
- [さいごに](#さいごに)

## はじめに

### 手続き型マクロのライブラリの作成

ワークショップに取り組む前に、手続き型マクロを含むライブラリの書き方を知っておいた方が取り組みやすいと思いました。

```shell
cargo new --lib hello-proc-macro
cd hello-proc-macro
```

手続き型マクロのライブラリを作成するには`Cargo.toml`に下記を追加する必要があります。
```toml
[lib]
proc-macro = true
```

手続き型マクロを書くのにほぼ必須と言ってよい3つのクレートがあるので追加します。各クレートの説明は使用時に行います。
```shell
cargo add syn --features full,extra-traits
cargo add quote
cargo add proc-macro2
```

どのようにマクロが展開されるのかを見るために[cargo expand](https://github.com/dtolnay/cargo-expand)をインストールします。
```shell
cargo install cargo-expand
```

手続き型マクロのライブラリでは下記のエラーメッセージの通り通常の関数はexportできません。`src/lib.rs`の内容を全て消しましょう。
```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo build
   Compiling hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
error: `proc-macro` crate types currently cannot export any items other than functions tagged with `#[proc_macro]`, `#[proc_macro_derive]`, or `#[proc_macro_attribute]`
 --> src/lib.rs:1:1
  |
1 | pub fn add(left: u64, right: u64) -> u64 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

代わりに`src/lib.rs`に下記の内容を書きます。ワークショップの`lib.rs`と同じコードです。
```rust
use proc_macro::TokenStream;

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = input;

    unimplemented!()
}
```

テスト用に`src/main.rs`を作成し、下記の内容を書きます。ワークショップの1番目のテストとほぼ同じコードです。
```rust
use hello_proc_macro::Builder;

#[derive(Builder)]
pub struct Command {
    executable: String,
    args: Vec<String>,
}

fn main() {}
```

ビルドしたときに下記のエラーメッセージが出力されれば、手続き型マクロを書く準備が完了です😊
```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo build
   Compiling hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
error: proc-macro derive panicked
 --> src/main.rs:3:10
  |
3 | #[derive(Builder)]
  |          ^^^^^^^
  |
  = help: message: not implemented
```

### 手続き型マクロの仕組み

手続き型マクロの関数は`proc_macro::TokenStream`を受け取り`TokenStream`を返しています。まずは`TokenStream`とは何なのかを調べてみましょう。`Debug`トレイトを実装しているので、`dbg!`などで表示ができます。
```rust
use proc_macro::TokenStream;

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = input;

    dbg!(&input);

    unimplemented!()
}
```

マクロの展開はコンパイル時に行われるので`cargo check`を実行すると関数が実行され、`TokenStream`の内容が表示されます。出力結果は長いので折りたたんでいます。
<details>
<summary>`cargo check`の出力結果</summary>
  
```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo check
    Checking hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
[src/lib.rs:7:5] &input = TokenStream [
    Ident {
        ident: "pub",
        span: #0 bytes(51..54),
    },
    Ident {
        ident: "struct",
        span: #0 bytes(55..61),
    },
    Ident {
        ident: "Command",
        span: #0 bytes(62..69),
    },
    Group {
        delimiter: Brace,
        stream: TokenStream [
            Ident {
                ident: "executable",
                span: #0 bytes(76..86),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(86..87),
            },
            Ident {
                ident: "String",
                span: #0 bytes(88..94),
            },
            Punct {
                ch: ',',
                spacing: Alone,
                span: #0 bytes(94..95),
            },
            Ident {
                ident: "args",
                span: #0 bytes(100..104),
            },
            Punct {
                ch: ':',
                spacing: Alone,
                span: #0 bytes(104..105),
            },
            Ident {
                ident: "Vec",
                span: #0 bytes(106..109),
            },
            Punct {
                ch: '<',
                spacing: Alone,
                span: #0 bytes(109..110),
            },
            Ident {
                ident: "String",
                span: #0 bytes(110..116),
            },
            Punct {
                ch: '>',
                spacing: Joint,
                span: #0 bytes(116..117),
            },
            Punct {
                ch: ',',
                spacing: Alone,
                span: #0 bytes(117..118),
            },
        ],
        span: #0 bytes(70..120),
    },
]
```

</details>

出力結果を見ると`TokenStream`とは[TokenTree](https://doc.rust-lang.org/proc_macro/enum.TokenTree.html)のリストであることがわかります。`TokenStream`は`FromIterator<TokenTree>`を実装しているので、下記のように`TokenStream`を作成することができます。
```rust
use proc_macro::{Delimiter, Group, Ident, Span, TokenStream, TokenTree};

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = input;

    let tt = [
        TokenTree::Ident(Ident::new("fn", Span::call_site())),
        TokenTree::Ident(Ident::new("hello", Span::call_site())),
        TokenTree::Group(Group::new(Delimiter::Parenthesis, TokenStream::new())),
        TokenTree::Group(Group::new(Delimiter::Brace, TokenStream::new())),
    ];

    tt.into_iter().collect()
}
```

> [!NOTE]
> ネタバレになってしまいますが、今後このような書き方はすることがないので覚える必要はありません。

`cargo expand`を実行して、マクロ展開後のソースコードを見てみましょう。
```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo expand --bin hello-proc-macro
    Checking hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.05s

#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
use hello_proc_macro::Builder;
pub struct Command {
    executable: String,
    args: Vec<String>,
}
fn hello() {}
fn main() {}
```

`#[derive(..)]`を付けた構造体の下に、`fn hello() {}`という関数が出力されましたね😊このように戻り値の`TokenStream`を展開した結果が構造体の下に追加されます。`impl XXX for Command { ... }`のように構造体に対してトレイトを実装する`TokenStream`を作成すれば、`Debug`や`Clone`と同様に構造体に対して自動でトレイトを実装することができるのです。思っていたよりもシンプルな仕組みだと感じたのではないでしょうか？

> [!NOTE]
> 手続き型マクロで定義できるマクロは"Derive macro", "Function-like macro", "Attribute macro"の3種類があります。戻り値の`TokenStream`がソースコードに「追加」されるのは"Derive macro"のみです。他の2つでは「置換」されます。

### 戻り値の`TokenStream`の作成

戻り値の`TokenStream`を作成するのに`TokenTree`をたくさん書くのはさすがに大変すぎるので、便利なクレートが用意されています。それが`quote`クレートです。

```rust
use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let _ = input;

    quote! {
        fn hello() {}
    }
    .into()
}
```

このように`quote!`の中に通常のRustコードを書くだけで、そのコードを構文解析して前の例と同じ`TokenStream`を作成してくれます。

> [!NOTE]
> `quote!`マクロが返す`TokenStream`は`proc_macro2::TokenStream`です。`.into()`により`proc_macro2::TokenStream`から`proc_macro::TokenStream`へ変換を行っています。`proc_macro2`クレートは`proc_macro`クレートのラッパーです。なぜ`proc_macro2`クレートが存在するのかは、他の方が書いた説明を参照してください。

### 入力の`TokenStream`の解析

引数で与えられた`TokenStream`をもとに戻り値の`TokenStream`を作成すればよいということになりますが、前に見たように`TokenStream`は単純な`TokenTree`のリストなので、戻り値の`TokenStream`を出力するのに必要な情報を取り出すことが難しいです。これを行うのに大変便利なのが`syn`クレートです。

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    dbg!(&input);

    quote! {
        fn hello() {}
    }
    .into()
}
```

`parse_macro_input!(input as DeriveInput)`の部分に注目してください。ここで`TokenStream`を解析して、[DeriveInput](https://docs.rs/syn/latest/syn/struct.DeriveInput.html)という構文木を出力しています。`cargo check`でデバッグ表示してみましょう。
<details>
<summary>DeriveInputのデバッグ表示</summary>

```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo check
    Checking hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
[src/lib.rs:9:5] &input = DeriveInput {
    attrs: [],
    vis: Visibility::Public(
        Pub,
    ),
    ident: Ident {
        ident: "Command",
        span: #0 bytes(62..69),
    },
    generics: Generics {
        lt_token: None,
        params: [],
        gt_token: None,
        where_clause: None,
    },
    data: Data::Struct {
        struct_token: Struct,
        fields: Fields::Named {
            brace_token: Brace,
            named: [
                Field {
                    attrs: [],
                    vis: Visibility::Inherited,
                    mutability: FieldMutability::None,
                    ident: Some(
                        Ident {
                            ident: "executable",
                            span: #0 bytes(76..86),
                        },
                    ),
                    colon_token: Some(
                        Colon,
                    ),
                    ty: Type::Path {
                        qself: None,
                        path: Path {
                            leading_colon: None,
                            segments: [
                                PathSegment {
                                    ident: Ident {
                                        ident: "String",
                                        span: #0 bytes(88..94),
                                    },
                                    arguments: PathArguments::None,
                                },
                            ],
                        },
                    },
                },
                Comma,
                Field {
                    attrs: [],
                    vis: Visibility::Inherited,
                    mutability: FieldMutability::None,
                    ident: Some(
                        Ident {
                            ident: "args",
                            span: #0 bytes(100..104),
                        },
                    ),
                    colon_token: Some(
                        Colon,
                    ),
                    ty: Type::Path {
                        qself: None,
                        path: Path {
                            leading_colon: None,
                            segments: [
                                PathSegment {
                                    ident: Ident {
                                        ident: "Vec",
                                        span: #0 bytes(106..109),
                                    },
                                    arguments: PathArguments::AngleBracketed {
                                        colon2_token: None,
                                        lt_token: Lt,
                                        args: [
                                            GenericArgument::Type(
                                                Type::Path {
                                                    qself: None,
                                                    path: Path {
                                                        leading_colon: None,
                                                        segments: [
                                                            PathSegment {
                                                                ident: Ident {
                                                                    ident: "String",
                                                                    span: #0 bytes(110..116),
                                                                },
                                                                arguments: PathArguments::None,
                                                            },
                                                        ],
                                                    },
                                                },
                                            ),
                                        ],
                                        gt_token: Gt,
                                    },
                                },
                            ],
                        },
                    },
                },
                Comma,
            ],
        },
        semi_token: None,
    },
}
```

</details>

あとはここから必要な情報を取り出して、`quote!`マクロを使って戻り値の`TokenStream`を作成するだけです😊例として`Command`構造体からフィールド名を取り出してみましょう。通常のRustコードを書くときと同様に、パターンマッチをしていきます。

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DataStruct, DeriveInput, Fields, FieldsNamed};

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    let Data::Struct(DataStruct {
        fields: Fields::Named(FieldsNamed { named, .. }),
        ..
    }) = input.data
    else {
        panic!("#[derive(Builder)]が使えるのは名前付きの構造体のみです。");
    };

    let fields = named
        .iter()
        .map(|field| field.ident.as_ref().unwrap())
        .collect::<Vec<_>>();

    let ret = quote! {
        #(
            fn #fields() {}
        )*
    };

    dbg!(&ret);

    ret.into()
}
```

> [!WARNING]
> 上記のコードではパターンマッチに失敗したときに`panic!`していますが、これは正しいエラーハンドリングではありません。

`fields`変数の型は`Vec<&syn::Ident>`です。`quote!`マクロでは`#(...)*`という構文により、`Iterator`の要素ごとにマクロを展開することができます。「あれ、なんか出力結果がおかしいな」と思ったら上記のコードのように戻り値の`TokenStream`をデバッグ表示してみて、思った通りの結果になっているかを確認してみましょう。

```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo expand --bin hello-proc-macro
    Checking hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
[src/lib.rs:28:5] &ret = TokenStream [
    Ident {
        ident: "fn",
        span: #5 bytes(41..48),
    },
    Ident {
        ident: "executable",
        span: #0 bytes(76..86),
    },
    Group {
        delimiter: Parenthesis,
        stream: TokenStream [],
        span: #5 bytes(41..48),
    },
    Group {
        delimiter: Brace,
        stream: TokenStream [],
        span: #5 bytes(41..48),
    },
    Ident {
        ident: "fn",
        span: #5 bytes(41..48),
    },
    Ident {
        ident: "args",
        span: #0 bytes(100..104),
    },
    Group {
        delimiter: Parenthesis,
        stream: TokenStream [],
        span: #5 bytes(41..48),
    },
    Group {
        delimiter: Brace,
        stream: TokenStream [],
        span: #5 bytes(41..48),
    },
]
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.06s

#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
use hello_proc_macro::Builder;
pub struct Command {
    executable: String,
    args: Vec<String>,
}
fn executable() {}
fn args() {}
fn main() {}
```

想定通り、`fn executable() {}`と`fn args() {}`の2行が出力されましたね🎉ここまでの内容がわかればワークショップを開始することができるのではないかと思います。はじめに、の最後としてエラーハンドリングについて少し説明します。

### エラーハンドリング

正しいエラーハンドリングが何かは私もわかっていないのですが、いくつか見た参考資料をまとめると下記のようにするのがよいのではないでしょうか。

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DataStruct, DeriveInput, Fields, FieldsNamed};

#[proc_macro_derive(Builder)]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    expand(input)
        .unwrap_or_else(|e| e.into_compile_error())
        .into()
}

fn expand(input: DeriveInput) -> syn::Result<proc_macro2::TokenStream> {
    let Data::Struct(DataStruct {
        fields: Fields::Named(FieldsNamed { named, .. }),
        ..
    }) = input.data
    else {
        return Err(syn::Error::new_spanned(
            input,
            "#[derive(Builder)]が使えるのは名前付きの構造体のみです。",
        ));
    };

    let fields = named
        .iter()
        .map(|field| field.ident.as_ref().unwrap())
        .collect::<Vec<_>>();

    Ok(quote! {
        #(
            fn #fields() {}
        )*
    })
}
```

`fn derive(input: TokenStream) -> TokenStream`という型をみて、戻り値が`Result`ではないことに疑問を持った人もいるのではないでしょうか。手続き型マクロでは`TokenStream`に`compile_error!`マクロを出力することでエラーを表現します。上記のコードでは`syn::Error::into_compile_error()`というメソッドを用いて、エラーを表現しています。

> [!NOTE]
> `compile_error!`マクロはRustの標準ライブラリで定義されています。

マクロを展開するロジックを別の関数に分けて、戻り値を`syn::Result<proc_macro2::TokenStream>`としています。パターンマッチなどでエラーとなった場合は`syn::Error`を返します。`proc_macro::Span`はソースコードの範囲を表しています。`syn::Error`の作成時には`Span`を設定することでマクロがどこでエラーになったのか詳細なエラーメッセージを表示することができます。

`src/main.rs`を書き換えて名前付きではない構造体に`#[derive(Builder)]`を付けてエラーを発生させてみましょう。

```rust
use hello_proc_macro::Builder;

#[derive(Builder)]
pub struct Command;

fn main() {}
```

```shell
lvs7k@wsl2:~/hello-proc-macro$ cargo check
    Checking hello-proc-macro v0.1.0 (/home/lvs7k/hello-proc-macro)
error: #[derive(Builder)]が使えるのは名前付きの構造体のみです。
 --> src/main.rs:4:1
  |
4 | pub struct Command;
  | ^^^^^^^^^^^^^^^^^^^
```

指定したエラーメッセージが表示されて、`Span`の範囲のソースコードでエラーになったことがわかります。

それでは実際にワークショップのbuilderに取り組んでみましょう😊

## builder

### 01-parse

builderの`src/lib.rs`には`unimplemented!()`が書かれているためエラーとなってしまいます。このテストではとりあえずエラーにならなければよいので、空の`TokenStream`を返しましょう。また、テストのコメントに書かれている通り入力の`TokenStream`を`parse_macro_input!`で`DeriveInput`構文木へと変換して中身を見てみましょう。

### 02-create-builders

このテストはいきなりちょっと難しいです😭もし出来なかったとしても落ち込まなくてよいと思います😊目標は`Command`構造体の後に下記のようなコードを出力することです。

```rust
pub struct CommandBuilder {
    executable: Option<String>,
    args: Option<Vec<String>>,
    env: Option<Vec<String>>,
    current_dir: Option<String>,
}

impl Command {
    pub fn builder() -> CommandBuilder {
        CommandBuilder {
            executable: None,
            args: None,
            env: None,
            current_dir: None,
        }
    }
}
```

だからといって`quote!`に直接このコードを書いてはいけません。`DeriveInput`から`Command`構造体のフィールド名とその型を取得して、`quote!`を使いましょう。`CommandBuilder`という識別子を作るには、[syn::IdentのExample](https://docs.rs/syn/latest/syn/struct.Ident.html#examples)や[quoteのformat_ident!](https://docs.rs/quote/latest/quote/macro.format_ident.html)が参考になります。

### 03-call-setters

テスト02で作成した`CommandBuilder`に値を設定するメソッドを定義しましょう。

### 04-call-build

`CommandBuilder`に`build()`というメソッドを追加しましょう。`CommandBuilder`のフィールドに設定された値から`Command`を作成します。

`CommandBuilder`のフィールドに`None`が残っていたらエラーにしましょう。エラーの型については重要ではないので`build()`メソッドの戻り値は`Result<Command, Box<dyn std::error::Error>>`で十分です。`std::error::Error`は`From<String>`を実装しているので、`String`を返せばよいと思います。

### 05-method-chaining

このテストは03と04が正しく実装されていれば問題ありません。テスト03で作成したメソッドは`fn executable(&mut self, executable: String) -> &mut Self`のように`&mut Self`を返しているので、下記のように書くことができて便利だよねという内容です。

```rust
let command = Command::builder()
    .executable("cargo".to_owned())
    .args(vec!["build".to_owned(), "--release".to_owned()])
    .env(vec![])
    .current_dir("..".to_owned())
    .build()
    .unwrap();
```

### 06-optional-field

このテストでは`Command`のフィールドの中に`Option<String>`が存在します。やりたいことは、フィールドの型が`Option`の場合は`CommandBuilder`のフィールドが未設定(`None`)の状態でも`build()`メソッドで`Command`が作成できるようにすることです。具体的には下記の事項に取り組みましょう。

- フィールドの型が`Option<String>`の場合、
  - 今までの実装だと対応する`Builder`のフィールドの型は`Option<Option<String>>`になってしまうが、`Option<String>`になるようにする。
  - `Builder`に実装するセッターメソッドの引数を`Option<String>`ではなく`String`にする。
  - `build()`メソッドで`Command`を作成する際に、`Builder`のフィールドの値をクローンして設定する。

難しいというか面倒くさいのはテストのコメントにも書かれているように`Option<String>`から`String`の部分を取り出すところです。頑張ってパターンマッチしていきましょう😭

### 07-repeated-field

難易度的にはこのテストが実質このワークショップの最後のテストだと思います、難しいですが頑張りましょう😊

下記のコードのように、フィールドに対して属性が設定されています。

```rust
#[derive(Builder)]
pub struct Command {
    executable: String,
    #[builder(each = "arg")]
    args: Vec<String>,
    #[builder(each = "env")]
    env: Vec<String>,
    current_dir: Option<String>,
}
```

やりたいことは例えば`#[builder(each = "arg")]`の場合、`CommandBuilder`のセッターメソッドとして`arg()`という`Vec`に要素を1つ追加するメソッドを定義することです。`args()`という`Vec`を一括で設定するセッターメソッドも残しておかなければなりません。`env`のようにフィールド名と同じ名前が指定されている場合は、要素を1つ追加するメソッドの定義のみでよいです。また、`#[builder(each = "...")]`が付いているフィールドの型は`Vec`であることを前提としてよいです。

- フィールドの型が`Vec<String>`の場合、
  - `Builder`のフィールドの型は`Option<Vec<String>>`ではなく`Vec<String>`にする。
  - 一括で設定するセッターメソッドの引数は`Vec<String>`として、引数の値をそのまま`Builder`に設定する。
  - 1要素を追加するセッターメソッドの引数は`String`として、引数の値を`Vec`へ`push`する。
  - `build()`メソッドで`Command`を作成する際に、`Builder`のフィールドの値をクローンして設定する。

### 08-unrecognized-attribute

`#[builder(eac = "arg")]`のように`each`以外だった場合はエラーとなるようにします。下記のように属性部分に下線が引かれるように適切に`Span`を設定しましょう。

```shell
error: expected `builder(each = "...")`
  --> tests/08-unrecognized-attribute.rs:22:7
   |
22 |     #[builder(eac = "arg")]
   |       ^^^^^^^^^^^^^^^^^^^^
```

### 09-redefined-prelude-types

マクロ展開先のソースコードで例えば`type Option = ();`のように`Option`が別の意味になってしまっていても、マクロが動作するようにします。具体的にはコメントにあるように`Option`を`std::option::Option`のように絶対パスで指定すればよいです。

### 私が書いたコード

Rust初心者のため綺麗なコードではないと思いますが、私がワークショップにチャレンジした際のコードを記載します。参考になれば幸いです😊

```rust
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::{format_ident, quote};
use syn::{
    parse_macro_input, Attribute, Data, DataStruct, DeriveInput, Fields, FieldsNamed,
    GenericArgument, Ident, LitStr, PathArguments, PathSegment, Type, TypePath,
};

#[proc_macro_derive(Builder, attributes(builder))]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // syn::Resultがエラーだった場合は、`into_compile_error()`でTokenStreamに変換する。
    expand(input)
        .unwrap_or_else(|e| e.into_compile_error())
        .into()
}

fn expand(input: DeriveInput) -> syn::Result<TokenStream2> {
    let struct_name = &input.ident;
    let builder_name = format_ident!("{}Builder", struct_name);

    // #[derive(Builder)]の対象が名前付きの構造体ではない場合はエラー。
    let Data::Struct(DataStruct {
        fields: Fields::Named(fields_named),
        ..
    }) = input.data
    else {
        return Err(syn::Error::new_spanned(
            input,
            "Builder trait can only be derived for a named struct.",
        ));
    };

    let builder_struct_fields = expand_builder_struct_fields(&fields_named)?;
    let builder_default_fields = expand_builder_default_fields(&fields_named)?;
    let builder_methods = expand_builder_methods(&fields_named)?;
    let build_method = expand_build_method(struct_name, &fields_named)?;

    // 今回は使用しないが、#[derive(Builder)]を付けた構造体の型パラメータやwhere句部分を取り出す際に使用する。
    // let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

    Ok(quote! {
        pub struct #builder_name {
            #builder_struct_fields
        }

        impl #struct_name {
            pub fn builder() -> #builder_name {
                #builder_name {
                    #builder_default_fields
                }
            }
        }

        impl #builder_name {
            #builder_methods
            #build_method
        }
    })
}

// Builder構造体のフィールド部分のTokenStreamを作成する。
fn expand_builder_struct_fields(fields: &FieldsNamed) -> syn::Result<TokenStream2> {
    let mut ret = TokenStream2::new();

    for field in &fields.named {
        let field_name = field.ident.as_ref().unwrap();
        let ty = &field.ty;

        if get_types_argument(&field.ty, "Option").is_some()
            || get_types_argument(&field.ty, "Vec").is_some()
        {
            // 元の構造体のフィールドの型がOptionかVecならば、そのままBuilder構造体のフィールドとする。
            ret.extend(quote! {
                #field_name: #ty,
            });
        } else {
            // それ以外の場合はOptionにする。
            ret.extend(quote! {
                #field_name: std::option::Option<#ty>,
            });
        }
    }

    Ok(ret)
}

// Option<...>とVec<...>の...の部分を取り出す。
fn get_types_argument<'a>(typ: &'a Type, ident_str: &'_ str) -> Option<&'a GenericArgument> {
    let Type::Path(TypePath { path, .. }) = typ else {
        return None;
    };

    let PathSegment {
        ident,
        arguments: PathArguments::AngleBracketed(abga),
    } = path.segments.first()?
    else {
        return None;
    };

    if ident != ident_str {
        return None;
    }

    Some(&abga.args[0])
}

// `builder()`メソッドの戻り値の構造体のフィールド部分のTokenStreamを作成する。
fn expand_builder_default_fields(fields: &FieldsNamed) -> syn::Result<TokenStream2> {
    let mut ret = TokenStream2::new();

    for field in &fields.named {
        let field_name = field.ident.as_ref().unwrap();

        if let Some(_arg) = get_types_argument(&field.ty, "Vec") {
            ret.extend(quote! {
                #field_name: Vec::new(),
            });
        } else {
            ret.extend(quote! {
                #field_name: std::option::Option::None,
            });
        }
    }

    Ok(ret)
}

// Builder構造体のフィールドを設定するメソッド定義部分のTokenStreamを作成する。
fn expand_builder_methods(fields: &FieldsNamed) -> syn::Result<TokenStream2> {
    let mut ret = TokenStream2::new();

    for field in &fields.named {
        let field_name = field.ident.as_ref().unwrap();
        let ty = &field.ty;

        // 元の構造体のフィールドの型がOption<...>の場合はメソッドの引数に...型を受け取り、フィールドに設定する。
        if let Some(arg) = get_types_argument(&field.ty, "Option") {
            ret.extend(quote! {
                fn #field_name(&mut self, #field_name: #arg) -> &mut Self {
                    self.#field_name = std::option::Option::Some(#field_name);
                    self
                }
            });
            continue;
        }

        // `builder(each = "...")`の"..."の部分のLitStrを取得する。
        if let Some(each_method_name) = get_builder_each_attribute(&field.attrs)? {
            // `builder(each = "...")`の"..."の部分のLitStrより、フィールド設定メソッドのIdentを作成する。
            let each_method_name = format_ident!("{}", each_method_name.value());

            if let Some(arg) = get_types_argument(&field.ty, "Vec") {
                // 元の構造体のフィールドの型がVecの場合は、Builder構造体のフィールドのVecに1要素をプッシュするメソッドを作成する。
                ret.extend(quote! {
                    fn #each_method_name(&mut self, #each_method_name: #arg) -> &mut Self {
                        self.#field_name.push(#each_method_name);
                        self
                    }
                });
            } else {
                // #[builder(each = "...")]を付けることができるフィールドはVecのみ。
                return Err(syn::Error::new_spanned(
                    &field.ty,
                    "a field with builder attribute must have the type Vec.",
                ));
            }

            // eachで指定した名称が構造体のフィールド名と同じ場合は、Vecに1要素をプッシュするメソッドのみを作成する。
            // 下のコードのフィールドを一括設定するメソッドを定義しないようにcontinueする。
            if field_name == &each_method_name {
                continue;
            }
        }

        if let Some(_arg) = get_types_argument(&field.ty, "Vec") {
            // 元の構造体のフィールドの型がVecの場合で「eachが指定されていない」or「eachで指定された名称とフィールド名が異なる」場合は、
            // メソッドの引数で受け取ったVecをそのままBuilderのフィールドに設定する。
            ret.extend(quote! {
                fn #field_name(&mut self, #field_name: #ty) -> &mut Self {
                    self.#field_name = #field_name;
                    self
                }
            });
        } else {
            // それ以外の場合はBuilderのフィールドはOptionなので、メソッドの引数の値をSomeで設定する。
            ret.extend(quote! {
                fn #field_name(&mut self, #field_name: #ty) -> &mut Self {
                    self.#field_name = std::option::Option::Some(#field_name);
                    self
                }
            });
        }
    }

    Ok(ret)
}

// `builder(each = "...")`の"..."の部分のLitStrを取得する。
// Reference: syn::ParseNestedMeta.value
fn get_builder_each_attribute(attrs: &[Attribute]) -> syn::Result<Option<LitStr>> {
    let mut res = None;

    for attr in attrs {
        if attr.path().is_ident("builder") {
            attr.parse_nested_meta(|meta| {
                if meta.path.is_ident("each") {
                    let value = meta.value()?;
                    let s: LitStr = value.parse()?;
                    res = Some(s);
                    Ok(())
                } else {
                    Err(meta.error("expected `builder(each = \"...\")`"))
                }
            })?;
            break;
        }
    }

    Ok(res)
}

// impl ...Builderのbuild()メソッド定義のTokenStreamを作成する。
fn expand_build_method(struct_name: &Ident, fields: &FieldsNamed) -> syn::Result<TokenStream2> {
    let mut return_struct_fields = TokenStream2::new();

    for field in &fields.named {
        let field_name = field.ident.as_ref().unwrap();

        if get_types_argument(&field.ty, "Option").is_some()
            || get_types_argument(&field.ty, "Vec").is_some()
        {
            // 元の構造体のフィールドの型がOptionかVecならばBuilder構造体のフィールドの型も同じ型なので、クローンして設定する。
            return_struct_fields.extend(quote! {
                #field_name: self.#field_name.clone(),
            });
        } else {
            // 元の構造体のフィールドがOptionかVec以外ならばBuilder構造体のフィールドの型はOptionなので、
            // Optionの中身を取り出して設定する。OptionがNoneならばエラー。
            return_struct_fields.extend(quote! {
                #field_name: self.#field_name.as_ref().cloned().ok_or(format!("{} is not yet set.", stringify!(#field_name)))?,
            });
        }
    }

    Ok(quote! {
        pub fn build(&mut self) -> std::result::Result<#struct_name, std::boxed::Box<dyn std::error::Error>> {
            Ok(#struct_name {
                #return_struct_fields
            })
        }
    })
}
```

## さいごに

Rustについてわかりやすい記事を書いてくださっている皆さんに感謝しています。皆さんの記事を見ているうちに私も学んだことを文章にして残したいという気持ちが高まり、今回初めてこのようなものを書いてみました。私はRust上級者でも頭が良いわけでもありませんが、Rust学習者の中には私と同じようなレベルの人たちもいるかもしれません。そのような人たちに私が書いたものが少しでも参考になり、何かしらRustコミュニティへ還元できたら嬉しいです😊

## 参考資料

- [Rust Latam: procedural macros workshop](https://github.com/dtolnay/proc-macro-workshop)
- [proc-macro-workshopの進め方（derive_builder編）](https://zenn.dev/taro137/articles/bb45823b83d354)
- [Rustマクロ冬期講習 Advent Calendar 2024](https://qiita.com/advent-calendar/2024/rust-macro)
