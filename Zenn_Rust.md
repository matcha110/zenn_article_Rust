#
##
###
```Rust

```

# 「The Rust Programming Language 日本語版」を読んだ備忘録
Rustの勉強をするため、「The Rust Programming Language 日本語版」を読んで写経したり、自分用にまとめなおしてみた備忘録
https://doc.rust-jp.rs/book-ja/

# 1. 事始め
cargoの使い方
```Rust
// プロジェクトを作成
cargo new hello_cargo
// プロジェクトをビルド
cargo build
// プロジェクトのビルドと実行
cargo run
// バイナリを生成せずにプロジェクトをビルド
// (エラーがないか確認可能) 
cargo check
```
cargoの長所はOSに関わらず同じコマンドであること

# 2. 数当てゲームのプログラミング
## 変数の設定
```Rust
let a = 5; // 不変
let mut b = 5; // 可変

// a = 10
// a *= 2
// 不変変数の再代入は不可

// b = 10
// b *= 2
// 可変変数の再代入は可能
```
普段はPythonを使うことが多いため、可変・不変を各変数に定義するのは煩雑に思えてしまう

mutableに再代入しない場合はwarningメッセージが出る
Rustはメッセージに従うだけで一定の質が担保されている感じは初心者の自分にとって良い気がする
難しい言語であるのは間違いないが、案外初心者が最初に覚えるべき言語なのかもしれない

## 簡単なプログラム(数当てゲーム)
```Rust
use rand::Rng;
use std::cmp::Ordering;
use std::io; // ライブラリの呼び出し
// https://doc.rust-lang.org/std/prelude/index.html

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    loop { //無限ループ
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue, //不正な入力の処理
        };

        println!("You guessed: {}", guess);
        
        match guess.cmp(&secret_number) {
                Ordering::Less => println!("Too small!"),
                Ordering::Greater => println!("Too big!"),
                Ordering::Equal => {
                    println!("You win!");
                    break; //ループ終了
                }
            }
    }
}
```
# 3 一般的なプログラミングの概念
## 3.1 変数と可変性
```Rust
fn main() {
    let a: i64 = 10;
    // シャドーイング
    {
        // letを使わないとエラーが出る
        // ただし、mutは不要！
        let a = a * 2;
        println!("The value of a in the inner scope is: {}", a);
    }
println!("The value of a in the outer scope is: {}", a);
}

// output
// The value of a in the inner scope is: 20
// The value of a in the outer scope is: 10
```
シャドーイングは{}内だけ値を持つため、{}から出ると値は更新されない

## 3.2 データ型
### 整数型
|大きさ|符号付き|符号なし|
|----|----|----|
8-bit|i8|u8
16-bit|i16|u16
32-bit|i32|u32
64-bit|i64|u64
arch|isize|usize

|数値リテラル|例|
|----|----|
|10進数|98_222|
|16進数|0xff|
|8進数|0o77|
|2進数|0b1111_0000|
|バイト(u8だけ)|b'A'|

タプル型
```Rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let x0 = x.0; //タプルの0番目にアクセス 500
    let x1 = x.1; //6.4
    let x2 = x.2; //1
}
```
pythonはタプルも配列ともにx[0]でアクセスするので異なる！

配列型
```Rust
fn main() {
    let a = [1, 2, 3, 4, 5];

    let first = a[0]; //1
    let second = a[1];　//2
}
```

## 3.3 関数
```Rust
fn five() -> i32 { //返り値の型指定
    5 //最後の" ; "は不要！もしつけるとエラーになる！
}

fn main() {
    let x = five();
    println!("The value of x is: {}", x);
}
```

## 3.5 制御フロー
loop, while, forの使い分けのためにとりあえずFizzbuzzを書いてみる
### loopを使用したFizzbuzz
```Rust
fn main() {
    let mut number = 1;

    loop {
        if number % 15 == 0 {
            println!("Fizzbuzz");
        } else if number % 3 == 0 {
            println!("Fizz");
        } else if number % 5 == 0 {
            println!("Buzz");
        } else if number == 101 {
            break;
        } else {
            println!("{}", number);
        }
        number += 1;
    }
}
```
### whileを使用したFizzbuzz
```Rust
fn main() {
    let mut number = 1;

    while number != 100 {
        if number % 15 == 0 {
            println!("Fizzbuzz");
        } else if number % 3 == 0 {
            println!("Fizz");
        } else if number % 5 == 0 {
            println!("Buzz");
        } else {
            println!("{}", number);
        }
        number += 1;
    }
}
```
### forを使用したFizzbuzz
```Rust
fn main() {
    for number in 1..101 {
        if number % 15 == 0 {
            println!("Fizzbuzz");
        } else if number % 3 == 0 {
            println!("Fizz");
        } else if number % 5 == 0 {
            println!("Buzz");
        } else {
            println!("{}", number);
        }
    }
}
```

# 4.所有権
所有権があることでガベージコレクタが不要となっている！

### スタックとヒープ
### スタック
高速。新しいデータを置いたり、 データを取得する場所を探す必要が絶対にない
スタック上のデータは全て既知の固定サイズでなければならない。

