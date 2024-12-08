---
title: "RustでBrainfuckのJITコンパイルをする"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Brainfuck", "言語処理系", "JIT"]
published: true
---

## これは何？[^1]

[^1]: 理情のコンパイラ実験で用いられている[mincamlのサイト](https://esumii.github.io/min-caml/)のトップページを参照

これは[理情 Advent Calendar 2024](https://adventar.org/calendars/10445)の８日目の記事です。他の人の記事も読んでいって下さい！

https://adventar.org/calendars/10445

## はじめに
この記事では、RustでBrainfuckのインタプリタとJITコンパイラを作り、実行時間を比較します。その後、簡単な最適化を行います。Rustで簡単なJITコンパイラを作ってみたい人の参考になれば幸いです。

## JITコンパイラとは？
JITコンパイラとは、実行時に機械語へのコンパイルを行うコンパイラのことです。実行時にコンパイルをするため、実行するまで分からない動的な要素(動的型や動的ディスパッチ等)がある言語の高速化に用いられます。JavaのHotSpot VMや、JavaScriptのV8が有名です。一般的には、実行時のコンパイルにはコストがかかるため、バイトコードのインタプリタとJITコンパイラの両方が実装されており、頻繁に呼び出されるメソッドやループをJITコンパイルすることで、処理速度を向上させます。

今回は言語仕様が非常にシンプルなBrainfuckのJITコンパイラを作ります。

:::message
本記事のプログラムの実行環境は、x86-64およびSystem V ABIを前提とします。M1 Mac等を使用している方はご注意下さい。
:::

## Brainfuckの言語仕様
Wikipediaの[Brainfuckの記事](https://ja.wikipedia.org/wiki/Brainfuck)を参照して下さい。

https://ja.wikipedia.org/wiki/Brainfuck

## インタプリタの実装
まず、簡潔なインタプリタの実装を行います。

メモリはu8の配列で表されます。
```rust:interpreter.rs
#[derive(Debug)]
pub struct Runtime {
    instructions: Vec<char>,
    pc: usize,
    jump_stack: Vec<usize>,
    memory: Vec<u8>,
    // Data pointer
    dp: usize,
}
```

```rust:interpreter.rs
pub const MEMORY_SIZE: usize = 30000;

impl Runtime {
    pub fn new(program: &str) -> Self {
        Self {
            instructions: program
                .chars()
                .filter(|c| ['>', '<', '+', '-', '.', ',', '[', ']'].contains(c))
                .collect(),
            pc: 0,
            jump_stack: Vec::new(),
            memory: vec![0; MEMORY_SIZE],
            dp: MEMORY_SIZE / 2,
        }
    }

    pub fn run(&mut self) {
        while self.pc < self.instructions.len() {
            match self.instructions[self.pc] {
                '>' => self.dp += 1,
                '<' => self.dp -= 1,
                '+' => self.memory[self.dp] = self.memory[self.dp].wrapping_add(1),
                '-' => self.memory[self.dp] = self.memory[self.dp].wrapping_sub(1),
                '.' => putchar(self.memory[self.dp]),
                ',' => self.memory[self.dp] = readchar(),
                '[' => {
                    if self.memory[self.dp] == 0 {
                        let mut bracket_count = 1;
                        while bracket_count != 0 {
                            self.pc += 1;
                            match self.instructions[self.pc] {
                                '[' => bracket_count += 1,
                                ']' => bracket_count -= 1,
                                _ => (),
                            }
                        }
                    } else {
                        self.jump_stack.push(self.pc);
                    }
                }
                ']' => {
                    if self.memory[self.dp] != 0 {
                        self.pc = *self.jump_stack.last().unwrap();
                    } else {
                        self.jump_stack.pop();
                    }
                }
                c => eprintln!("Invalid instruction: {c}"),
            }
            self.pc += 1;
        }
    }
}
```

RustではDebugビルド時はオーバーフローするとpanicしてしまうので、それを回避するために`wrapping_*`系メソッドを使っています。
`.`と`,`で`putchar`と`readchar`を呼び出していますが、これらは以下のように定義されます。

```Rust:interpreter.rs
pub fn putchar(c: u8) {
    print!("{}", c as char);
}

pub fn readchar() -> u8 {
    let mut buffer = [0];
    match std::io::stdin().read_exact(&mut buffer) {
        Ok(_) => buffer[0],
        Err(e) => panic!("Error reading from stdin: {}", e),
    }
}
```
これらを関数にしているのは、後にJITコンパイラで用いるためです。

さて、これでインタプリタが実装されましたが、まだ何も最適化がされていないので非常に遅いです。測定してみましょう。

こちらの[素因数分解を行うプログラム](https://github.com/eliben/code-for-blog/blob/main/2017/bfjit/tests/testcases/factor.bf)[^2]で、素数179424691の素因数分解を実行します。

[^2]: [こちらの記事](https://eli.thegreenplace.net/2017/adventures-in-jit-compilation-part-1-an-interpreter.html)より。

```rust:main.rs
use brainfuck_jit::interpreter::Runtime;

fn main() {
    let start = std::time::Instant::now();
    let mut runtime = Runtime::new(include_str!("../tests/factor.bf"));
    runtime.run();
    let end = start.elapsed();
    println!("time: {}ms", end.as_millis());
}
```

```bash
$ echo 179424691 | cargo run --release
..
179424691: 179424691
time: 17519ms
```

## JITコンパイラの実装
次に、JITコンパイラの実装を行います。実用的なJITコンパイラの実装には、[xbyak](https://github.com/herumi/xbyak)のようなJITアセンブラを用いるようなのですが、今回はメモリに直接x86-64の機械語を埋め込んでいく方針で実装します。

```rust: compiler.rs
#[derive(Debug)]
pub struct Compiler<'a> {
    instructions: &'a str,
    jump_stack: Vec<*mut u8>,
    code_current: *mut u8,
    code_start: *mut u8,
}

const CODE_AREA_SIZE: usize = 4096 * 16;
const PAGE_SIZE: usize = 4096;

extern "C" {
    fn mprotect(addr: *const libc::c_void, len: libc::size_t, prot: libc::c_int) -> libc::c_int;
}

impl<'a> Compiler<'a> {
    pub unsafe fn new(program: &'a str) -> Self {
        let layout = alloc::Layout::from_size_align(CODE_AREA_SIZE, PAGE_SIZE).unwrap();
        let code_start = alloc::alloc(layout);
        let r = mprotect(
            code_start as *const libc::c_void,
            CODE_AREA_SIZE,
            libc::PROT_READ | libc::PROT_WRITE | libc::PROT_EXEC,
        );
        assert!(r == 0, "mprotect failed");

        Self {
            instructions: program,
            jump_stack: Vec::new(),
            code_current: code_start,
            code_start,
        }
    }
```

`*mut u8`はRustの可変な生ポインタであり、Cのポインタと同じように使うことができます。`std::alloc::alloc`を用いて、`Layout`で指定した形式のメモリ領域を確保することができます。

確保したメモリ領域はそのままでは実行出来ないため、`mprotect`システムコールを用いて読み込み・書き込み・実行が可能になるように変更します。

ここでは`libc`crateを使っています。

https://github.com/rust-lang/libc

### 関数のプロローグ
それでは、確保したメモリ領域に機械語を書き込みます。
```rust:compiler.rs
    pub unsafe fn compile(&mut self) {
        // prologue
        // push rbp
        self.emit_code(&[0x50 + 5]);
        // mov rbp, rsp
        self.emit_code(&[0x48, 0x89, 0b11_100_101]);
        // push rbx
        self.emit_code(&[0x50 + 3]);
        // mov rbx, rdi
        self.emit_code(&[0x48, 0x89, 0b11_111_011]);
        // add rsp, -8
        self.emit_code(&[0x48, 0x83, 0b11_000_100, 0xf8]);
```

関数のプロローグです。x86-64では、call命令でリターンアドレスがスタックに積まれます。その上にベースポインタを積み、callee-savedレジスタをスタックに積み、スタックポインタを16byte alignされた状態にします。

`emit_code`は、バイト列を書き込むための命令です。
```rust:compiler.rs
    unsafe fn emit_code(&mut self, code: &[u8]) {
        for byte in code {
            *self.code_current = *byte;
            self.code_current = self.code_current.add(1);
        }
        if self.code_current as usize - self.code_start as usize >= CODE_AREA_SIZE {
            panic!("Code area overflow");
        }
    }
```

一つずつ命令を見ていきましょう。
```rust
// push rbp
self.emit_code(&[0x50 + 5]);
```

x86-64の機械語は、可変長命令でアドレッシングモードが多彩なので、非常にわかりにくいです。以下の記事が参考になりました。

https://zenn.dev/mod_poppo/articles/x86-64-machine-code

具体的に命令がどのようなバイト列になるかは、[Intelのマニュアル](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)を見ると分かります。

今回はpush命令なので、`Intel® 64 and IA-32 Architectures Software Developer’s Manual Combined Volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4`のVolume 2, Chapter 4.3を参照します。

すると、64bitレジスタをpushするときの機械語は次のような形式であると分かります。
| Opcode | Instruction |
| - | - |
| 50+rd | PUSH r64 |

これは、`0x50`にレジスタ番号を足したものが実際の命令になるという意味です。(ただし、レジスタ番号8番以降は、REXプリフィックスを用いる必要があります。)

rbpレジスタは番号が5番のレジスタであるため、命令は`0x55`となります。

```rust
// mov rbp, rsp
self.emit_code(&[0x48, 0x89, 0b11_100_101]);
```

次はmov命令です。

| Opcode | Instruction |
| - | - |
| REX.W + 89/r | MOV r/m64, r64 |
| REX.W + 8B/r | MOV r64, r/m64 |

早速複雑な命令が出てきました。`REX.W`とは、REXプリフィックスのWビットを1にするという意味です。REXプリフィックスについては、詳しくは[こちら](https://zenn.dev/mod_poppo/articles/x86-64-machine-code#rex%E3%83%97%E3%83%AA%E3%83%95%E3%82%A3%E3%83%83%E3%82%AF%E3%82%B9)を参照ください。

オペコードの後ろにある`/r`は、ModR/Mバイトのreg/opcodeに`r64`のレジスタ番号を格納するという意味です。ModR/Mバイトについても、[こちら](https://zenn.dev/mod_poppo/articles/x86-64-machine-code#modr%2Fm%E3%83%90%E3%82%A4%E3%83%88%E3%81%A8sib%E3%83%90%E3%82%A4%E3%83%88)を参照ください。

今回は両方ともレジスタを指定するので、命令は2通りあるのですが、`REX.W + 89/r`の方を用いています。

`mov rbp, rsp`の場合、ModR/Mバイトは
- `r/m64`にレジスタを指定するのでmodは`11`
- `reg/opcode`にrspの番号である4(=`100`)
- `r/m`にrbpのレジスタ番号である5(=`101`)

より、`0b11_100_101`となります。

```rust
// push rbx
self.emit_code(&[0x50 + 3]);
// mov rbx, rdi
self.emit_code(&[0x48, 0x89, 0b11_111_011]);
```
`push rbx`で、callee-savedレジスタであるrbxの中身を保存し、`mov rbx, rdi`で関数の第一引数であるrdiの中身をrbxに移動しています。

関数の第一引数にはBrainfuckのメモリのポインタの初期値を渡されることにしています。プログラムの実行をする関数内では、rbxをメモリのポインタとして用います。

```rust
// add rsp, -8
self.emit_code(&[0x48, 0x83, 0b11_000_100, 0xf8]);
```
ABIで、関数呼び出しを行う時はrspを16byte alignさせると決められているので、ここで予め揃えておきます。

| Opcode | Instruction |
| - | - |
| REX.W + 83 /0 ib | ADD r/m64, imm8 |

`/0`より、ModR/Mバイトの`reg/opcode`に`000`が指定されます。ibは1byteの即値を表しており、-8は`0xf8`となります。(即値は符号拡張される。)

### プログラム本体のコンパイル
次に、Brainfuckのプログラムをコンパイルします。
```rust: compiler.rs
for instr in self.instructions.chars() {
    match instr {
        // add rbx, 1
        '>' => self.emit_code(&[0x48, 0x83, 0b11_000_011, 1]),
        // add rbx, -1
        '<' => self.emit_code(&[0x48, 0x83, 0b11_000_011, 0xff]),
        // add [rbx], 1
        '+' => self.emit_code(&[0x80, 0b00_000_011, 1]),
        // add [rbx], -1
        '-' => self.emit_code(&[0x80, 0b00_000_011, 0xff]),
        '.' => {
            // mov dil, [rbx]
            self.emit_code(&[0x40, 0x8a, 0b00_111_011]);
            // mov r10, imm (address of putchar)
            self.emit_code(&[0b0100_1001, 0xb8 + 2]);
            self.emit_code(&(interpreter::putchar as *const () as u64).to_le_bytes());
            // call r10
            self.emit_code(&[0x41, 0xff, 0b11_010_010])
        }
        ',' => {
            // mov r10, imm (address of readchar)
            self.emit_code(&[0b0100_1001, 0xb8 + 2]);
            self.emit_code(&(interpreter::readchar as *const () as u64).to_le_bytes());
            // call r10
            self.emit_code(&[0x41, 0xff, 0b11_010_010]);
            // mov [rbx], al
            self.emit_code(&[0x88, 0b00_000_011]);
        }
        '[' => {
            // cmp [rbx], 0
            self.emit_code(&[0x80, 0b00_111_011, 0]);
            // je 0 (dummy)
            self.emit_code(&[0x0f, 0x84, 0, 0, 0, 0]);
            self.jump_stack.push(self.code_current);
        }
        ']' => {
            // cmp [rbx], 0
            self.emit_code(&[0x80, 0b00_111_011, 0]);

            let loop_start = self.jump_stack.pop().unwrap();
            let offset = loop_start as i32 - (self.code_current as i32 + 6);
            // jne imm (offset)
            self.emit_code(&[0x0f, 0x85]);
            self.emit_code(&offset.to_ne_bytes());

            let offset = self.code_current as i32 - loop_start as i32;
            for (i, byte) in offset.to_le_bytes().iter().enumerate() {
                *loop_start.sub(4).add(i) = *byte;
            }
        }
        _ => {}
    }
}
```

#### `>`と`<`のコンパイル
`>`と`<`は、メモリを指すポインタであるrbxを増減させます。
```rust
// add rbx, 1
'>' => self.emit_code(&[0x48, 0x83, 0b11_000_011, 1]),
// add rbx, -1
'<' => self.emit_code(&[0x48, 0x83, 0b11_000_011, 0xff]),
```

#### `+`と`-`のコンパイル
`+`と`-`は、rbxが指すメモリ1byteに即値を足します。
```rust
// addb [rbx], 1
'+' => self.emit_code(&[0x80, 0b00_000_011, 1]),
// addb [rbx], -1
'-' => self.emit_code(&[0x80, 0b00_000_011, 0xff]),
```

| Opcode | Instruction |
| - | - |
| 80 /0 ib | ADD r/m8 imm8 |

レジスタでアドレスを指定し、displacementが0の時は、ModR/Mバイトのmodに`00`を指定します。

#### `.`のコンパイル
```rust
'.' => {
    // mov dil, [rbx]
    self.emit_code(&[0x40, 0x8a, 0b00_111_011]);
    // mov r10, imm (address of putchar)
    self.emit_code(&[0b0100_1001, 0xb8 + 2]);
    self.emit_code(&(interpreter::putchar as *const () as u64).to_le_bytes());
    // call r10
    self.emit_code(&[0x41, 0xff, 0b11_010_010])
}
```

`.`は、rbxが指すメモリ1byteを読み込み、`putchar`関数を呼び出します。

- `mov r8, r/m8`

| Opcode | Instruction |
| - | - |
| 8A /r | MOV r8, r/m8 |
| REX + 8A /r | MOV r8, r/m8 |

関数の第一引数用レジスタrdiの下位8bitレジスタdilを指定するには、REXプリフィックスを付ける必要があります。(付けないと、4\~7番のレジスタを指定した時に、0~3番のレジスタの8\~15bitを表すレジスタが指定される。)

- `mov r10, imm64`

| Opcode | Instruction |
| - | - |
| REX.W + B8 + rd io | MOV r64, imm64 |

r10はレジスタ番号が10番と8以上であるため、REXプリフィックスのBビットに1を指定する必要があります。ioは8byteの即値を表します。

- `call r10`

| Opcode | Instruction |
| - | - |
| FF /2 | CALL r/m64 |

絶対アドレスで関数を呼び出します。

#### `,`のコンパイル
```rust
',' => {
    // mov r10, imm (address of readchar)
    self.emit_code(&[0b0100_1001, 0xb8 + 2]);
    self.emit_code(&(interpreter::readchar as *const () as u64).to_le_bytes());
    // call r10
    self.emit_code(&[0x41, 0xff, 0b11_010_010]);
    // mov [rbx], al
    self.emit_code(&[0x88, 0b00_000_011]);
}
```

`,`は、`readchar`を呼び出し、結果をrbxが指すメモリに格納します。

`mov [rbx], al`は、関数の戻り値レジスタであるraxの下位8bitを表すalの値を、rbxの指すメモリ上に格納します。

| Opcode | Instruction |
| - | - |
| 88 /r | MOV r/m8, r8 |

#### '['のコンパイル
```rust
'[' => {
    // cmpb [rbx], 0
    self.emit_code(&[0x80, 0b00_111_011, 0]);
    // je 0 (dummy)
    self.emit_code(&[0x0f, 0x84, 0, 0, 0, 0]);
    self.jump_stack.push(self.code_current);
}
```
x86-64では、cmp命令を用いて比較をした後、対応するjump命令を用いて条件付きジャンプを行います。

- `cmpb [rbx], 0`

| Opcode | Instruction |
| - | - |
| 80 /7 ib | CMP r/m8, imm8 |

- `je 0`

`[`の時点ではまだ対応する`]`の命令の位置が確定していないため、ここではダミーの相対アドレスとして0を指定しています。

Jump系の命令はJccの項目にまとめられています。

| Opcode | Instruction |
| - | - |
| 0F 84 cd | JE rel32 |
| 0F 85 cd | JNE rel32 |

後で確定した相対アドレスを書き込むため、self.jump_stackに`je 0`の直後のアドレスを記録しておきます。

#### `]`のコンパイル
```rust
']' => {
    // cmp [rbx], 0
    self.emit_code(&[0x80, 0b00_111_011, 0]);

    let loop_start = self.jump_stack.pop().unwrap();
    let offset = loop_start as i32 - (self.code_current as i32 + 6);
    // jne imm (offset)
    self.emit_code(&[0x0f, 0x85]);
    self.emit_code(&offset.to_ne_bytes());

    let offset = self.code_current as i32 - loop_start as i32;
    for (i, byte) in offset.to_le_bytes().iter().enumerate() {
        *loop_start.sub(4).add(i) = *byte;
    }
}
```

確定した相対アドレスを書き込みます。
`[`では、ポインタの指す値が0ならば、対応する`]`の直後にジャンプするため、`[`の`je`命令の直後から`]`の`jne`命令の直後までジャンプします。(jumpする位置は、jump命令の直後の命令のアドレスからの相対アドレスで指定します。)
`]`では、ポインタの指す値が0でないならば、対応する`[`の直後にジャンプするため、`]`の`jne`命令の直後から`[`の`je`命令の直後にジャンプします。`jne rel32`の長さは6byteであるため、`jne`命令のアドレスに6byte足したアドレスからの相対となっています。

### 関数のエピローグ
```rust:compiler.rs
        // epilogue
        // add rsp, 8
        self.emit_code(&[0x48, 0x83, 0b11_000_100, 8]);
        // pop rbx
        self.emit_code(&[0x58 + 3]);
        // mov rsp, rbp
        self.emit_code(&[0x48, 0x89, 0b11_101_100]);
        // pop rbp
        self.emit_code(&[0x58 + 5]);
        // ret
        self.emit_code(&[0xc3]);
```

callee-savedレジスタであるrbxの値を復帰します。

### 関数の実行
```rust:compiler.rs
    pub unsafe fn run(&self) {
        let f: fn(u64) = std::mem::transmute(self.code_start);
        let memory = vec![0; interpreter::MEMORY_SIZE];
        let dp = memory.as_ptr().add(interpreter::MEMORY_SIZE / 2) as u64;
        f(dp);
    }
```

メモリ領域に書き込んだ命令を実行するには、`std::mem::transmute`関数を用いてバイト列の解釈を`*mut u8`から`fn(u64)`に変えます。

それでは、JITコンパイラの性能を測定してみましょう。インタプリタの時と同様に、素因数分解を行うプログラムを実行します。

```rust:main.rs
use brainfuck_jit::compiler::Compiler;

fn main() {
    let start = std::time::Instant::now();
    unsafe {
        let mut compiler = Compiler::new(include_str!("../tests/factor.bf"));
        compiler.compile();
        compiler.run();
    }
    let end = start.elapsed();
    println!("time: {}ms", end.as_millis());
}
```

```bash
$ echo 179424691 | cargo run --release
..
179424691: 179424691
time: 2210ms
```

| 実装 | 実行時間(ms) |
| - | - |
| インタプリタ | 17519 |
| JITコンパイラ | 2210 |

このように、JITコンパイラの方がインタプリタよりも早くなっていることが分かります。

## 簡単な最適化
今までのインタプリタ・JITコンパイラの実装には明らかに無駄があります。
例えば、インタプリタでは`[`に対応する`]`を毎回探してしまっています。また、インタプリタ・JITコンパイラの双方で、連続する`>`, `<`, `+`, `-`をまとめられていません。

これらをまとめるパーサーを書き、インタプリタ・JITコンパイラから利用するように変更しました。
```rust:parser.rs
#[derive(Debug, Clone, Copy)]
pub enum Instruction {
    PointerIncrement(u32),
    PointerDecrement(u32),
    ValueIncrement(u8),
    ValueDecrement(u8),
    PutChar,
    ReadChar,
    LoopStart { end: usize },
    LoopEnd { start: usize },
    End,
}

pub fn parser(program: &str) -> Vec<Instruction> {
    use Instruction::*;
    let mut instrs = Vec::new();
    let mut jump_stack = Vec::new();
    for instr in program.chars() {
        match instr {
            '>' => {
                if let Some(PointerIncrement(n)) = instrs.last_mut() {
                    *n = n.wrapping_add(1);
                } else {
                    instrs.push(PointerIncrement(1));
                }
            }
            '<' => {
                if let Some(PointerDecrement(n)) = instrs.last_mut() {
                    *n = n.wrapping_add(1);
                } else {
                    instrs.push(PointerDecrement(1));
                }
            }
            '+' => {
                if let Some(ValueIncrement(n)) = instrs.last_mut() {
                    *n = n.wrapping_add(1);
                } else {
                    instrs.push(ValueIncrement(1));
                }
            }
            '-' => {
                if let Some(ValueDecrement(n)) = instrs.last_mut() {
                    *n = n.wrapping_add(1);
                } else {
                    instrs.push(ValueDecrement(1));
                }
            }
            '.' => instrs.push(PutChar),
            ',' => instrs.push(ReadChar),
            '[' => {
                instrs.push(LoopStart { end: 0 });
                jump_stack.push(instrs.len() - 1);
            }
            ']' => {
                let start = jump_stack.pop().unwrap();
                instrs[start] = LoopStart { end: instrs.len() };
                instrs.push(LoopEnd { start });
            }
            _ => (),
        }
    }
    instrs.push(End);
    instrs
}
```

インタプリタで、プログラムの終了の判定を毎回しなくても良いように、命令列の最後にEndを付け加えるようにしました。

すると、実行結果は以下のようになりました。

| 実装 | 実行時間(ms) |
| - | - |
| インタプリタ(最適化なし) | 17519 |
| JITコンパイラ(最適化なし) | 2210 |
| インタプリタ(最適化あり) | 4257 |
| JITコンパイラ(最適化あり) | 1150 |

インタプリタは(もとが相当遅いのもあって)かなり速くなりましたが、JITコンパイラは2倍程度の高速化で収まっています。

## 余談
本当は以前作ったWebAssemblyの(fibが実行出来る程度の)JITコンパイラ[^3]の記事にしようと思ったのですが、WebAssemblyの意味論の話からする気力が無かったため、よりシンプルなBrainfuckのJITコンパイラを作りました。いつか何らかのwasmの記事も書こうと思います。
この記事が、JITコンパイラを作ろうと思っている人の参考になれば幸いです。

[^3]: https://github.com/taka231/wasm_jit
