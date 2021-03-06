---
title: Stringとstr入門
tags: Rust
author: Mizunashi_Mana
slide: false
---
文字列はプログラミングにおいての基本要素です。それはRustにおいても例外ではありません。

残念なことに、文字列にはつまづきやすいポイントが詰まっています。ざっと挙げてみましょう。

* 文字コードと改行コードについて
* エスケープシーケンス
* 文字列フォーマットについて
* 正規表現について
* ANSI Terminalシーケンスとカラーライズについて
* 列の一種であること
* etc.

みなさん、どれも一度はつまづいたことがあるんじゃ無いでしょうか？少なくとも僕はつまづきました。

さて、どれも言語問わず共通の話題だと思いますが、今回はこのうちのRustでつまづきやすいのでは無いかと思う部分、文字列が列の一種であることに焦点を当てたお話をします。Rust入門者の人の助けになればと思います。

## 文字列を扱うのは至難の技

文字列は列の一種です。プログラミング言語によって、その扱いは様々ですが、静的・動的、GCのあるなし、プログラミングスタイルの違いを問わず、列であることによって苦労していないという言語はなさそうです。

例えば、Rubyでは[3.0から文字列リテラルをデフォルトでfreezeさせることが決まりました](https://bugs.ruby-lang.org/issues/11473)。これはパフォーマンス上の理由からです。パフォーマンスといえば、Haskellでも古くから、標準で提供されているString型と別に、準標準として[Textパッケージ](https://hackage.haskell.org/package/text)というものが提供されており、実務上は多くの場合Stringではなくこちらが利用されます。C++でも標準でstringサポートがありますが、Qtで提供されているQStringと文字コードとの兼ね合いからAPIの利用方法が異なり、QStringの方が扱いやすいなど[議論](http://www.cplusplus.com/forum/beginner/188810/)が絶えませんね;)

言語の管理者の悩みの種は幾つかありますが、最も顕著なのは以下の点についてです。

* サイズが固定されていない
  * intなどと違って、値によってメモリ上のサイズが異なる
* 内部表現によって、時間と空間、扱いやすさのトレードオフが大きい
  * UTF-8表現は空間はおおよそ効率がいいが、長さ計算が遅い
  * UTF-16表現はバランスが良いが、各文字コードへの変換が遅い（普通ファイルなどをUTF-16で管理しない）
  * 内部表現を配列（リスト）で提供すると扱いやすいが、効率が悪い
* 変更されるかされないかによって最適化のしやすさが異なる
  * Rubyの例
  * リテラルはバイト列としてそのままプログラムに埋め込まれる場合が多いが、それをいちいち実行時に文字列表現に持ち上げるのは、効率が悪い

そして、これはRustについてもメインテーマです。特にRustではGCがないため、サイズが変化するデータは、C++などと同じくとても扱いにくいものです。そのため、RubyやJavaScriptでいとも簡単に扱えたものが、非常に複雑になっているように見えるでしょう。

特に文字列は基本的な要素であるため、その複雑さが入門者にとって大きな障壁になると思います。しかしながら、この難解さには以上のような背景があることは押さえておいてほしいと思います。

では、Rustについての話を始めましょう。

## Rustに存在する二つの文字列を表す型

Rustには文字列を表す型が二つあります。

一つはプリミティブ型のstr、もう一つが標準ライブラリが提供するStringです。といってもそれほど大した違いはありません。

str型はu8のスライス、String型はu8のベクタと同じです。ただ両方とも、列の内容としてUTF-8しか許可していないというその一点だけが異なります。なので、実質並びに制限のある8byteの配列スライスとベクタです。strの方はプリミティブ型ですが、Stringの方は標準ライブラリの提供型なので実装が単純に見て取れます。

```rust
pub struct String {
    vec: Vec<u8>,
}
```
from https://github.com/rust-lang/rust/blob/1.13.0/src/libcollections/string.rs#L262

はい、もうまんまですね。ただし、メンバはpublicになっていないので直接内容にアクセスしたり、書き換えたりはできません。strもStringも、u8のバイト列などから作る方法が用意されていますが、UTF-8並びになっているかが検査されるようになっています[^for_unsafe_utf8]。strとStringの相互変換においては両方中身がUTF-8なわけですから検査は必要無いですね。というわけで、strとStringは構成されるときに構成要素がUTF-8になっている、ただのu8ベクタと扱いが同じというわけです。

そのようなわけで、Stringの方を文字列、strの方を文字列スライス、または単に文字列と対比してスライスと読んだりします。

[^for_unsafe_utf8]: もちろん、パフォーマンスの都合上unsafeなものを使うことで、検査無しに文字列にできるAPIも用意されています。

## 文字列と文字列スライスの違い

さて、念のためスライスとベクタ、ついでにややこしいので配列も一緒に、その違いについておさらいしておきましょう。

* 配列は、C/C++でおなじみの固定長配列
   * 長さnの配列は`[T; n]`という型表記を使い、`[m; n]`で全てをmに初期化、`[a1, a2, ...]`で順に初期化したインスタンスを生成できます
* ベクタは、他のプログラミング言語でおなじみのヒープに格納される可変長配列
   * `Vec<T>`という型表記を使い、`vec!`マクロや、`Vec::new()`などからインスタンスを生成できます
* スライスは、実行時に長さの決定する固定長配列
   * `[T]`という型表記を使い、配列やベクタからスライス表記を用いて作成できます
   * 実際にはビューとして機能し、他のデータ構造への参照になっています。必ず他のデータ構造を元に作られ、その際にデータのコピーは発生しません。

ベクタ以外はプリミティブ型です。

スライスは動的に長さが決定するので、何か変換処理をかますことで（長さをチェックしたりして）配列に変換することは可能ですが、単純には変換できないため配列への変換はあまりサポートしていません。その代わりベクタにはto_vecを使用することで変換が可能です。

配列やベクタからスライスへは、スライス表記を用いて簡単に変換することができます。

```rust
&[1, 2, 3][0..2]     // => [1, 2]: &[i32]
&vec!(1, 2, 3)       // => [1, 2, 3]: &[i32]
&vec!(1, 2, 3)[0..2] // => [1, 2]: &[i32]
```

さて、strは`&[u8]`と、Stringは`Vec<u8>`に対応します。つまり、上で述べた性質をこの二つの型も持っているというわけです。文字列の方もある程度まとめておきましょう。

* ベクタと対応する文字列
  * もちろん可変長です
  * Stringという型表記を使い、文字列スライスからto_stringを使って作成できます
* 文字列スライス
  * strという型表記を使います。文字列リテラルは`&'static str`型になります
  * 必ず元となるデータがあります。Stringから作成したり、文字列リテラルの場合はプログラム中にそのままバイト列として埋め込まれます

スライスやベクタと本質的に同じデータですが、UTF-8しか許容していないためバイト列などはstrとStringでは扱えません。そのためRustでは、バイト列は、文字列型で杜撰に管理するようなことはなく、`&[u8]`や`Vec<u8>`で管理します。

そのような背景もあって、バイト列管理への気兼ねが完全に取り払われており、UTF-8に内容が制限されている他に、strとStringは本来のスライスやベクタに無い機能を提供していたり、制限があったりもします。その際たるものが、インデックスアクセスを直接はできないというものです。

```rust
"hello"[0] // error!
"あいうえお".as_bytes()[0] // => 227: u8
"あいうえお".chars().nth(0) // => Some('あ'): Option<char>
```

strはu8のスライスだということを思い出してください。これは中身がUTF-8であると保証されているとはいえ、n番目のu8データが文字であるかどうかは分かりません。多バイト領域の一部を表しているかもしれないのです。そのため、単純にn番目の文字を取り出すことはできません。Rustでは文字列のインデックスアクセスに、u8のデータとしてとるか、文字としてとるかを明示する必要があります。Rustではそれに加えて、文字データにアクセスするのはコストであることを明示するために、文字データのアクセス用にはイテレータを用意しています。

ただ、スライスとベクタに対応するところも残してあります。例えば、Stringにはスライス表記が使えます。ただ気をつけたいのは、スライス表記でのインデックスはバイト列上のもので、きちんとUTF-8上の正しい区切り目を指定してあげる必要があるということですが。

```rust
&String::from("あいうえお")[0..6] // => "あい": &str
&String::from("あいうえお")[0..5] // error!
```

strからStringへの変換はto_vecと対応するto_stringが用意されています。

さて、スライスとベクタの関係と同じということは、パフォーマンス上とても重要なトピックです。文字列と文字列スライスの対比としてよく出されるのが次の例です。

```rust
let string: String = ...

string == "hello".to_string()
&string == "hello"
```

上記の比較はどちらも同じ結果をもたらします。書き方は圧倒的にスライスに変換する方がto_stringするよりも短いです。しかしながら、といってもたかが10文字ほどです。しかしながら、パフォーマンス上ではスライスの作成はコピーが発生しませんが、Stringの作成はスライスの内容を一旦ベクタに変換する必要があります。実際にはPartialEqがどちらも設定されていますから、単純に比較するだけで大丈夫です。

```rust
string == "hello"
```

ですが、似たような状況はいくつも発生し今回のPartialEqのようなものが用意されているとは限りません。このコードのパフォーマンス上の違いについて頭に入れておくことは、Rustを書く上で重要なことです。そして、これが文字列と文字列スライスの最大の違いであり、そして配列やベクタとスライス、より広げて考えればオブジェクトと参照の違いにおいてとても重要なことです。

少しは文字列と文字列スライスについての理解が進んだでしょうか？では、最後に様々なケースにおける文字列と文字列スライスの対比を幾つか見てみましょう。

## Case Examples

### 関数の引数において

例えば、次のような関数を考えてみます。

```rust
fn print_str(str: String) {
  println!("String: {}", str);
}
```

実際に使ってみましょう。

```rust
print_str("Hello".to_string()); // => "String: Hello"
```

ちゃんとコンパイルは通りますし、実行もできますし、問題は無さそうです。。。本当に問題は無いでしょうか？

ちょっと、`print_str`関数を次のように変更してみましょう。

```rust
fn print_str(mut str: String) {
  println!("String: {}", str);
}

let string = "Hello".to_string();
print_str(string);
```

少しワーニングが出ますが問題なくコンパイルでき、動きました。あれ、mutableな値しか受け付けないはずなのに、immutableな値が渡せる！。。。とはさすがにもうなる人はいないですかね[^notice_less_knowledge]。はい、この関数は引数を受け取った時それをmoveして利用するのです。ですが、ここでは別にそのデータを何か操作するわけではなくただ単に読み込んで表示するだけですから所有権の移動は必要ありません。

一般に、引数に渡された変数を読み込んで何かする関数は、内部で何かそのデータをさらに加工して使うというのでも無い限り、文字列スライスを指定することが良いとされています。これは、上の方でも説明した、比較においてコピーが発生するto_stringよりもas_sliceの方が良いというのと同じですね[^generic_str]。ということで書き直してみます。

```rust
fn print_str(str: &str) {
  println!("String: {}", str);
}

print_str("Hello");
```

では、引数にStringを指定したい時はどんな時でしょう？一つのケースとしてはString自体を破壊的に変更したい時、もう一つはStringそのものではなく、Stringを元とするデータ（例えばStringの配列など）が必要な時です。例えば、Stringの後ろに文字列をさらに付け加える関数を作ってみましょう。

```rust
fn append_world(str: &mut String) {
  str.push_str(" World");
}

let mut string = "Hello".to_string();
append_world(&mut string);
println!("{}", string); // => Hello World
```

[^generic_str]: もっとアドバンスなこととして、ジェネリックにStrやAsRefなどを使う技とかもありますが、そういうのの解説は他の人に任せておきましょう！（チラッチラッ
[^notice_less_knowledge]: すいません、筆者が普通にハマりました

### 関数の返り値において

またまた、一つ関数を考えてみましょう。

さっきの`append_world`を今度は元データを壊すのでなく、新しい文字列を返すようにしてみます。

```rust
fn append_world(str: &str) -> &str {
  let mut result = String::with_capacity(str.len() + 6);
  result.push_str(str);
  result.push_str(" World");
  return &result;
}

println!("{}", append_world("Hello"));
```

コンパイルすると怒られてしまいました。よく考えてみてください。所有権の基本作用です。変数resultは関数内のスタックに入れられており、関数を出ると解放されてしまってどこにもいなくなってしまいます。C/C++でよくやるミスですね。一般に、返り値が引数から作られていたとしても、全く新たなデータの場合Stringで返すのが得策です。では、ちょっと書き直してみましょう。

```rust
fn append_world(str: &str) -> String {
  let mut result = String::with_capacity(str.len() + 6);
  result.push_str(str);
  result.push_str(" World");
  return result;
}

println!("{}", append_world("Hello"));
```

動きましたね[^borrow_and_cow]。では、strを返して方がいいパターンも紹介しておきましょう。

```rust
fn head_ascii(str: &str) -> &str {
  &str[0..1]
}

head_ascii("hello") // => "h"
```

良い子は絶対真似をしないでねって感じのコードですが、今回は解説用なので大目に見てもらいましょう[^note_head_ascii]。このように、何か変更を加えるわけではなく、中身を条件に合わせて切り出すような場合は、strを使う方が効率的です。

[^borrow_and_cow]: こちらもアドバンスなこととして、もっと効率良くデータを扱うために、borrowモジュールを活用する方法などがありますが、こちらも誰か解説してくれるでしょう！（チラッチラッ
[^note_head_ascii]: ま、コンパイル通るし多少はね？本来はちゃんと長さを判定してOptionで包んだり、panicが起こることをコメントに書いたりするべきです

### 構造体のメンバにおいて

構造体のメンバとして文字列を格納したいことは多々あります。では、何か適当に作ってみましょう。

```rust
struct Person {
  name: &str,
}

let person = Person { name: "Bob" };
```

あらら、さっそくコンパイラに怒られてしまいました。構造体に参照を入れる場合、ライフタイムの明示的な指定が必要ですね。指定してみます。

```rust
struct Person<'a> {
  name: &'a str,
}

let person = Person { name: "Bob" };
```

おー、コンパイル通りました。しかしながら、このデータには若干問題があります。Personのデータはもちろん名前を変えたいですし、ミュータブルにしたい時もイミュータブルにしたい時もあります。もちろん、データをコピーしたい時もありますね。そういった場合に名前がスライスになっているのはどうでしょうか？名前がスライスということは、名前自体のデータは構造体内ではなくどこか別のところにあるということになります。それは不便ですね。

一般的に構造体のメンバに文字列を入れたい場合はStringを使います。文字列リテラルを使用する際は、いちいちto_stringを使わなければいけませんが。

```rust
struct Person {
  name: String,
}

let person = Person { name: "Bob".to_string() };
```

ここらへんは、C/C++の事情とあまり変わりありませんね。

## 最後に

以上、Stringとstrの入門でした。

Rustは、本当に文献が良く、新生なのにも関わらず広く充実しています。そして、細かいところで今までのベストプラクティスから生まれた新たな試みが息づいている言語だと思います。

システムプログラミングを視野に入れていることから、多少スクリプト言語ばかりを触ってきた人たちには解りにくい要素もあるかもしれませんが、とてもいい言語なので是非触ってもらいたいと思います。

あと、本当に最後にですが、僕は実はRustそんなに触ってないので、間違いが結構あるかもしれません。すみませんが、そういう場合指摘してもらえるとありがたいです。宜しくお願いします。

## 参考文献

* https://doc.rust-lang.org/std/primitive.str.html
* https://doc.rust-lang.org/std/string/struct.String.html
* http://hazama1616.blogspot.jp/2016/03/crust-10.html
* http://stackoverflow.com/questions/24158114/rust-string-versus-str
* http://www.steveklabnik.com/rust-issue-17340/

