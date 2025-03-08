# WebAssemblyのランタイムの一部分を実装してみる

**目次**
- はじめに
  - 目標
- '1. テキスト形式とバイナリ形式
  - '1.1. サンプルプログラム
  - '1.2. 知っておきたいこと
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
これと同じように例えば`block`が`[i32 i32] -> [i32]`という型を持つならば、次の2つの条件を満たさなければなりません。

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
