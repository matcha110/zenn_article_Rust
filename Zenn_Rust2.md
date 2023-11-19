# 9.エラー処理
回復可能なエラー->Result<T, E>値

プログラムが回復不能なエラー->実行を中止するpanic!マクロ

```Rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    // 失敗理由によって動作を変える
    let f = match f {
        // ファイルが存在し、ファイルを開くことが成功する場合
        Ok(file) => file,
        // ファイルが存在しない場合　かつ　ファイル作成は可能の場合
        // パターンマッチングの場合、引数にrefが必要
        Err(ref error) if error.kind() == ErrorKind::NotFound => {
            match File::create("hello.txt") {
                Ok(fc) => fc,

                // ファイルが存在しない場合　かつ　ファイル作成も不可能の場合
                Err(e) => {
                    panic!(
                        "Tried to create file but there was a problem: {:?}",
                        e
                    )
                },
            }
        },
        // ファイルは存在するが、開くことに失敗する場合
        Err(error) => {
            panic!(
                "There was a problem opening the file: {:?}",
                error
            )
        },
    };
}
```
expectを使用してエラー処理を書く場合
```Rust
use std::fs::File;

fn main() {
    // hello.txtを開くのに失敗しました
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

```Rust
#![allow(unused)]
fn main() {
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    // 文末の?はmatch文とほぼ同義
    // ?演算子は戻り値にResultを持つ関数でしか使用できない
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
    }
}
```
第2章の数当てゲームにエラー処理を追加
```Rust
use std::io;
use std::cmp::Ordering;
use rand::Rng;

// Guess型の宣言
// 独自の型を作るパターン
pub struct Guess {
    value: u32,
}

// Guess型の実装
// Guess型が1～100の数字であることをここで担保する
impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }
// ゲッター
    pub fn value(&self) -> u32 {
        self.value
    }
}

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        // Guess型の新しいインスタンスを作成
        let guess: Guess = match guess.trim().parse() {
            Ok(num) => Guess::new(num),// 変更前：Ok(num) => num,　
            Err(_) => continue,//不正な入力の処理
        };

        println!("You guessed: {}", guess.value());

        // Guess型の値を比較
        match guess.value().cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
//output
// Please input your guess.
//input
// 0
//output
// thread 'main' panicked at 'Guess value must be between 1 and 100, got 0.'
```
# 10.ジェネリック型、トレイト、ライフタイム
## 10.1 ジェネリック型 & 10.2 トレイト
トレイト境界の書き方の一例

見にくい例
```Rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
```

見やすい例
```Rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

// ジェネリクスを不使用の場合
// 各型ごとに関数を作成する

```Rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);
   assert_eq!(result, 100);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
   assert_eq!(result, 'y');
}

```

トレイトを使用してジェネリクな型に対して動くように修正した場合
共通の関数を作成する
```Rust
// Copy：moveするたびに所有権を取るのではなくコピーする（クローンも不要）
// PartialOrd：型に限らず比較することが可能となる
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}

```
## 10.3 ライフタイム
Rustにおいて、型推論と同様にライフタイムも暗黙的に推論する
ライフタイムの主な目的は、ダングリング参照を回避すること
ダングリング参照：存在しないデータへの参照をすること
ジェネリックな型引数、トレイト境界、ライフタイムを全て使用した例
```Rust
use std::fmt::Display;
// 2つの文字列のうち長い方を返すlongest関数
fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display, //トレイト境界の宣言
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest_with_an_announcement(
        string1.as_str(),
        string2,
        "Today is someone's birthday!",
    );
    println!("The longest string is {}", result);
}
```

# 11.自動テスト
テスト関数が行う3つの動作
    1.必要なデータや状態をセットアップする。
    2.テスト対象のコードを走らせる。
    3.結果が想定通りであることを断定（以下、アサーションという）する。

テストの例
```Rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        assert_eq!(2 + 2, 4);
    }
}
fn main() {}

// $ cargo test
//    Compiling adder v0.1.0 (file:///projects/adder)
//     Finished test [unoptimized + debuginfo] target(s) in 0.59s
//      Running target/debug/deps/adder-92948b65e88960b4

// running 1 test
// test tests::exploration ... ok

// test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

//    Doc-tests adder

// running 0 tests

// test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

```Rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        // 引数に関数を入れる場合
        assert_eq!(4, add_two(2));
    }
}

fn main() {}
```

### assert!マクロ
```Rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };
    }
    [test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle {
            width: 8,
            height: 7,
        };
        let smaller = Rectangle {
            width: 5,
            height: 1,
        };

        assert!(!smaller.can_hold(&larger));
    }
}

fn main() {}
```

第2章の数当てゲームに特定のパニックメッセージでpanic!を引き起こすことをテストする
```Rust
use std::io;
use std::cmp::Ordering;
use rand::Rng;
pub struct Guess {
    value: u32,
}

impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1 {
            panic!(
                //予想値は1以上でなければなりませんが、{}でした。
                "Guess value must be greater than or equal to 1, got {}.",
                value
            );
        } else if value > 100 {
            panic!(
                //予想値は100以下でなければなりませんが、{}でした。
                "Guess value must be less than or equal to 100, got {}.",
                value
            );
        }

        Guess { value }
    }
    pub fn value(&self) -> u32 {
        self.value
    }
}

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: Guess = match guess.trim().parse() {
            Ok(num) => Guess::new(num),
            Err(_) => continue,
        };

        println!("You guessed: {}", guess.value());

        match guess.value().cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    //予想値は100以下でなければなりません
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }

    #[test]
    //予想値は1以上でなければなりません
    #[should_panic(expected = "Guess value must be greater than or equal to 1, got 0.")]
    fn less_than_0() {
        Guess::new(0);
    }
}
```

Result<T, E>をテストで使用してパニックではなく、Errを返す
```Rust
#![allow(unused)]
fn main() {
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
}
```
テストを並行に実行する場合、またはテストケースを選ぶ場合
```Rust
// テストケースを並行に実行する場合(デフォルト)
cargo test

// テストケースを選ぶ場合
cargo test <テスト関数名>
ex.cargo test greater_than_100

cargo test <テスト関数名の一部>
// テスト関数名の一部を含むテストをすべて行う
ex.cargo test than
```
毎回は行いたくないテストケースがある場合（時間のかかるテスト等）

#[test]の下に#[ignore]を追加する
```Rust
#![allow(unused)]
fn main() {
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore] //追加
fn expensive_test() {
    // 実行に1時間かかるコード
    // code that takes an hour to run
}
}
```
逆に、[ignore]を指定したテストケースのみ行いたい場合
```Rust
cargo test -- --ignored
```

# 12.入出力プロジェクト
```Rust
use std::env;
use std::fs::File;　//ファイルを扱うために必要
use std::io::prelude::*; //ファイル入出力処理をするのに有用なトレイトを含む

fn main() {
    let args: Vec<String> = env::args().collect();
    // 引数の値を変数に保存する
    // プログラム名がベクタの最初の値であるargs[0]を占めているので、
    // 引数の1番目と2番目になることに注意する
    let query = &args[1];
    let filename = &args[2];

    // {}を探しています
    println!("Searching for {}", query);
    // {}というファイルの中
    println!("In file {}", filename);

    // ファイルが見つかりませんでした
    let mut f = File::open(filename).expect("file not found");

    let mut contents = String::new();
    f.read_to_string(&mut contents)
        // ファイルの読み込み中に問題がありました
        .expect("something went wrong reading the file");

    // テキストは\n{}です
    println!("With text:\n{}", contents);
}
```
### リファクタリングしてモジュール性とエラー処理を向上させる

Rustコミュニティにより定めるガイドラインは、以下とする
・バイナリプロジェクトの責任の分離
    1.プログラムをmain.rsとlib.rsに分け、ロジックをlib.rsに移動する。
    2.コマンドライン引数の解析ロジックが小規模な限り、main.rsに置いても良い。
    3.コマンドライン引数の解析ロジックが複雑化の様相を呈し始めたら、main.rsから抽出してlib.rsに移動する。

・main関数に残る責任は以下に限定する
    1.引数の値でコマンドライン引数の解析ロジックを呼び出す
    2.他のあらゆる設定を行う
    3.lib.rsのrun関数を呼び出す
    4.runがエラーを返した時に処理する

「1.プログラムをmain.rsとlib.rsに分け、ロジックをlib.rsに移動する。」
を行うための準備
コマンドライン引数解析ロジックを関数に分ける
```Rust
fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let filename = &args[2];

    (query, filename)
}
```
Config構造体を追加、関数呼び出しのまま
```Rust
use std::env;
use std::fs::File;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    let mut f = File::open(config.filename).expect("file not found");

    // --snip--
}

struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();

    Config { query, filename }
}
```
関数ではなく、インスタンスを生成する形に書き換える
```Rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    let config = Config::new(&args); //インスタンスの生成
    // --snip--
}

struct Config {
    query: String,
    filename: String,
}

// --snip--
impl Config {
    fn new(args: &[String]) -> Config {
        if args.len() < 3 {
        // 引数の数が足りません
        panic!("not enough arguments");
        }
        let query = args[1].clone();
        let filename = args[2].clone();

        Config { query, filename }
    }
}
```

panic!を呼び出す代わりにnewからResultを返すエラー処理に書き換える
```Rust
impl Config {
    fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config { query, filename })
    }
}
```


# 13.イテレータとクロージャ

# 14.cargo

# 15.スマートポインタ

# 16.並行処理

# 17.オブジェクト指向

# 18.パターンマッチング

# 19.高度な機能

# 20.マルチスレッド処理

```Rust

```