### ヒープ
コンパイル時にサイズがわからなかったり、サイズが可変のデータの置き場
ポインタを返す

```Rust
fn main() {
let s1 = String::from("hello");
let s2 = s1;

// println!("s1 = {}, s2 = {}", s1, s2); //エラーとなる
println!("s2 = {}", s2);
}
```
```Rust
fn main() {
let s1 = String::from("hello");
let s2 = s1.clone(); //deepcopy()と同じ

println!("s1 = {}, s2 = {}", s1, s2);
}
```
```Rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    //'{}'の長さは、{}です
    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len()メソッドは、Stringの長さを返します

    (s, length)
}
```

## 4.2 参照と借用
```Rust
fn main() {
    let s1 = String::from("hello");
    // 可変な参照は常に1つまで
    // 不変な参照は個数制限なし

    let len = calculate_length(&s1);

    // '{}'の長さは、{}です
    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

```Rust
// 不変な参照の後に可変な参照は不可
let mut s = String::from("hello");

let r1 = &s; // 問題なし
let r2 = &s; // 問題なし
let r3 = &mut s; // 大問題！
```
## 4.3 スライス型


```Rust
let s = String::from("hello");
let len = s.len();
// 半開区間
let slice_head = &s[0..2];
let slice_head = &s[..2]; //上記と同じ

let slice_tail = &s[3..len];
let slice_tail = &s[3..]; //上記と同じ

let slice = &s[0..len];
let slice = &s[..]; //上記と同じ

println!("{}", slice);
```
# 5.構造体
```Rust
#![allow(unused)]
fn main() {
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

// インスタンス全体が可変でなければならない
// あるパラメータのみ可変は不可！
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    sign_in_count: 1,
    active: true,
};

// 構造体のインスタンスを作成
user1.email = String::from("anotheremail@example.com");
}
```

```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    println!("rect1 is {:?}", rect1);
}
```
### メソッド記法（impl）
```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```

# 6.Enum(列挙型)
取り得る値を列挙することで型の定義が可能
構造体の使い分け
enumの列挙子は、その識別子の元に名前空間分けされていること

## 簡単なプログラム(数当てゲーム)　一部追加
```Rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");
    let secret_number = rand::thread_rng().gen_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess
                            .trim()
                            .parse() {
            Ok(num) => num,
            Err(_) => {
                // エラー時のコメント追加
                // matchのアームで複数行にわたる場合は{}で囲う
                // このとき、;が必要
                println!("This input is incorrect! Please enter an integer.");
                continue;
            }
        };

        println!("You guessed: {}", guess);

        match guess.cmp(&secret_number) {
                Ordering::Less => println!("Too small!"),
                Ordering::Greater => println!("Too big!"),
                Ordering::Equal => {
                    println!("You win!");
                    break;
                }
            }
    }
}
```
### matchを使用したFizzbuzz
```Rust
fn main() {
    for number in 1..101 {
        match number {
            number if number % 15 ==0 
            
            => println!("Fizzbuzz"),
            number if number % 3 == 0 => println!("Fizz"),
            number if number % 5 == 0 => println!("Buzz"),
            number => println!("{}", number),
        }
    }
}
```
ここまで書いた後で、色々探してたらとても良い記事を発見した
https://qiita.com/hinastory/items/543ae9749c8bccb9afbc

# 7.パッケージ、クレート、モジュール
モジュールシステム
└── パッケージ: my_project
    ├── クレート: my_crate (バイナリ)
    │   └── ソースコード: src/main.rs
    └── クレート: my_library (ライブラリ)
      　  └── ソースコード: src/lib.rs

モジュール(mod)
```Rust
mod front_of_house {
    // pubを付ける
    pub mod hosting {
        // pubを付ける
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    // 絶対パス
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    // 相対パス
    front_of_house::hosting::add_to_waitlist();
}

fn main() {}
```
上記のコードは、"use"を用いることでより短いコードに書き換えることが出来る

```Rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
```
下記のように関数までuseに持ち込むのは慣例的ではないため割けた方が良い
どこでadd_to_waitlistが定義されたのかが不明瞭になるため
```Rust
use crate::front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
    add_to_waitlist();
    add_to_waitlist();
}
```
一方、構造体やenum等をuseで持ち込む場合、フルパスを書くのが慣例的
```Rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
    }

fn main() {}
```
# ここまでZenで公開済

エイリアス指定
```Rust
#![allow(unused)]
fn main() {
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
    Ok(())
}

fn function2() -> IoResult<()> {
    // --snip--
    Ok(())
    }
}
```

glob演算子
```Rust
#![allow(unused)]
fn main() {
use std::collections::*;
}
```
# 一般的なコレクション
### 8.1
新しいベクタの生成
```Rust
fn main() {
    let v: Vec<i32> = Vec::new();
}
```
```Rust
fn main() {
    let v = vec![1, 2, 3];
}
```


# エラー処理
# ジェネリック型、トレイト、ライフタイム
# 自動テスト


# 入出力プロジェクト
# イテレータとクロージャ
# cargo
# スマートポインタ
# 並行処理
# オブジェクト指向
# パターンマッチング
# 機能
# マルチスレッド処理
```Rust
```



