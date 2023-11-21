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
スマートポインタの一つに"BOX<T>"がある。
ボックスを使うことで、スタックではなく、ヒープにデータを格納することが出来る。
その際、スタックに残るのは、ヒープデータへのポインタである。

スタックの場合と同じようにBox<T>を使ってヒープにデータを格納する例
```Rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```
# 16.並行処理

# 17.オブジェクト指向

# 18.パターンマッチング

# 19.高度な機能

# 20.マルチスレッド処理

```Rust

```