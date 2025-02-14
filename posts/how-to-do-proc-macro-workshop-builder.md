# proc-macro-workshopのbuilderをどう進めたらよいか

Rustの手続き型マクロを学ぶのに[Rust Latam: procedural macros workshop](https://github.com/dtolnay/proc-macro-workshop)のbuilderから始めるのが良いと知ったが、builderの説明を読んでもどう進めてよいか全くわからなかった私のような人に向けて文章を残します。私自身もマクロの初心者なので参考程度にしてください。

**目次**
- [はじめに](#はじめに)
  - [手続き型マクロのライブラリの作成](#手続き型マクロのライブラリの作成)
  - [手続き型マクロの仕組み](#手続き型マクロの仕組み)

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
