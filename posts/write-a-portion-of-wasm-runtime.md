# WebAssemblyのランタイムの一部分を実装してみる

**目次**
- はじめに
- '1. テキスト形式とバイナリ形式
  - '1.1. サンプルプログラム
  - '1.2. 知っておきたいこと
- '2. 構文木の定義と構文解析
  - '2.1. モジュール構成
  - '2.2. 構文木の定義
  - '2.3. 構文解析
- おわりに

## はじめに

[WebAssembly Specification Release 2.0](https://webassembly.github.io/spec/core/)を読みながらランタイムを一部分だけ実装してみようと頑張った記録を残します。
仕様を正しく理解できているかわからないので、もし参考にする際は注意してください。

### 目標

- フィボナッチ数列を計算する`wasm`の検証と実行を行う`Rust`プログラムの作成

## 1. テキスト形式とバイナリ形式

### 1.1. サンプルプログラム

今回は下記のフィボナッチ数列を計算するWebAssemblyのテキスト形式をバイナリ形式に変換したものを検証・実行します。

```wat
(module
  (func $fibonacci (param $n i32) (result i32)
    (local $prev i32)
    (local $curr i32)
    (local $i i32)

    ;; if n <= 0 then return 0
    (i32.le_s (local.get $n) (i32.const 0))
    (if
      (then
        i32.const 0
        return
      )
    )

    ;; initialize local values
    (local.set $prev (i32.const 0))
    (local.set $curr (i32.const 1))
    (local.set $i (i32.const 1))

    ;; a structured instruction can consume input and produce output
    (block $mainloop (result i32)
      (block $break
        (loop $continue
          ;; if $i == $n then break
          (i32.eq (local.get $i) (local.get $n))
          br_if $break

          ;; push $curr on the stack
          (local.get $curr)
          ;; $curr := $prev + $curr
          (local.set $curr
            (i32.add (local.get $prev) (local.get $curr))
          )
          ;; set old $curr to $prev
          local.set $prev

          ;; $i += 1
          (local.set $i
            (i32.add (local.get $i) (i32.const 1))
          )

          br $continue
        )
      )
      local.get $curr
    )
  )

  (export "fibonacci" (func $fibonacci))
)
```

このテキスト形式を[wasm-tools](https://github.com/bytecodealliance/wasm-tools)でバイナリ形式に変換し、`dump`コマンドで中身を見た結果がこちらです。

```shell
lvs7k@wsl2:~/wasmpl-rs$ wasm-tools parse example/fibonacci.wat -o example/fibonacci.wasm
lvs7k@wsl2:~/wasmpl-rs$ wasm-tools dump example/fibonacci.wasm
  0x0 | 00 61 73 6d | version 1 (Module)
      | 01 00 00 00
  0x8 | 01 06       | type section
  0xa | 01          | 1 count
--- rec group 0 (implicit) ---
  0xb | 60 01 7f 01 | [type 0] SubType { is_final: true, supertype_idx: None, composite_type: CompositeType { inner: Func(FuncType { params: [I32], results: [I32] }), shared: false } }
      | 7f
 0x10 | 03 02       | func section
 0x12 | 01          | 1 count
 0x13 | 00          | [func 0] type 0
 0x14 | 07 0d       | export section
 0x16 | 01          | 1 count
 0x17 | 09 66 69 62 | export Export { name: "fibonacci", kind: Func, index: 0 }
      | 6f 6e 61 63
      | 63 69 00 00
 0x23 | 0a 43       | code section
 0x25 | 01          | 1 count
============== func 0 ====================
 0x26 | 41          | size of function
 0x27 | 01          | 1 local blocks
 0x28 | 03 7f       | 3 locals of type I32
 0x2a | 20 00       | local_get local_index:0
 0x2c | 41 00       | i32_const value:0
 0x2e | 4c          | i32_le_s
 0x2f | 04 40       | if blockty:Empty
 0x31 | 41 00       | i32_const value:0
 0x33 | 0f          | return
 0x34 | 0b          | end
 0x35 | 41 00       | i32_const value:0
 0x37 | 21 01       | local_set local_index:1
 0x39 | 41 01       | i32_const value:1
 0x3b | 21 02       | local_set local_index:2
 0x3d | 41 01       | i32_const value:1
 0x3f | 21 03       | local_set local_index:3
 0x41 | 02 7f       | block blockty:Type(I32)
 0x43 | 02 40       | block blockty:Empty
 0x45 | 03 40       | loop blockty:Empty
 0x47 | 20 03       | local_get local_index:3
 0x49 | 20 00       | local_get local_index:0
 0x4b | 46          | i32_eq
 0x4c | 0d 01       | br_if relative_depth:1
 0x4e | 20 02       | local_get local_index:2
 0x50 | 20 01       | local_get local_index:1
 0x52 | 20 02       | local_get local_index:2
 0x54 | 6a          | i32_add
 0x55 | 21 02       | local_set local_index:2
 0x57 | 21 01       | local_set local_index:1
 0x59 | 20 03       | local_get local_index:3
 0x5b | 41 01       | i32_const value:1
 0x5d | 6a          | i32_add
 0x5e | 21 03       | local_set local_index:3
 0x60 | 0c 00       | br relative_depth:0
 0x62 | 0b          | end
 0x63 | 0b          | end
 0x64 | 20 02       | local_get local_index:2
 0x66 | 0b          | end
 0x67 | 0b          | end
 0x68 | 00 4a       | custom section
 0x6a | 04 6e 61 6d | name: "name"
      | 65
 0x6f | 01 0c       | function name section
 0x71 | 01          | 1 count
 0x72 | 00 09 66 69 | Naming { index: 0, name: "fibonacci" }
      | 62 6f 6e 61
      | 63 63 69
 0x7d | 02 15       | local section
 0x7f | 01          | 1 count
 0x80 | 00          | function 0 local name section
 0x81 | 04          | 4 count
 0x82 | 00 01 6e    | Naming { index: 0, name: "n" }
 0x85 | 01 04 70 72 | Naming { index: 1, name: "prev" }
      | 65 76
 0x8b | 02 04 63 75 | Naming { index: 2, name: "curr" }
      | 72 72
 0x91 | 03 01 69    | Naming { index: 3, name: "i" }
 0x94 | 03 1e       | label section
 0x96 | 01          | 1 count
 0x97 | 00          | function 0 label name section
 0x98 | 03          | 3 count
 0x99 | 01 08 6d 61 | Naming { index: 1, name: "mainloop" }
      | 69 6e 6c 6f
      | 6f 70
 0xa3 | 02 05 62 72 | Naming { index: 2, name: "break" }
      | 65 61 6b
 0xaa | 03 08 63 6f | Naming { index: 3, name: "continue" }
      | 6e 74 69 6e
      | 75 65
```

### 1.2. 知っておきたいこと

#### WebAssemblyは[スタックマシン](https://ja.wikipedia.org/wiki/%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E3%83%9E%E3%82%B7%E3%83%B3)として動作する

例えばWebAssemblyで`1 + 2`を計算するコードは下記のようになります。

```wat
i32.const 1 ;; stack: [1]
i32.const 2 ;; stack: [1][2]
i32.add     ;; stack: [3]
```

WebAssemblyのランタイムは値を保持するスタックを持っていて、それを出し入れすることで計算を進めて行きます。

#### ローカル変数はスタック上にはない

関数にはパラメータ(`param`)とローカル変数(`local`)を定義することができます。

```wat
  (func $fibonacci (param $n i32) (result i32)
    (local $prev i32)
    (local $curr i32)
    (local $i i32)
```

この2つを「パラメータ→ローカル変数の順に結合したもの」を、スタックとは別の場所で管理しています。
ローカル変数を操作するのに`local.get`や`local.set`という命令を用いますが、引数として`index`を受け取ります。
例えば今回のサンプルプログラムはパラメータが1つでローカル変数が3つなので、パラメータのインデックスは`0`、ローカル変数のインデックスは`1`~`3`となります。

```shell
 0x49 | 20 00       | local_get local_index:0
...
 0x5e | 21 03       | local_set local_index:3
```

例えばローカル変数同士を足し算したかったら、`local.get`によってローカル変数をスタックに積む必要があります。

```wat
local.get $prev
local.get $curr
i32.add
```

#### Structured Instruction

[Control Instructions](https://webassembly.github.io/spec/core/syntax/instructions.html#control-instructions)によると、`block`, `loop`, `if`の3つの命令はStructured Instructionと呼ばれています。

```
instr ::= ...
        | block blocktype instr* end
        | loop blocktype instr* end
        | if blocktype instr* else instr* end
```

この3つの命令の中ではブランチ命令(`br`, `br_if`, `br_table`)を使うことができます。
`block`と`if`の中でブランチ命令を使うと対応する`end`にジャンプし、`loop`の中だとループの先頭にジャンプします。
例えば他のプログラミング言語にあるような`break`や`continue`を行うには下記のようにします。

```wat
(block $break
  (loop $continue
    ...
    br_if $break ;; 条件を満たしたらループを抜ける
    ...
    br $continue ;; ループの先頭に戻る
  )
)
```

Rustでは`loop { .. }`と書くだけでループしますが、WebAssemblyでは`loop`の中でブランチ命令を使わないとループしないことに注意してください。

#### Structured Instructionはインプットとアウトプットを持つ

```
instr ::= ...
        | block blocktype instr* end
        | loop blocktype instr* end
        | if blocktype instr* else instr* end
```

一般的なプログラミング言語と異なり、`block`, `loop`, `if`は`blocktype`というインプットとアウトプットの型を持ちます。
例えば`i32.add`という命令は「スタックから2つの`i32`を取り出し、1つの`i32`をスタックに追加する」ので、`i32.add: [i32 i32] -> [i32]`と表現することができます。
これと同じように例えば`block`が`[i32 i32] -> [i32]`という型を持つならば、次の3つの条件を満たさなければなりません。

- `block`実行時にスタックのトップに`i32`が2つ積まれている
- `block`の終了時に（例え`br`命令でジャンプしたとしても）スタックのトップに`i32`が1つ積まれている
- 余分な値がスタック上に残っていてはならない

3つめの条件に付いて、例えば以下のようなコードはエラーになります。
スタック上に余分な`3`という値が積まれたまま残ってしまっているからです。

```wat
(module
  (func $hello (result i32)
    i32.const 1   ;; stack: [1]
    i32.const 2   ;; stack: [1][2]

    (block (param i32 i32) (result i32)
      drop        ;; stack: [1]
      drop        ;; stack: 
      i32.const 3 ;; stack: [3]
      i32.const 4 ;; stack: [3][4]
    )
  )
)
```

> [!NOTE]
> `br`や`return`といった命令は`i32.add`などと異なり`stack-polymorphic`な型を持ちます。
> そのため、上記の条件を満たさなくてもエラーとなりません。詳しくは検証のコードを書く際に説明します。

例えば下記のコードはどちらもエラーではありません。

```wat
(module
  (func $hello (result i32)
    i32.const 1   ;; stack: [1]
    i32.const 2   ;; stack: [1][2]

    (block $break (param i32 i32) (result i32)
      drop        ;; stack: [1]
      drop        ;; stack: 
      i32.const 3 ;; stack: [3]
      i32.const 4 ;; stack: [3][4]
      br $break
    )
  )
)
```

```wat
(module
  (func $hello (result i32)
    i32.const 1   ;; stack: [1]
    i32.const 2   ;; stack: [1][2]

    (block $break (param i32 i32) (result i32)
      return
    )
  )
)
```

#### ブランチ命令のジャンプ先は内側から番号が振られる

例えば下記のようなコードがバイナリ形式へと変換されたとき、

```wat
    (block $mainloop (result i32)
      (block $break
        (loop $continue
          ...
          br_if $break
          ...
          br $continue
        )
      )
    )
```

最も内側を`0`として順番に番号が振られていきます。

```wat
    (block 2 (result i32)
      (block 1
        (loop 0
          ...
          br_if 1
          ...
          br 0
        )
      )
    )
```

### 2. 構文木の定義と構文解析

#### 2.1. モジュール構成

今回のランタイムではバイナリ形式を構文解析して構文木の形にして、木をたどりながら命令を実行していくシンプルなものとします。
分かりやすさを重視するため、なるべく[WebAssembly Specification](https://webassembly.github.io/spec/core/)とソースコードを対応させていきます。

モジュール構成は下記のようにします。
この章では構文木の定義(`structure.rs`)とバイナリ形式の解析(`binary.rs`)を実装していきます。

```shell
lvs7k@wsl2:~/wasmpl-rs$ tree --gitignore
.
├── Cargo.toml
├── example
│   └── fibonacci.wat
└── src
    ├── binary.rs      # まず、これと
    ├── embedding.rs
    ├── execution.rs
    ├── interpreter.rs
    ├── lib.rs
    ├── main.rs
    ├── structure.rs   # これを作っていきます
    ├── validation
    │   └── func.rs
    └── validation.rs
```

#### 2.2. 構文木の定義(`structure.rs`)

サンプルプログラムを実行するのに必要な部分だけ定義します。

##### [Types](https://webassembly.github.io/spec/core/syntax/types.html)

```rust
// Types

use std::rc::Rc;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NumType {
    I32,
    I64,
    F32,
    F64,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ValType {
    Num(NumType),
}

pub type ResultType = Vec<ValType>;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct FuncType {
    pub input: ResultType,
    pub output: ResultType,
}
```

##### [Instructions](https://webassembly.github.io/spec/core/syntax/instructions.html)

```rust
// Instructions

#[derive(Debug)]
pub enum Instr {
    // Control Instructions
    Block {
        blocktype: BlockType,
        expr: Expr,
    },
    Loop {
        blocktype: BlockType,
        expr: Expr,
    },
    If {
        blocktype: BlockType,
        true_branch: Expr,
        false_branch: Expr,
    },
    Br {
        labelidx: Idx,
    },
    BrIf {
        labelidx: Idx,
    },
    Return,

    // Numeric Instructions
    I32Const {
        val: i32,
    },
    IBinOp {
        bit: Bit,
        op: IBinOp,
    },
    IRelOp {
        bit: Bit,
        op: IRelOp,
    },

    // Varable Instructions
    LocalGet {
        localidx: Idx,
    },
    LocalSet {
        localidx: Idx,
    },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BlockType {
    Empty,
    Idx(Idx),
    Val(ValType),
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Bit {
    B32,
    B64,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Sign {
    Signed,
    Unsigned,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum IBinOp {
    Add,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum IRelOp {
    Eq,
    Le { sign: Sign },
}

pub type Expr = Vec<Instr>;
```

##### [Modules](https://webassembly.github.io/spec/core/syntax/modules.html)

```rust
// Modules

#[derive(Debug)]
pub struct Module {
    pub types: Vec<FuncType>,
    pub funcs: Vec<Rc<Func>>,
    pub exports: Vec<Export>,
}

pub type Idx = u32;

#[derive(Debug)]
pub struct Func {
    pub type_: Idx,
    pub locals: Vec<ValType>,
    pub body: Expr,
}

#[derive(Debug)]
pub struct Export {
    pub name: String,
    pub desc: ExportDesc,
}

#[derive(Debug)]
pub enum ExportDesc {
    Func(Idx),
}
```

#### 2.3. 構文解析(`binary.rs`)

##### Parser型

構文解析には`nom`などのパーサコンビネータライブラリを使っても良いですが、WebAssemblyのバイナリ形式はシンプルなので今回は自分で書いてみます。

```rust
use std::{
    io::{Read, Seek, SeekFrom},
    rc::Rc,
};

use crate::structure::*;

#[derive(Debug)]
pub struct Parser<R> {
    pub reader: R,
}
```

構文解析に失敗したときのエラー型を下記のように定義します。

```rust
#[derive(Debug)]
pub enum ParseError {
    Io(std::io::Error),
    Utf8(std::str::Utf8Error),
    Backtrack,
    Malformed,
}

impl From<std::io::Error> for ParseError {
    fn from(value: std::io::Error) -> Self {
        Self::Io(value)
    }
}

impl From<std::str::Utf8Error> for ParseError {
    fn from(value: std::str::Utf8Error) -> Self {
        Self::Utf8(value)
    }
}
```

WebAssemblyのバイナリ形式の解析には1バイトの先読み(`peek`)ができると便利です。
先読みにはいろいろな方法があると思います。

- `Parser`に`peeked: Option<u8>`のようなフィールドを用意する
- `BufRead`トレイトを利用する
- `Seek`トレイトを利用する

`Seek`トレイトを使ったことがなかったので、今回は`R: Read + Seek`として解析を進めてみようと思います。
まずは解析を進めるのに便利なメソッドから定義していきます。

```rust
impl<R: Read + Seek> Parser<R> {
    pub fn new(reader: R) -> Self {
        Self { reader }
    }

    pub fn byte(&mut self) -> Result<u8, ParseError> {
        let mut buf = [0; 1];
        self.reader.read_exact(&mut buf)?;
        Ok(buf[0])
    }

    pub fn peek(&mut self) -> Result<u8, ParseError> {
        let b = self.byte()?;
        self.reader.seek(SeekFrom::Current(-1))?;
        Ok(b)
    }

    pub fn consume(&mut self, byte: u8) -> Result<(), ParseError> {
        if self.byte()? != byte {
            return Err(ParseError::Malformed);
        }
        Ok(())
    }
```

バイナリをあるパターンで解析しようとして失敗したらまた別のパターンでの解析をしたいことがあります。
また、失敗しても別のパターンでの解析を試さずそのままエラーを報告したいときもあります。
パーサが`Backtrack`というエラーを返した場合は、`Seek`で解析の開始地点まで巻き戻します。

```rust
    pub fn opt<T>(
        &mut self,
        mut parser: impl FnMut(&mut Parser<R>) -> Result<T, ParseError>,
    ) -> Result<Option<T>, ParseError> {
        let start_pos = self.reader.stream_position()?;
        match parser(self) {
            Ok(t) => Ok(Some(t)),
            Err(ParseError::Backtrack) => {
                self.reader.seek(SeekFrom::Start(start_pos))?;
                Ok(None)
            }
            Err(e) => Err(e),
        }
    }
```

[Vectors](https://webassembly.github.io/spec/core/binary/conventions.html#vectors)に`vec(B) ::= n:u32 (x:B)^n => x^n`と書かれています。
最初に`u32`がLEB128形式でエンコードされていて、その数値分だけ`B`が続きます。

```rust
    pub fn vec<T>(
        &mut self,
        mut parser: impl FnMut(&mut Parser<R>) -> Result<T, ParseError>,
    ) -> Result<Vec<T>, ParseError> {
        let n = self.u32()?;
        (0..n).map(|_| parser(self)).collect()
    }
```

`vec`と似たようなパターンで、最初にバイト数(`u32`)があって次にそのバイト数分だけ何かが続くパターンがあります。
`u32`を解析したあと引数のパーサを実行し、サイズが合っていなければエラーとします。

```rust
    pub fn size_prefixed<T>(
        &mut self,
        mut parser: impl FnMut(&mut Parser<R>) -> Result<T, ParseError>,
    ) -> Result<T, ParseError> {
        let size = self.u32()?;

        let start_pos = self.reader.stream_position()?;
        let ret = parser(self)?;

        if self.reader.stream_position()? - start_pos != size as u64 {
            return Err(ParseError::Malformed);
        }

        Ok(ret)
    }
```

##### [Values](https://webassembly.github.io/spec/core/binary/values.html)

WebAssemblyにおいては整数はLEB128という形式でバイナリにエンコードされています。
[WikipediaのLEB128の記事が難しすぎる](https://github.com/lvs7k/rust-blog/blob/main/posts/wikipedia-leb128-is-too-difficult.md)という記事を書いたので、参考に実装します。
記事を書いたあとに他の人が書いたもっと綺麗なコードを見たので、それを参考にしています。

```rust
    // Value

    // LEB128: https://en.wikipedia.org/wiki/LEB128
    pub fn u32(&mut self) -> Result<u32, ParseError> {
        let bit = std::mem::size_of::<u32>() * 8;
        let mut result = 0;
        let mut shift = 0;
        let mut buf;

        loop {
            buf = self.byte()?;

            let b = (buf & 0b01111111) as u32;

            if (shift >= bit - (bit % 7)) && (b >= (1 << (bit % 7))) {
                return Err(ParseError::Malformed);
            }

            result |= b << shift;
            shift += 7;

            if buf & 0b10000000 == 0 {
                break;
            }
        }

        Ok(result)
    }

    pub fn i32(&mut self) -> Result<i32, ParseError> {
        let bit = std::mem::size_of::<i32>() * 8;
        let mut result = 0;
        let mut shift = 0;
        let mut buf;

        loop {
            buf = self.byte()?;

            let b = (buf & 0b01111111) as i32;

            if shift >= bit - (bit % 7) {
                let is_positive = (b & 0b01000000) == 0;

                if is_positive {
                    if b >= (1 << (bit % 7)) {
                        return Err(ParseError::Malformed);
                    }
                } else {
                    let mask = (!0 << (bit % 7)) & 0b01111111;
                    if b & mask != mask {
                        return Err(ParseError::Malformed);
                    }
                }
            }

            result |= b << shift;
            shift += 7;

            if buf & 0b10000000 == 0 {
                let is_negative = (b & 0b01000000) != 0;

                if is_negative && shift <= bit {
                    result |= !0 << shift;
                }
                break;
            }
        }

        Ok(result)
    }

    pub fn f32(&mut self) -> Result<f32, ParseError> {
        let mut buf = [0; 4];
        self.reader.read_exact(&mut buf)?;
        Ok(f32::from_le_bytes(buf))
    }
```

WebAssemblyの文字列はUTF-8です。
`std::str::from_utf8`でバイナリからRustの文字列へ変換します。

```rust
    pub fn name(&mut self) -> Result<String, ParseError> {
        let utf8_bytes = self.vec(Self::byte)?;
        let name = std::str::from_utf8(&utf8_bytes[..])?.to_string();
        Ok(name)
    }
```

##### [Types](https://webassembly.github.io/spec/core/binary/types.html)

今回必要となる型はこれだけです。

```rust
    // Types

    pub fn valtype(&mut self) -> Result<ValType, ParseError> {
        Ok(match self.byte()? {
            b'\x7f' => ValType::Num(NumType::I32),
            b'\x7e' => ValType::Num(NumType::I64),
            b'\x7d' => ValType::Num(NumType::F32),
            b'\x7c' => ValType::Num(NumType::F64),
            _ => return Err(ParseError::Malformed),
        })
    }

    pub fn resulttype(&mut self) -> Result<ResultType, ParseError> {
        self.vec(Self::valtype)
    }

    pub fn functype(&mut self) -> Result<FuncType, ParseError> {
        self.consume(b'\x60')?;
        Ok(FuncType {
            input: self.vec(Self::valtype)?,
            output: self.vec(Self::valtype)?,
        })
    }
```

##### [Instructions](https://webassembly.github.io/spec/core/binary/instructions.html)

`if`の構文解析が`false`だった場合の部分がある場合とない場合があり、少し難しいです。
`instrs`というメソッドを用意して、それを使って`if`を解析しています。

```rust
    // Instructions

    pub fn blocktype(&mut self) -> Result<BlockType, ParseError> {
        let start_pos = self.reader.stream_position()?;

        if self.byte()? == b'\x40' {
            return Ok(BlockType::Empty);
        }

        self.reader.seek(SeekFrom::Start(start_pos))?;
        if let Ok(valtype) = self.valtype() {
            return Ok(BlockType::Val(valtype));
        }

        self.reader.seek(SeekFrom::Start(start_pos))?;
        if let Ok(x) = self.i32() {
            return Ok(BlockType::Idx(u32::try_from(x).unwrap()));
        }

        Err(ParseError::Malformed)
    }

    pub fn instr(&mut self) -> Result<Instr, ParseError> {
        Ok(match self.byte()? {
            // Control Instructions
            b'\x02' => Instr::Block {
                blocktype: self.blocktype()?,
                expr: self.expr()?,
            },
            b'\x03' => Instr::Loop {
                blocktype: self.blocktype()?,
                expr: self.expr()?,
            },
            b'\x04' => {
                let blocktype = self.blocktype()?;
                let true_branch = self.instrs()?;
                match self.byte()? {
                    b'\x05' => Instr::If {
                        blocktype,
                        true_branch,
                        false_branch: self.expr()?,
                    },
                    b'\x0b' => Instr::If {
                        blocktype,
                        true_branch,
                        false_branch: Vec::new(),
                    },
                    _ => return Err(ParseError::Malformed),
                }
            }
            b'\x0c' => Instr::Br {
                labelidx: self.u32()?,
            },
            b'\x0d' => Instr::BrIf {
                labelidx: self.u32()?,
            },
            b'\x0f' => Instr::Return,

            // Variable Instructions
            b'\x20' => Instr::LocalGet {
                localidx: self.u32()?,
            },
            b'\x21' => Instr::LocalSet {
                localidx: self.u32()?,
            },

            // Numeric Instructions
            b'\x46' => Instr::IRelOp {
                bit: Bit::B32,
                op: IRelOp::Eq,
            },
            b'\x4c' => Instr::IRelOp {
                bit: Bit::B32,
                op: IRelOp::Le { sign: Sign::Signed },
            },
            b'\x41' => Instr::I32Const { val: self.i32()? },
            b'\x6a' => Instr::IBinOp {
                bit: Bit::B32,
                op: IBinOp::Add,
            },

            _ => unimplemented!(),
        })
    }

    fn instrs(&mut self) -> Result<Vec<Instr>, ParseError> {
        let mut res = Vec::new();

        loop {
            res.push(self.instr()?);
            let next_byte = self.peek()?;
            if next_byte == b'\x05' || next_byte == b'\x0b' {
                break;
            }
        }

        Ok(res)
    }

    pub fn expr(&mut self) -> Result<Expr, ParseError> {
        let instrs = self.instrs()?;
        self.consume(b'\x0b')?;
        Ok(instrs)
    }
```

##### [Modules](https://webassembly.github.io/spec/core/binary/modules.html)

Indexは多くの種類があり別々の型に分けても良いですが、今回はそこまでする必要がないと思ったので共通の`Idx`という型にしました。

また、各セクションを解析するのに便利な`section`メソッドを用意します。
仕様に`section_N(B) ::= N:byte size:u32 cont:B`と書かれています。
最初にセクションのIDが1バイト、次にセクションのバイト数、最後にそのバイト数だけセクションのコンテンツがあります。
コンテンツの大きさが`size:u32`と合っていなければエラーです。

```rust
    // Modules

    pub fn idx(&mut self) -> Result<Idx, ParseError> {
        self.u32()
    }

    fn section<T>(
        &mut self,
        section_id: u8,
        parser: impl FnMut(&mut Parser<R>) -> Result<T, ParseError>,
    ) -> Result<T, ParseError> {
        self.consume(section_id)
            .map_err(|_| ParseError::Backtrack)?;
        self.size_prefixed(parser)
    }
```

次に各セクションを仕様に従って解析していきます。
Custom sectionというセクションがありますが、今回のプログラム実行には関係ないので読み飛ばします。

```rust
    pub fn customsec(&mut self) -> Result<(), ParseError> {
        self.consume(b'\x00').map_err(|_| ParseError::Backtrack)?;
        let size = self.u32()?;

        let start_pos = self.reader.stream_position()?;
        let _name = self.name()?;

        // skip bytes
        let remain = start_pos + size as u64 - self.reader.stream_position()?;
        for _ in 0..remain {
            self.byte()?;
        }

        Ok(())
    }

    pub fn typesec(&mut self) -> Result<Vec<FuncType>, ParseError> {
        self.section(b'\x01', |parser| parser.vec(Self::functype))
    }

    pub fn funcsec(&mut self) -> Result<Vec<Idx>, ParseError> {
        self.section(b'\x03', |parser| parser.vec(Self::idx))
    }

    pub fn exportsec(&mut self) -> Result<Vec<Export>, ParseError> {
        self.section(b'\x07', |parser| parser.vec(Self::export))
    }

    fn export(&mut self) -> Result<Export, ParseError> {
        Ok(Export {
            name: self.name()?,
            desc: self.exportdesc()?,
        })
    }

    fn exportdesc(&mut self) -> Result<ExportDesc, ParseError> {
        Ok(match self.byte()? {
            b'\x00' => ExportDesc::Func(self.idx()?),
            _ => unimplemented!(),
        })
    }

    pub fn codesec(&mut self) -> Result<Vec<(Vec<ValType>, Expr)>, ParseError> {
        self.section(b'\x0a', |parser| parser.vec(Self::code))
    }

    fn code(&mut self) -> Result<(Vec<ValType>, Expr), ParseError> {
        self.size_prefixed(Self::func)
    }

    fn func(&mut self) -> Result<(Vec<ValType>, Expr), ParseError> {
        Ok((
            self.vec(Self::locals)?.into_iter().flatten().collect(),
            self.expr()?,
        ))
    }

    fn locals(&mut self) -> Result<Vec<ValType>, ParseError> {
        let n = self.u32()?;
        let t = self.valtype()?;
        Ok(vec![t; n as usize])
    }
```

最後に`Module`という構文木を作り上げて完成です🎉

バイナリ形式の最初にマジックナンバーとバージョンがありますが、その後の各セクションは一つもない可能性があります。
用意した`opt`というメソッドを使って各セクションを解析していきます。

```rust
    pub fn module(&mut self) -> Result<Module, ParseError> {
        self.magic()?;
        self.version()?;

        self.opt(Self::customsec)?;
        let types = self.opt(Self::typesec)?.unwrap_or_default();
        self.opt(Self::customsec)?;
        let typeidxs = self.opt(Self::funcsec)?.unwrap_or_default();
        self.opt(Self::customsec)?;
        let exports = self.opt(Self::exportsec)?.unwrap_or_default();
        self.opt(Self::customsec)?;
        let codes = self.opt(Self::codesec)?.unwrap_or_default();
        self.opt(Self::customsec)?;

        assert_eq!(typeidxs.len(), codes.len());

        let funcs = typeidxs
            .into_iter()
            .zip(codes)
            .map(|(type_, (locals, body))| {
                Rc::new(Func {
                    type_,
                    locals,
                    body,
                })
            })
            .collect();

        Ok(Module {
            types,
            funcs,
            exports,
        })
    }

    fn magic(&mut self) -> Result<(), ParseError> {
        let magic = b"\x00\x61\x73\x6d";
        for b in magic {
            if *b != self.byte()? {
                return Err(ParseError::Malformed);
            }
        }
        Ok(())
    }

    fn version(&mut self) -> Result<(), ParseError> {
        let magic = b"\x01\x00\x00\x00";
        for b in magic {
            if *b != self.byte()? {
                return Err(ParseError::Malformed);
            }
        }
        Ok(())
    }
}
```

実際にサンプルプログラムを構文解析した`Module`をデバッグ表示すると下記のようになります。
長いので折りたたんでいます。

<details>
<summary>Moduleのデバッグ表示</summary>
  
```shell
[src/main.rs:33:5] &module = Module {
    types: [
        FuncType {
            input: [
                Num(
                    I32,
                ),
            ],
            output: [
                Num(
                    I32,
                ),
            ],
        },
    ],
    funcs: [
        Func {
            type_: 0,
            locals: [
                Num(
                    I32,
                ),
                Num(
                    I32,
                ),
                Num(
                    I32,
                ),
            ],
            body: [
                LocalGet {
                    localidx: 0,
                },
                I32Const {
                    val: 0,
                },
                IRelOp {
                    bit: B32,
                    op: Le {
                        sign: Signed,
                    },
                },
                If {
                    blocktype: Empty,
                    true_branch: [
                        I32Const {
                            val: 0,
                        },
                        Return,
                    ],
                    false_branch: [],
                },
                I32Const {
                    val: 0,
                },
                LocalSet {
                    localidx: 1,
                },
                I32Const {
                    val: 1,
                },
                LocalSet {
                    localidx: 2,
                },
                I32Const {
                    val: 1,
                },
                LocalSet {
                    localidx: 3,
                },
                Block {
                    blocktype: Val(
                        Num(
                            I32,
                        ),
                    ),
                    expr: [
                        Block {
                            blocktype: Empty,
                            expr: [
                                Loop {
                                    blocktype: Empty,
                                    expr: [
                                        LocalGet {
                                            localidx: 3,
                                        },
                                        LocalGet {
                                            localidx: 0,
                                        },
                                        IRelOp {
                                            bit: B32,
                                            op: Eq,
                                        },
                                        BrIf {
                                            labelidx: 1,
                                        },
                                        LocalGet {
                                            localidx: 2,
                                        },
                                        LocalGet {
                                            localidx: 1,
                                        },
                                        LocalGet {
                                            localidx: 2,
                                        },
                                        IBinOp {
                                            bit: B32,
                                            op: Add,
                                        },
                                        LocalSet {
                                            localidx: 2,
                                        },
                                        LocalSet {
                                            localidx: 1,
                                        },
                                        LocalGet {
                                            localidx: 3,
                                        },
                                        I32Const {
                                            val: 1,
                                        },
                                        IBinOp {
                                            bit: B32,
                                            op: Add,
                                        },
                                        LocalSet {
                                            localidx: 3,
                                        },
                                        Br {
                                            labelidx: 0,
                                        },
                                    ],
                                },
                            ],
                        },
                        LocalGet {
                            localidx: 2,
                        },
                    ],
                },
            ],
        },
    ],
    exports: [
        Export {
            name: "fibonacci",
            desc: Func(
                0,
            ),
        },
    ],
}
```
  
</details>
