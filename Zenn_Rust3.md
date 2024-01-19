# 13.イテレータとクロージャ
クロージャ：変数に保存できる関数に似た文法要素

The bookの補助資料として、下記の動画を参考にした
参考：https://www.youtube.com/watch?v=tw2WCjBTgRM&t=11265s

```Rust
fn main(){

fn add_one_v1 (x:u32) -> u32{
    x+1
}

let add_one_v2 = |x: u32| -> u32 { x + 1 };
// 型推論出来るため、型は省略可
let add_one_v3 = |x| x + 1;

println!("add_one_v1: {}", add_one_v1(1));
println!("add_one_v2: {}", add_one_v2(1));
println!("add_one_v3: {}", add_one_v3(1));
// 一度推論すると型は確定するため、整数型のあと、浮動小数点型には変更できない
println!("add_one_v3: {}", add_one_v3(1.0)); //error
}
```

イテレータ：一連の要素を処理する方法
```Rust
let v1 = vec![1, 2, 3];
let v1_iter = v1.iter();

for val in v1_iter {
    println!("Got: {}", val);
}
// output
// Got: 1
// Got: 2
// Got: 3
```

イテレータを消費するメソッド
例えば、sumは呼び出し対象のイテレータの所有権を奪うので、sum呼び出し後にv1_iterを使用することはできない
```Rust

#![allow(unused)]
fn main() {
#[test]
fn iterator_sum() {
    let v1 = vec![1, 2, 3];
    let v1_iter = v1.iter();
    let total: i32 = v1_iter.sum();

    assert_eq!(total, 6);
}
}
```

イテレータを別の種類のイテレータに変えるメソッド
```Rust
#![allow(unused)]
fn main() {
let v1: Vec<i32> = vec![1, 2, 3];
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

assert_eq!(v2, vec![2, 3, 4]);
}
```

クロージャとイテレータを使用した例
```Rust
#![allow(unused)]
fn main() {
#[derive(PartialEq, Debug)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_my_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter()
        .filter(|s| s.size == shoe_size) //s.size == shoe_sizeのときのみベクタに戻す関数
        .collect()
}

#[test]
fn filters_by_size() {
    let shoes = vec![
        Shoe { size: 10, style: String::from("sneaker") },
        Shoe { size: 13, style: String::from("sandal") },
        Shoe { size: 10, style: String::from("boot") },
    ];

    let in_my_size = shoes_in_my_size(shoes, 10);

    assert_eq!(
        in_my_size,
        vec![
            Shoe { size: 10, style: String::from("sneaker") },
            Shoe { size: 10, style: String::from("boot") },
        ]
    );
}
}
```
参考サイト：https://doc.rust-lang.org/std/iter/trait.Iterator.html

独自のイテレータを作成する
```Rust
#![allow(unused)]
fn main() {
struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;

        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}

#[test]
fn calling_next_directly() {
    let mut counter = Counter::new();

    assert_eq!(counter.next(), Some(1));
    assert_eq!(counter.next(), Some(2));
    assert_eq!(counter.next(), Some(3));
    assert_eq!(counter.next(), Some(4));
    assert_eq!(counter.next(), Some(5));
    assert_eq!(counter.next(), None);
}
}
```
```Rust
#![allow(unused)]
fn main() {
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;

        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}

#[test]
fn using_other_iterator_trait_methods() {
    let sum: u32 = Counter::new().zip(Counter::new().skip(1))
                                 .map(|(a, b)| a * b) //(1*2, 2*3, 3*4, 4*5の組を作る)
                                 .filter(|x| x % 3 == 0) //条件を満たす2*3, 3*4の組のみを加算
                                 .sum(); //2*3 + 3*4 = 18
    assert_eq!(18, sum);
}
}
```

section12のプログラムをイテレータを用いたものに修正する

src/main.rs
```Rust
extern crate minigrep;

use std::env;
use std::process;

use minigrep::Config;

fn main() {
    let config = Config::new(env::args()).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });


    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {}", e);
        process::exit(1);
    }
}
```

