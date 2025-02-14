# proc-macro-workshopのbuilderをどう進めたらよいか

Rustの手続き型マクロを学ぶのに[Rust Latam: procedural macros workshop](https://github.com/dtolnay/proc-macro-workshop)のbuilderから始めるのが良いと知ったが、builderの説明を読んでもどう進めてよいか全くわからなかった私のような人に向けて文章を残します。私自身もマクロの初心者なので参考程度にしてください。

**目次**
- [はじめに](#はじめに)
  - [手続き型マクロのライブラリの作成](#手続き型マクロのライブラリの作成)
  - [手続き型マクロの仕組み](#手続き型マクロの仕組み)
  - [戻り値のTokenStreamの作成](#戻り値のtokenstreamの作成)
  - [入力のTokenStreamの解析](#入力のtokenstreamの解析)

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

手続き型マクロを書くのに必須ではないがほぼ必須と言ってよい3つのクレートがあるので追加します。各クレートの説明は使用時に行います。
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

代わりに`src/lib.rs`に下記の内容を書きます。ワークショップの`lib.rs`とほぼ同じコードです。
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

あとはここから必要な情報を取り出して、`quote!`マクロを使って戻り値の`TokenStream`を作成するだけです😊例として`Command`構造体の`arg`というフィールドを取り出してみましょう。通常のRustコードを書くときと同様に、パターンマッチをしていきます。

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
