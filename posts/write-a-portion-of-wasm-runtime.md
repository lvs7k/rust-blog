# WebAssemblyのランタイムの一部分を実装してみる

**目次**
- [はじめに](#はじめに)
- ['1. テキスト形式とバイナリ形式](#1-テキスト形式とバイナリ形式)
  - ['1.1. サンプルプログラム](#11-サンプルプログラム)
  - ['1.2. 知っておきたいこと](#12-知っておきたいこと)
- ['2. 構文木の定義と構文解析](#2-構文木の定義と構文解析)
  - ['2.1. モジュール構成](#21-モジュール構成)
  - ['2.2. 構文木の定義](#22-構文木の定義structurers)
  - ['2.3. 構文解析](#23-構文解析binaryrs)
- ['3. 検証](#3-検証)
  - ['3.1. 関数以外の検証](#31-関数以外の検証validationrs)
  - ['3.2. 関数の検証](#32-関数の検証validationfuncrs)
- ['4. 実行](#4-実行)
  - ['4.1. 実行時の型](#41-実行時の型executionrs)
  - ['4.2. インタープリタ](#42-インタープリタinterpreterrs)
- ['5. ホスト環境への埋め込み](#5-ホスト環境への埋め込み)
- ['6. 実際に動かしてみよう](#6-実際に動かしてみようmainrs)
- [おわりに](#おわりに)

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

セクションIDの解析に失敗したからといって、バイナリ形式が不正(`Malformed`)であるとは限りません。
そのため、セクションIDの解析失敗時は`Backtrack`というエラーを返して別のパーサを試します。
しかし、セクションIDの解析に成功した後その中身の解析でエラーになった場合は、そのバイナリ形式は不正です。

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

### 3. 検証

関数の検証が他に比べて複雑なので、別のモジュールに分けます。

```shell
lvs7k@wsl2:~/wasmpl-rs$ tree --gitignore
.
├── Cargo.toml
├── example
│   └── fibonacci.wat
└── src
    ├── binary.rs
    ├── embedding.rs
    ├── execution.rs
    ├── interpreter.rs
    ├── lib.rs
    ├── main.rs
    ├── structure.rs
    ├── validation
    │   └── func.rs    # これと
    └── validation.rs  # これを作ります
```

#### 3.1. 関数以外の検証(`validation.rs`)

今回の検証のプログラムは戻り値を`Option`として検証に成功したら`Some`失敗したら`None`とします。
これだとどこで検証でエラーになったのかわからないので、より実用的なランタイムを作る場合は`Result`を使いましょう。

```rust
pub mod func;

use crate::structure::*;

fn error_if(pred: bool) -> Option<()> {
    if pred { None } else { Some(()) }
}
```

検証には[Contexts](https://webassembly.github.io/spec/core/valid/conventions.html#contexts)という型を使用します。
今回必要なフィールドは`types`と`funcs`のみです。この後すぐに何を設定するか説明します。

前に説明した通り`block`, `loop`, `if`は`BlockType`というインプットとアウトプットの型を持ちます。
`BlockType`には3パターンの形式があり、`get_functype`メソッドは`BlockType`を`FuncType`へ変換しています。
`BlockType`に型のインデックスが指定されていた場合、`Context`にその型が存在するかをチェックします。
存在しなければ`None`で検証エラーです。

```rust
#[derive(Debug, Default)]
struct Context {
    types: Vec<FuncType>,
    funcs: Vec<FuncType>,
}

impl Context {
    fn get_functype(&self, blocktype: &BlockType) -> Option<FuncType> {
        match blocktype {
            BlockType::Empty => Some(FuncType {
                input: Vec::new(),
                output: Vec::new(),
            }),
            BlockType::Idx(idx) => self.types.get(*idx as usize).cloned(),
            BlockType::Val(valtype) => Some(FuncType {
                input: Vec::new(),
                output: vec![*valtype],
            }),
        }
    }
}
```

この`validate_module`関数が検証のメインとなる関数です。
`Context`には`Module`の`types`と、関数の型(`FuncType`)を設定します。

その後、関数とエクスポートした値を検証していきます。
エクスポートの検証（今回は関数のみ）は、その値が`Context`に存在するかどうかチェックしているだけです。

```rust
pub fn validate_module(module: &Module) -> Option<()> {
    let context = Context {
        types: module.types.clone(),
        funcs: module
            .funcs
            .iter()
            .map(|func| module.types[func.type_ as usize].clone())
            .collect(),
    };

    for func in &module.funcs {
        func::validate_func(&context, func)?;
    }

    for export in &module.exports {
        validate_export(&context, export)?;
    }

    Some(())
}

fn validate_export(context: &Context, export: &Export) -> Option<()> {
    match export.desc {
        ExportDesc::Func(idx) => {
            context.funcs.get(idx as usize)?;
        }
    }

    Some(())
}
```

#### 3.2. 関数の検証(`validation/func.rs`)

まず関数の検証とはどのようなことをするのかイメージをつかみましょう。
WebAssemblyの仕様のAppendixに[Validation Algorithm](https://webassembly.github.io/spec/core/appendix/algorithm.html)というページがあり、
親切なことに疑似コードで関数の検証のアルゴリズムについて書いてくれています。

例えば下記のような関数を検証するとします。
型を入れるスタックを用意して、各命令を実行したときのスタックの状態（型のみ）を考えていきます。

```wat
(func $hello (result i32)
  i32.const 1 ;; stack: [i32]
  i32.const 2 ;; stack: [i32][i32]
  i32.add     ;; stack: [i32]
)
```

`i32.const`を実行したらスタックに`i32`を追加します。ここで検証エラーになることはありません。
`i32.add`を実行したら「スタックに`i32`が2つ存在すること」をチェックし、なければ検証エラーとします。
その後、スタックに足し算の結果`i32`を追加します。

スタックに追加する型は下記のように定義します。
WebAssemblyの命令の中には`drop`や`select`のような`value-polymorphic`と呼ばれる型を問わない命令が存在します。
`drop`はスタックから値を1つ取り出す命令ですが、`i32.add`と異なりどんな型の値がスタックに乗っていてもかまいません。

これを表現するために下記のような`Any`という型を用意します。
スタックから値を取り出す際に期待する型かどうかチェックしますが、`Any`の場合はチェックOKとなります。

```rust
use crate::structure::*;

use super::{Context, error_if};

#[derive(Debug, Clone, Copy)]
enum ValTypeAny {
    Exact(ValType),
    Any,
}

impl PartialEq for ValTypeAny {
    fn eq(&self, other: &Self) -> bool {
        use ValTypeAny::*;
        match (self, other) {
            (Any, _) => true,
            (_, Any) => true,
            (Exact(a), Exact(b)) => a == b,
        }
    }
}
```

疑似コードと同様に`CtrlFrame`という型を用意します。
`block`, `loop`, `if`といったStructured Instructionはネストすることができます。
これらの命令を実行するたびに、`CtrlFrame`を次に定義する`Validator`のスタックに追加していきます。

```rust
#[derive(Debug)]
struct CtrlFrame {
    is_loop: bool,
    functype: FuncType,
    height: usize,
    unreachable: bool,
}
```

`is_loop`でループかそれ以外かを判定できるようにしています。
理由は`loop`のみブランチ命令(`br`など)の実行時のチェック内容が異なるからです。

##### [Labels](https://webassembly.github.io/spec/core/exec/runtime.html#labels)について

> [!WARNING]
> 説明が下手くそなので時間があれば書き直す。

ここで私が仕様を読んでいて一番理解に苦しんだ`Label`について説明します。
Executionの[Instructions](https://webassembly.github.io/spec/core/exec/instructions.html)を見ると、`block`と`loop`に次のような式が書かれています。

```
block blocktype instr* end

F; val^m block bt instr* end -> F; label_n {ε} val^m instr* end
```

これは下記のようなことを言っているのだと思います。

- `block`の実行は`label`の実行に置き換えることができる
- `label`のあとの`{ .. }`の部分は、このラベルに対してブランチ命令でジャンプした際に次に実行する処理を表す（以後、「継続」と呼びます）
- `label`は引数(arity)を持っており、継続へのインプットの数を表す

上の例では`block`内でそのブロックに対してブランチ命令でジャンプするとブロックを抜けるので、継続はなし(ε)となっています。
そして、`block`の`blocktype`の戻り値の数は`n`なので、継続（ブロックを抜けた後の処理）へのインプットの数は`n`となります。
要は「ブロックを抜けるときは`blocktype`の戻り値の数だけスタックに値を積まなければならない」ということです。

```
loop blocktype instr* end

F; val^m loop bt instr* end -> F; label_m { loop bt instr* end } val^m instr* end
```

`block`が理解できれば`loop`も理解できるようになります。
`loop`では`label`の引数(arity)が`block`とは異なり`blocktype`の引数の数となっています。
`loop`内でそのループに対してブランチ命令でジャンプするとループの先頭に戻るので、継続は`{ loop bt instr* end }`となります。
`label`の引数(arity)は継続に対するインプットの数なので、`blocktype`の引数の数になります。

`CtrlFrame`に`is_loop`というフィールドを持たせているのは、`loop`だけこの`label`の引数(arity)が異なるからです。
`if`は`block`に置き換えることができるので`block`と同じ扱いです。

##### 関数の検証（続き）

関数の検証には次の型を使います。


```rust
#[derive(Debug, Default)]
struct Validator {
    stack: Vec<ValTypeAny>,
    ctrls: Vec<CtrlFrame>,
    locals: Vec<ValType>,
    return_: Option<ResultType>, // None if no return is allowed, as in free-standing expressions
}
```

疑似コードを見ながら同じようなコードを実装していきます。

```rust
impl Validator {
    fn push_val(&mut self, type_: ValTypeAny) {
        self.stack.push(type_)
    }

    fn push_vals(&mut self, types: &[ValType]) {
        for valtype in types {
            self.push_val(ValTypeAny::Exact(*valtype));
        }
    }

    fn pop_val(&mut self) -> Option<ValTypeAny> {
        if self.stack.len() == self.current_ctrl().height && self.current_ctrl().unreachable {
            return Some(ValTypeAny::Any);
        }
        error_if(self.stack.len() == self.current_ctrl().height)?;
        self.stack.pop()
    }

    fn pop_val_exact(&mut self, expect: &ValType) -> Option<ValTypeAny> {
        let actual = self.pop_val()?;
        let expect = ValTypeAny::Exact(*expect);
        error_if(actual != expect)?;
        Some(actual)
    }

    fn pop_vals_exact(&mut self, expects: &[ValType]) -> Option<Vec<ValTypeAny>> {
        let popped: Option<Vec<_>> = expects
            .iter()
            .rev()
            .map(|valtype| self.pop_val_exact(valtype))
            .collect();

        popped.map(|mut v| {
            v.reverse();
            v
        })
    }

    fn current_ctrl(&mut self) -> &mut CtrlFrame {
        self.ctrls
            .last_mut()
            .expect("instructions must be executed in a frame")
    }

    fn push_ctrl(&mut self, is_loop: bool, functype: FuncType) {
        let mut inputs = functype
            .input
            .iter()
            .map(|valtype| ValTypeAny::Exact(*valtype))
            .collect::<Vec<_>>();

        self.ctrls.push(CtrlFrame {
            is_loop,
            functype,
            height: self.stack.len(),
            unreachable: false,
        });

        self.stack.append(&mut inputs);
    }

    fn pop_ctrl(&mut self) -> Option<CtrlFrame> {
        let expects = self.current_ctrl().functype.output.clone();
        let height = self.current_ctrl().height;
        self.pop_vals_exact(&expects)?;
        error_if(self.stack.len() != height)?;
        self.ctrls.pop()
    }

    fn unreachable(&mut self) {
        let new_len = self.current_ctrl().height;
        assert!(
            new_len <= self.stack.len(),
            "do not resize the stack longer"
        );
        self.stack.resize(new_len, ValTypeAny::Any);
        self.current_ctrl().unreachable = true;
    }

    fn nth_label_type(&self, n: usize) -> Option<ResultType> {
        self.ctrls.get(self.ctrls.len() - 1 - n).map(|ctrl| {
            if ctrl.is_loop {
                ctrl.functype.input.clone()
            } else {
                ctrl.functype.output.clone()
            }
        })
    }
```

重要なのは`pop_val`と`unreachable`です。
前に次のコードがエラーにならないと書きました。

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

仕様のValidationの[return](https://webassembly.github.io/spec/core/valid/instructions.html#xref-syntax-instructions-syntax-instr-control-mathsf-return)を見ると、
`return: [t_1* t*] -> [t_2*]`と書かれています。
この関数の戻り値は`i32`なので、ここでの`return`は`return: [t_1* i32] -> [t_2*]`と解釈されます。
要は「`return`の実行時にスタックのトップの値が`i32`でさえあればよい」ということです。

この検証を行うために、`CtrlFrame`に`unreachable`というフラグを持たせています。
`return`の実行後に`block`を抜けるときに、`block`の戻り値の`i32`がスタックに追加されているかどうかチェックします。
`return`の実行時に`unreachable`フラグが立つため`pop_val`が`Any`を返し、チェックOKとなります。

```rust
    fn pop_val(&mut self) -> Option<ValTypeAny> {
        if self.stack.len() == self.current_ctrl().height && self.current_ctrl().unreachable {
            return Some(ValTypeAny::Any);
        }
        error_if(self.stack.len() == self.current_ctrl().height)?;
        self.stack.pop()
    }
```

疑似コードと異なり`label`を使っていないので、`block`, `loop`, `if`を検証するためのメソッドを定義します。
検証時は`wasm`の実行時とは異なり、ブランチ命令や`return`が実行されても早期リターンしません。
`if`も`true`と`false`の両方の場合を検証していきます。

```rust
    fn ctrl(
        &mut self,
        context: &Context,
        functype: &FuncType,
        expr: &Expr,
        is_loop: bool,
    ) -> Option<()> {
        self.pop_vals_exact(&functype.input)?;
        self.push_ctrl(is_loop, functype.clone());

        for instr in expr {
            self.instr(context, instr)?;
        }

        self.pop_ctrl()?;
        self.push_vals(&functype.output);

        Some(())
    }

    fn block(&mut self, context: &Context, blocktype: &BlockType, expr: &Expr) -> Option<()> {
        let functype = context.get_functype(blocktype)?;
        self.ctrl(context, &functype, expr, false)
    }

    fn loop_(&mut self, context: &Context, blocktype: &BlockType, expr: &Expr) -> Option<()> {
        let functype = context.get_functype(blocktype)?;
        self.ctrl(context, &functype, expr, true)
    }

    fn if_(
        &mut self,
        context: &Context,
        blocktype: &BlockType,
        true_branch: &Expr,
        false_branch: &Expr,
    ) -> Option<()> {
        let functype = context.get_functype(blocktype)?;

        self.pop_val_exact(&ValType::Num(NumType::I32))?;
        self.pop_vals_exact(&functype.input)?;

        // true branch
        self.push_ctrl(false, functype.clone());
        for instr in true_branch {
            self.instr(context, instr)?;
        }
        self.pop_ctrl()?;

        // false branch
        self.push_ctrl(false, functype.clone());
        for instr in false_branch {
            self.instr(context, instr)?;
        }
        self.pop_ctrl()?;

        // end
        self.push_vals(&functype.output);

        Some(())
    }
```

各命令の検証です。

```rust
    fn instr(&mut self, context: &Context, instr: &Instr) -> Option<()> {
        use Instr::*;

        match instr {
            Block { blocktype, expr } => self.block(context, blocktype, expr),
            Loop { blocktype, expr } => self.loop_(context, blocktype, expr),
            If {
                blocktype,
                true_branch,
                false_branch,
            } => self.if_(context, blocktype, true_branch, false_branch),
            Br { labelidx } => {
                error_if(self.ctrls.len() <= *labelidx as usize)?;
                let label_type = self.nth_label_type(*labelidx as usize)?;
                self.pop_vals_exact(&label_type)?;
                self.unreachable();
                Some(())
            }
            BrIf { labelidx } => {
                error_if(self.ctrls.len() <= *labelidx as usize)?;
                self.pop_val_exact(&ValType::Num(NumType::I32))?;
                let label_type = self.nth_label_type(*labelidx as usize)?;
                self.pop_vals_exact(&label_type)?;
                self.push_vals(&label_type);
                Some(())
            }
            Return => {
                let resulttype = self.return_.clone()?;
                self.pop_vals_exact(&resulttype)?;
                self.unreachable();
                Some(())
            }
            I32Const { .. } => {
                self.push_val(ValTypeAny::Exact(ValType::Num(NumType::I32)));
                Some(())
            }
            IBinOp { bit, .. } => {
                match bit {
                    Bit::B32 => {
                        self.pop_val_exact(&ValType::Num(NumType::I32))?;
                        self.pop_val_exact(&ValType::Num(NumType::I32))?;
                        self.push_val(ValTypeAny::Exact(ValType::Num(NumType::I32)));
                    }
                    Bit::B64 => {
                        self.pop_val_exact(&ValType::Num(NumType::I64))?;
                        self.pop_val_exact(&ValType::Num(NumType::I64))?;
                        self.push_val(ValTypeAny::Exact(ValType::Num(NumType::I64)));
                    }
                }
                Some(())
            }
            IRelOp { bit, .. } => {
                match bit {
                    Bit::B32 => {
                        self.pop_val_exact(&ValType::Num(NumType::I32))?;
                        self.pop_val_exact(&ValType::Num(NumType::I32))?;
                        self.push_val(ValTypeAny::Exact(ValType::Num(NumType::I32)));
                    }
                    Bit::B64 => {
                        self.pop_val_exact(&ValType::Num(NumType::I64))?;
                        self.pop_val_exact(&ValType::Num(NumType::I64))?;
                        self.push_val(ValTypeAny::Exact(ValType::Num(NumType::I32)));
                    }
                }
                Some(())
            }
            LocalGet { localidx } => {
                let valtype = self.locals.get(*localidx as usize)?;
                self.push_val(ValTypeAny::Exact(*valtype));
                Some(())
            }
            LocalSet { localidx } => {
                let valtype = *self.locals.get(*localidx as usize)?;
                self.pop_val_exact(&valtype)?;
                Some(())
            }
        }
    }
}
```

最後に親モジュール(`validation.rs`)で使用する`validate_func`関数を定義します。

```rust
pub(super) fn validate_func(context: &Context, func: &Func) -> Option<()> {
    let functype = context.funcs[func.type_ as usize].clone();

    let mut validator = Validator {
        // parameters & locals
        locals: functype
            .input
            .iter()
            .chain(func.locals.iter())
            .copied()
            .collect(),
        return_: Some(functype.output.clone()),
        ..Default::default()
    };

    let functype = FuncType {
        input: Vec::new(),
        output: functype.output,
    };

    validator.ctrl(context, &functype, &func.body, false)
}
```

ポイントは次の2つです。

- `Validator`の`locals`に関数のパラメータ＆ローカル変数の型を設定する
- インプットはなし、アウトプットは関数の戻り値と同じ、の`block`として検証を行う

[Invocation of function address a](https://webassembly.github.io/spec/core/exec/instructions.html#exec-invoke)に
`S; val^n (invoke a) -> S; frame_m {F} label_m {} instr* end end`と書かれています。
`label`の継続が何もないので、これは関数呼び出しは`block`の実行として置き換えられることを意味しています。
`label`の引数(arity)は関数の戻り値の数`m`となっています。
なので関数呼び出しの検証を`block`の検証に置き換えて行っています。

### 4. 実行

関数の検証が他に比べて複雑なので、別のモジュールに分けます。

```shell
lvs7k@wsl2:~/wasmpl-rs$ tree --gitignore
.
├── Cargo.toml
├── example
│   └── fibonacci.wat
└── src
    ├── binary.rs
    ├── embedding.rs
    ├── execution.rs   # これと
    ├── interpreter.rs # これを作ります
    ├── lib.rs
    ├── main.rs
    ├── structure.rs
    ├── validation
    │   └── func.rs
    └── validation.rs
```

#### 4.1. 実行時の型(`execution.rs`)

Executionの[Runtime Structure](https://webassembly.github.io/spec/core/exec/runtime.html)を見ながら必要な型を定義します。

```rust
use std::rc::Rc;

use crate::structure::*;

// Runtime Structure

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Num {
    I32(i32),
    I64(i64),
    F32(f32),
    F64(f64),
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Val {
    Num(Num),
}
```

後で必要になるので実行時の値から型を得るメソッドと、型から実行時のデフォルト値を取得するメソッドを定義します。

```rust
impl Val {
    pub fn valtype(&self) -> ValType {
        match self {
            Val::Num(Num::I32(_)) => ValType::Num(NumType::I32),
            Val::Num(Num::I64(_)) => ValType::Num(NumType::I64),
            Val::Num(Num::F32(_)) => ValType::Num(NumType::F32),
            Val::Num(Num::F64(_)) => ValType::Num(NumType::F64),
        }
    }
}

impl ValType {
    pub fn default_val(&self) -> Val {
        match self {
            ValType::Num(NumType::I32) => Val::Num(Num::I32(0)),
            ValType::Num(NumType::I64) => Val::Num(Num::I64(0)),
            ValType::Num(NumType::F32) => Val::Num(Num::F32(0.0)),
            ValType::Num(NumType::F64) => Val::Num(Num::F64(0.0)),
        }
    }
}
```

```rust
#[derive(Debug)]
pub enum Trap {
    Unreachable,
}

#[derive(Debug, Default)]
pub struct Store {
    pub funcs: Vec<FuncInst>,
}

pub type Addr = usize;

#[derive(Debug)]
pub struct ModuleInst {
    pub types: Vec<FuncType>,
    pub funcaddrs: Vec<Addr>,
    pub exports: Vec<ExportInst>,
}

#[derive(Debug, Clone)]
pub enum FuncInst {
    Wasm {
        type_: FuncType,
        module: Rc<ModuleInst>,
        code: Rc<Func>,
    },
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ExportInst {
    pub name: String,
    pub value: ExternVal,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ExternVal {
    Func(Addr),
    Table(Addr),
    Mem(Addr),
    Global(Addr),
}
```

Executionの[Modules](https://webassembly.github.io/spec/core/exec/modules.html)を見ながらモジュールのインスタンス化を行う関数を定義します。
仕様によると関数のインスタンス(`FuncInst`)はモジュールのインスタンス(`ModuleInst`)への参照を持つので、
先に関数のアドレスだけ割り当ててモジュールのインスタンスを作成し、`Rc<ModuleInst>`を関数のインスタンスに持たせています。

```rust
// Modules

fn allocmodule(store: &mut Store, module: &Module) -> Rc<ModuleInst> {
    let types = module.types.clone();

    // allocate function addresses
    let funcaddrs = (store.funcs.len()..store.funcs.len() + module.funcs.len()).collect::<Vec<_>>();

    let exports = module
        .exports
        .iter()
        .enumerate()
        .map(|(i, export)| ExportInst {
            name: export.name.clone(),
            value: ExternVal::Func(funcaddrs[i]),
        })
        .collect();

    let moduleinst = Rc::new(ModuleInst {
        types,
        funcaddrs,
        exports,
    });

    // allocate functions
    module.funcs.iter().for_each(|func| {
        store.funcs.push(FuncInst::Wasm {
            type_: moduleinst.types[func.type_ as usize].clone(),
            module: moduleinst.clone(),
            code: func.clone(),
        })
    });

    moduleinst
}

pub fn instantiate(store: &mut Store, module: &Module) -> Result<Rc<ModuleInst>, Trap> {
    Ok(allocmodule(store, module))
}
```

#### 4.2. インタープリタ(`interpreter.rs`)

このモジュールでインスタンス化された関数の実行を定義します。

ローカル変数はスタックとは別の場所で管理していると書きましたが、その場所がこの`Frame`という構造です。
後に定義する`Interpreter`という構造の中でこのフレームをスタックとして積み上げていきます。
関数呼び出しを行うごとにスタックにフレームをプッシュし、関数を抜けるときにポップします。

```rust
use std::rc::Rc;

use crate::{
    execution::*,
    structure::{self, *},
};

#[derive(Debug, Default)]
pub struct Frame {
    locals: Vec<Val>,
    #[allow(dead_code)]
    module: Option<Rc<ModuleInst>>,
}
```

前に説明したとおり`block`, `loop`, `if`は内側から番号が振られ、ブランチ命令によって指定した場所へジャンプすることができます。
今回のインタープリタでは`block`, `loop`, `if`のStructured InstructionはRustの関数として呼び出します。
例えば`br 0`なら今呼び出されている関数を早期リターン(`block`の場合)したり、continue(`loop`の場合)したりする必要があります。
`br 1`なら今呼び出されている関数の呼び出し元の関数へすぐに戻らなければなりません。

このような早期リターンを実現する際に便利なのがRustの[std::ops::ControlFlow](https://doc.rust-lang.org/std/ops/enum.ControlFlow.html)です。
今回は`Branch`と`Return`を区別する必要があるので、下記のような型を自作して使用します。

```rust
#[derive(Debug)]
enum Flow {
    Branch { relative_depth: usize },
    Return,
    Continue,
}
```

```rust
#[derive(Debug, Default)]
pub struct Interpreter {
    pub stack: Vec<Val>,
}

impl Interpreter {
    fn pop_i32(&mut self) -> i32 {
        let Some(Val::Num(Num::I32(i))) = self.stack.pop() else {
            unreachable!()
        };
        i
    }
}
```

[Function Calls](https://webassembly.github.io/spec/core/exec/instructions.html#function-calls)に関数呼び出しの方法について書かれています。
新しい`Frame`を作りパラメータとローカル変数を設定し、`block`の実行に置き換えています。


```rust
impl Interpreter {
    // [Function Calls](https://webassembly.github.io/spec/core/exec/instructions.html#function-calls)
    pub fn invoke(
        &mut self,
        store: &mut Store,
        frame: &mut Frame,
        funcaddr: &Addr,
    ) -> Result<(), Trap> {
        match store.funcs[*funcaddr].clone() {
            FuncInst::Wasm {
                type_,
                module,
                code,
            } => self.invoke_wasm(store, frame, type_, module, code),
        }
    }

    fn invoke_wasm(
        &mut self,
        store: &mut Store,
        _frame: &mut Frame,
        type_: FuncType,
        module: Rc<ModuleInst>,
        code: Rc<Func>,
    ) -> Result<(), Trap> {
        // pop parameters from the stack
        let args = (0..type_.input.len())
            .map(|_| self.stack.pop().unwrap())
            .collect::<Vec<_>>();

        // parameters & locals
        let locals = args
            .into_iter()
            .rev()
            .chain(code.locals.iter().map(|valtype| valtype.default_val()))
            .collect();

        let mut new_frame = Frame {
            locals,
            module: Some(module),
        };

        self.block(store, &mut new_frame, &code.body)?;

        Ok(())
    }
```

`block`の実行は`self.instr()`というメソッドで1命令ずつ実行をしていきます。
`self.instr()`が`Flow::Branch`を返したときにどう処理しているかに注目してください。

WebAssemblyの仕様で未だによくわかっていないのが、`block`や`loop`を抜けるときにスタックから余分な値を削除するのか？という点です。
検証のときはスタックに余分な値が残っていた場合エラーとします。

しかし、仕様の[Exiting instr* with label L](https://webassembly.github.io/spec/core/exec/instructions.html#exiting-xref-syntax-instructions-syntax-instr-mathit-instr-ast-with-label-l)を見ると
`label_n {instr*} val* end -> val*`と書かれており、スタックにに余分な値が残っていてもそのままにしているように思えます。

```rust
    fn block(&mut self, store: &mut Store, frame: &mut Frame, expr: &Expr) -> Result<Flow, Trap> {
        for instr in expr {
            match self.instr(store, frame, instr)? {
                Flow::Branch { relative_depth } => {
                    if relative_depth != 0 {
                        return Ok(Flow::Branch {
                            relative_depth: relative_depth - 1,
                        });
                    }
                    break;
                }
                Flow::Return => return Ok(Flow::Return),
                Flow::Continue => (),
            }
        }
        Ok(Flow::Continue)
    }
```

`loop`も`block`とほぼ同じですが、ブランチ命令のジャンプ先がループだった場合にそのループの先頭に戻る(`continue 'outer;`)ようにします。

```rust
    fn loop_(&mut self, store: &mut Store, frame: &mut Frame, expr: &Expr) -> Result<Flow, Trap> {
        'outer: loop {
            for instr in expr {
                match self.instr(store, frame, instr)? {
                    Flow::Branch { relative_depth } => {
                        if relative_depth != 0 {
                            return Ok(Flow::Branch {
                                relative_depth: relative_depth - 1,
                            });
                        }
                        continue 'outer;
                    }
                    Flow::Return => return Ok(Flow::Return),
                    Flow::Continue => (),
                }
            }
            break;
        }
        Ok(Flow::Continue)
    }
```

最後に各命令を処理するメソッドを定義します。

```rust
    fn instr(&mut self, store: &mut Store, frame: &mut Frame, instr: &Instr) -> Result<Flow, Trap> {
        use Instr::*;

        match instr {
            Block { expr, .. } => self.block(store, frame, expr),
            Loop { expr, .. } => self.loop_(store, frame, expr),
            If {
                true_branch,
                false_branch,
                ..
            } => {
                let cond = self.pop_i32();
                if cond != 0 {
                    self.block(store, frame, true_branch)
                } else {
                    self.block(store, frame, false_branch)
                }
            }
            Br { labelidx } => Ok(Flow::Branch {
                relative_depth: *labelidx as usize,
            }),
            BrIf { labelidx } => {
                let cond = self.pop_i32();
                if cond != 0 {
                    Ok(Flow::Branch {
                        relative_depth: *labelidx as usize,
                    })
                } else {
                    Ok(Flow::Continue)
                }
            }
            Return => Ok(Flow::Return),
            I32Const { val } => {
                self.stack.push(Val::Num(Num::I32(*val)));
                Ok(Flow::Continue)
            }
            IBinOp { bit, op } => {
                match bit {
                    Bit::B32 => {
                        let b = self.pop_i32();
                        let a = self.pop_i32();
                        match op {
                            structure::IBinOp::Add => self.stack.push(Val::Num(Num::I32(a + b))),
                        }
                    }
                    _ => unimplemented!(),
                };
                Ok(Flow::Continue)
            }
            IRelOp { bit, op } => {
                match bit {
                    Bit::B32 => {
                        let b = self.pop_i32();
                        let a = self.pop_i32();
                        match op {
                            structure::IRelOp::Eq => {
                                self.stack.push(Val::Num(Num::I32((a == b) as i32)))
                            }
                            structure::IRelOp::Le { sign: _ } => {
                                self.stack.push(Val::Num(Num::I32((a <= b) as i32)))
                            }
                        }
                    }
                    _ => unimplemented!(),
                };
                Ok(Flow::Continue)
            }
            LocalGet { localidx } => {
                self.stack.push(frame.locals[*localidx as usize]);
                Ok(Flow::Continue)
            }
            LocalSet { localidx } => {
                let val = self.stack.pop().unwrap();
                frame.locals[*localidx as usize] = val;
                Ok(Flow::Continue)
            }
        }
    }
}
```

### 5. ホスト環境への埋め込み

```shell
lvs7k@wsl2:~/wasmpl-rs$ tree --gitignore
.
├── Cargo.toml
├── example
│   └── fibonacci.wat
└── src
    ├── binary.rs
    ├── embedding.rs # これを作ります
    ├── execution.rs
    ├── interpreter.rs
    ├── lib.rs
    ├── main.rs
    ├── structure.rs
    ├── validation
    │   └── func.rs
    └── validation.rs
```

WebAssemblyの実装をホスト環境にどう埋め込むかについて、仕様のAppendixの[Embedding](https://webassembly.github.io/spec/core/appendix/embedding.html)に書かれています。
これと[Invocation](https://webassembly.github.io/spec/core/exec/modules.html#invocation)を読みながら、
インスタンス化したモジュールからエクスポートされた関数を取得し呼び出すコードを書いていきます。

```rust
use std::error;

use crate::{
    execution::*,
    interpreter::{Frame, Interpreter},
    structure::FuncType,
};

pub fn get_export_funcaddr(
    moduleinst: &ModuleInst,
    name: &str,
) -> Result<Addr, Box<dyn error::Error>> {
    for exportinst in &moduleinst.exports {
        if exportinst.name == name {
            match exportinst.value {
                ExternVal::Func(addr) => return Ok(addr),
                _ => break,
            };
        }
    }
    Err(format!("export name '{}' not found", name).into())
}

// [Invocation](https://webassembly.github.io/spec/core/exec/modules.html#invocation)
pub fn func_invoke(
    store: &mut Store,
    funcaddr: &Addr,
    vals: &[Val],
) -> Result<Vec<Val>, Box<dyn error::Error>> {
    let funcinst = store.funcs[*funcaddr].clone();
    let functype = match &funcinst {
        FuncInst::Wasm { type_, .. } => type_.clone(),
    };

    if vals.len() != functype.input.len() {
        return Err("provided argument number is different from expected".into());
    }

    for (provided, &expected) in vals.iter().map(Val::valtype).zip(functype.input.iter()) {
        if provided != expected {
            return Err("provided argument type is different from expected".into());
        }
    }

    match &funcinst {
        FuncInst::Wasm { .. } => func_invoke_wasm(store, funcaddr, &functype, vals),
    }
}

fn func_invoke_wasm(
    store: &mut Store,
    funcaddr: &Addr,
    functype: &FuncType,
    vals: &[Val],
) -> Result<Vec<Val>, Box<dyn error::Error>> {
    let mut interpreter = Interpreter::default();
    let mut dummy_frame = Frame::default();

    interpreter.stack.append(&mut vals.to_owned());

    interpreter
        .invoke(store, &mut dummy_frame, funcaddr)
        .map_err(|_| "invoke error")?;

    let mut result = (0..functype.output.len())
        .map(|_| interpreter.stack.pop().unwrap())
        .collect::<Vec<_>>();

    result.reverse();

    Ok(result)
}
```

### 6. 実際に動かしてみよう(`main.rs`)

```shell
lvs7k@wsl2:~/wasmpl-rs$ tree --gitignore
.
├── Cargo.toml
├── example
│   └── fibonacci.wat
└── src
    ├── binary.rs
    ├── embedding.rs
    ├── execution.rs
    ├── interpreter.rs
    ├── lib.rs
    ├── main.rs # これを作ります
    ├── structure.rs
    ├── validation
    │   └── func.rs
    └── validation.rs
```

汚いコードですが、16番目のフィボナッチ数列(987)が計算できるかやってみましょう。

```rust
use std::{
    env,
    fs::File,
    io::{self, BufReader},
};

use wasmpl_rs::{
    binary::Parser,
    embedding,
    execution::{self, Num, Store, Val},
    validation,
};

fn main() -> io::Result<()> {
    let mut args = env::args();
    if args.len() != 2 {
        println!("usage: cargo run <path of wasm>");
        return Ok(());
    }

    let wasm = BufReader::new(File::open(args.nth(1).unwrap())?);
    let mut parser = Parser::new(wasm);

    let module = match parser.module() {
        Ok(module) => module,
        Err(e) => {
            println!("{:?}", e);
            println!("{:?}", parser);
            return Ok(());
        }
    };

    assert!(validation::validate_module(&module).is_some());

    let mut store = Store::default();
    let moduleinst = match execution::instantiate(&mut store, &module) {
        Ok(moduleinst) => moduleinst,
        Err(e) => {
            println!("{:?}", e);
            return Ok(());
        }
    };

    let funcaddr = embedding::get_export_funcaddr(&moduleinst, "fibonacci").unwrap();

    let result = embedding::func_invoke(&mut store, &funcaddr, &[Val::Num(Num::I32(16))]).unwrap();

    dbg!(&result);

    Ok(())
}
```

ちゃんと987が計算できました🎉

```shell
lvs7k@wsl2:~/wasmpl-rs$ cargo run example/fibonacci.wasm
   Compiling wasmpl-rs v0.1.0 (/home/lvs7k/wasmpl-rs)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.63s
     Running `target/debug/wasmpl-rs example/fibonacci.wasm`
[src/main.rs:48:5] &result = [
    Num(
        I32(
            987,
        ),
    ),
]
```

## おわりに

今までこのような長い仕様を読んで実装する経験がなかったので、たったこれだけの内容ですがものすごく大変でした。
でも`wasm`の検証と実行についてなんとなくイメージをつかむことができました。

今わかっていなくて知りたいのは「どうやって標準出力に値を出力するのか？（hello, world!がしたい）」ということです。
機会があれば次はWebAssembly System Interface(WASI)について学びたいです。