src/lib.rs
```Rust
use std::fs::File;
use std::io::prelude::*;
use std::error::Error;
use std::env;

pub struct Config {
    pub query: String,
    pub filename: String,
    pub case_sensitive: bool,
}

// 修正前
// impl Config {
//     pub fn new(args: &[String]) -> Result<Config, &'static str> {
//         if args.len() < 3 {
//             return Err("not enough arguments");
//         }
        // clone()を2回使用しているが、極力避けたい
//         let query = args[1].clone();
//         let filename = args[2].clone();
//         let case_sensitive = env::var("CASE_INSENSITIVE").is_err(); //追加

//         Ok(Config { query, filename, case_sensitive})
//     }
// }

// 修正後(clone()を使用せずに同様のコードを書く)
impl Config {
    pub fn new(mut args: std::env::Args) -> Result<Config, &'static str> {
        args.next();

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };

        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config { query, filename, case_sensitive })
    }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    let results = if config.case_sensitive {
        search(&config.query, &contents)
    } else {
        search_case_insensitive(&config.query, &contents)
    };

    for line in results {
        println!("{}", line);
    }

    Ok(())
}

// 修正前
// pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
//     let mut results = Vec::new();

//     for line in contents.lines() {
//         if line.contains(query) {
//             results.push(line);
//         }
//     }

//     results
// }

// 修正後
// イテレータアダプタのメソッドを使用して簡潔に書く
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents.lines()
        .filter(|line| line.contains(query))
        .collect()
}

// テストの追加
#[warn(unused_imports)]
#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn one_result() {
        let query = "saf"; //検索したい単語
        let contents = "\
Rust:
safe, fast, productive.
Pick three.";

        assert_eq!(
            vec!["safe, fast, productive."], //検索単語が含まれる一文
            search(query, contents)
        );
    }
}

// PowerShellを使用している場合は、以下の2コマンドで実行する必要がある
// $env:CASE_INSENSITIVE=1
// cargo run to poem.txt
```

# 14.cargo

```Rust
cargo build //通常のビルド
cargo build --release //リリース用の設定でビルド
```
```Rust
// opt-levelはコンパイル時間は短いが、生成したコードの動作が遅くなるため、
// 開発時は0、リリース時は3を選択するのが一般的
[profile.dev]
opt-level = 0 //コンパイル時間は短いが、生成したコードの動作が遅い

[profile.release]
opt-level = 3 //コンパイル時間は長いが、生成したコード動作が速い
```

Crates.ioにクレートを一般公開する
```Rust
/// Adds one to the number given.
/// 与えられた数値に1を足す。
///
/// # Examples
///
/// ```
/// let five = 5;
///
/// assert_eq!(6, my_crate::add_one(5));
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

ファイル名: Cargo.toml
```Rust
[package]
name = "guessing_game"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
description = "A fun game where you guess what number the computer has chosen."
              (コンピュータが選択した数字を言い当てる面白いゲーム)
license = "MIT OR Apache-2.0"

[dependencies]
```
上記のように必要な情報をCargo.tomlに書き込み、下記コマンドでクレートを公開する
```Rust
cargo publish
```

# 15.スマートポインタ
### 1.BOX
スマートポインタの一つに"BOX<T>"がある。
ボックスを使うことで、スタックではなく、ヒープにデータを格納することが出来る。
その際、スタックに残るのはヒープデータへのポインタである。

スタックの場合と同じようにBox<T>を使ってヒープにデータを格納する例
```Rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```

Box<T>を参照のように使用する
```Rust
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}

// BOXを使用しない場合エラーが生じる
// 異なる型だから数値と数値への参照の比較は許されていない
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

### 2.Deref
Derefトレイト(標準ライブラリ)を実装して型を参照のように扱う
deref：selfを借用し、 内部のデータへの参照を返すメソッド
```Rust
#![allow(unused)]
fn main() {
use std::ops::Deref;

struct MyBox<T>(T);
impl<T> Deref for MyBox<T> {
    type Target = T; //関連型を定義

    fn deref(&self) -> &T {
        &self.0
    }
}
}
```

参照外し型強制が可変性と相互作用する方法
Derefトレイトを使用して不変参照に対して*をオーバーライドするように、 DerefMutトレイトを使用して可変参照の*演算子をオーバーライドすることは可能である。

具体的な例を以下に３つ挙げる
T: Deref<Target=U>の時、&Tから&U　=　不変参照 -> 不変参照
T: DerefMut<Target=U>の時、&mut Tから&mut U　=　可変参照 -> 可変参照
T: Deref<Target=U>の時、&mut Tから&U　=　可変参照 -> 不変参照
一方で、不変参照　-> 可変参照は不可である。

### 3.Drop

```Rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        // CustomSmartPointerをデータ`{}`とともにドロップするよ
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };      // 俺のもの
    let d = CustomSmartPointer { data: String::from("other stuff") };   // 別のもの
    println!("CustomSmartPointers created.");                           // CustomSmartPointerが生成された
}

```

### 4.Rc<T>
Rc形：reference counting(参照カウント)の省略形
```Rust
enum List {
    // Cons(i32, Box<List>),
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    // let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));　//BOXではなくRc型を使用する
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}

```

### 5.RefCell<T>
RefCell<T>型は、Rc<T>と異なり保持するデータに対して単独の所有権を表す
RefCell<T>を抱えるRc<T>があれば、 複数の所有者を持ち可変化できる値を得ることができる

```Rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}

// output
// a after = Cons(RefCell { value: 15 }, Nil)
// b after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
// c after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

### 6.循環参照
```Rust
continue;
```

# 16.並行処理
### スレッドを生成して、複数のコードを同時に走らせる方法
【スレッドの簡単な説明】
OSでは実行中のプログラムのコードはプロセスで走り、OSは同時に複数のプロセスを管理する。
自分のプログラム内で、独立した部分を同時に実行でき、その独立した部分を走らせる機能をスレッドと呼ぶ。
プログラム内の計算を複数のスレッドに分けると同時に複数の作業を行うためパフォーマンスが改善する。
一方で複雑度が増すという害もある。

スレッドを同時に走らせることが出来るので、異なるスレッドのコードが走る順番に関して保証はない。
そのため、以下のような問題が生じる可能性もある。
・スレッドがデータやリソースに矛盾した順番でアクセスする競合状態
・2つのスレッドがお互いにもう一方が持っているリソースを使用し終わるのを待ち、両者が継続するのを防ぐデッドロック
・特定の状況でのみ起き、確実な再現や修正が困難なバグ

言語がOSのAPIを呼び出してスレッドを生成するこのモデルを時に1:1と呼び、1つのOSスレッドに対して1つの言語スレッドを意味する。

【spawnで新規スレッドを生成】
```Rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
// output
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the main thread!
hi number 2 from the spawned thread!
hi number 3 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!

```

```Rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    handle.join().unwrap(); //この行を追加することで出力が混ざらないようになる

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
// output
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
```

# 17.オブジェクト指向
カプセル化
構造体を用いた実装例
```Rust

#![allow(unused)]
fn main() {
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}
impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            },
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
}

```
Rustでは、継承ではなくトレイトプロジェクトを使用して多相性を可能にする


# 18.パターンマッチング
### matchアーム
```Rust
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```
### 条件分岐if let式
```Rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        // あなたのお気に入りの色、{}を背景色に使用します
        println!("Using your favorite color, {}, as the background", color);
    } else if is_tuesday {
        // 火曜日は緑の日！
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            // 紫を背景色に使用します
            println!("Using purple as the background color");
        } else {
            // オレンジを背景色に使用します
            println!("Using orange as the background color");
        }
    } else {
        // 青を背景色に使用します
        println!("Using blue as the background color");
    }
}

```
### while let条件分岐ループ
```Rust

#![allow(unused)]
fn main() {
let mut stack = Vec::new();

stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);
}
}
```
### forループ
```Rust

#![allow(unused)]
fn main() {
let v = vec!['a', 'b', 'c'];

for (index, value) in v.iter().enumerate() {
    println!("{} is at index {}", value, index);
}
}
```
### 関数の引数
```Rust

#![allow(unused)]
fn main() {
fn foo(x: i32) {
    // コードがここに来る
    // code goes here
    }
}

```

# 19.高度な機能
### Unsafe Rust
基本的にはRustのメモリ安全保証がコンパイル時に強制されている。

しかしRustにはメモリ安全保証を強制しない第2の言語(Unsafe Rust)として用いることもできる。

#### unsafe Rustのメリット
- 生ポインタを参照外しすること
- unsafeな関数やメソッドを呼ぶこと
- 可変で静的な変数にアクセスしたり変更すること
- unsafeなトレイトを実装すること

ただし、unsafeコードで参照を使用しても安全性チェックは行う

unsafeコードの例
```Rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

### 高度なトレイト

### 高度な型

### 高度な関数とクロージャ

### マクロ


# 20.マルチスレッド処理
### シングルスレッドのWebサーバ
### シングルスレッドサーバ→マルチスレッドサーバ




```Rust

```
