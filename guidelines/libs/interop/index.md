{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# ライブラリ / 相互運用性のガイドライン {#libraries--interoperability-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 型は Send にする (M-TYPES-SEND) { #M-TYPES-SEND }

<why>Tokio やランタイムの抽象化の背後で使えること。</why>

互換性のため、公開型は `Send` にしてください。

- 生成されるフューチャは (明示的か暗黙的かを問わず) すべて `Send` でなければならない
- その他のほとんどの型も `Send` にするべきだが、例外もありうる

### フューチャ {#futures}

フューチャを明示的に宣言するときは、それが `Send` であり、今後も `Send` であり続けることを保証してください。

```rust
# use std::future::Future;
# use std::pin::Pin;
# use std::task::{Context, Poll};
#
struct Foo {}

impl Future for Foo {
    // 自分の型に対する `Future` の明示的な実装
    # type Output = ();
    #
    # fn poll(self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<<Self as Future>::Output> { todo!() }
}

// 型が `Send` であることをアサートする
const fn assert_send<T: Send>() {}
const _: () = assert_send::<Foo>();
```

`async` メソッドの呼び出しを通じてフューチャを暗黙的に返す場合も、それが `Send` であることを確かめてください。
すべてのメソッドをテストする必要はありませんが、少なくとも主要なエントリポイントは検証してください。

```rust,edition2021
async fn foo() { }

// TODO: これもマクロとして用意したい
fn assert_send<T: Send>(_: T) {}
_ = assert_send(foo());
```

### 普通の型 {#regular-types}

ほとんどの普通の型は `Send` にしてください。そうしないと、`.await` ポイントをまたいで保持されたときにフューチャに感染し、フューチャを `!Send` にしてしまいます。

```rust,edition2021
# use std::rc::Rc;
# async fn read_file(x: &str) {}
#
async fn foo() {
    let rc = Rc::new(123);      // <-- これを .await ポイントをまたいで保持すると、
    read_file("foo.txt").await; //     フューチャが `Send` でなくなる。
    dbg!(rc);
}
```

とはいえ、その型のデフォルトの使い方が _その場限り_ で、`.await` の境界をまたいで保持される理由がないなら、`!Send` でもかまいません。

```rust,edition2021
# use std::rc::Rc;
# struct Telemetry; impl Telemetry { fn ping(&self, _: u32) {} }
# fn telemetry() -> Telemetry  { Telemetry }
# async fn read_file(x: &str) {}
#
async fn foo() {
    // ここでは仮の Telemetry インスタンスを呼び出して
    // その場で使っている。Telemetry は !Send でもかまわないだろう。
    telemetry().ping(0);
    read_file("foo.txt").await;
    telemetry().ping(1);
}
```

> ### <tip></tip> Send のコスト {#the-cost-of-send}
>
> 理想的には、ワークスティーリング型のランタイムでは `Send` になり、thread-per-core モデルでは `Rc` や `RefCell` のような
> 非アトミックな型に基づいて `!Send` になる、そんな抽象化があればよいでしょう。
>
> 実際にはそのような抽象化は存在せず、非アトミックな場合には Tokio との互換性が失われます。つまり、thread-per-core の世界で
> 何かを成し遂げるには「世界を作り直す」必要があることになります。
>
> 良い知らせもあります。ほとんどの場合、アトミックや競合のないロックが測定できるほどの影響を持つのは、おおよそ 64 ワードごとよりも頻繁にアクセスした場合だけです。
>
> <div style="background-color:white;">
>
> ![TEXT](M-TYPES-SEND.png)
>
> </div>
>
> ホットループの中で大きな `Vec<AtomicUsize>` を扱うのは悪い考えです。しかし、それ以外は thread-per-core である非同期コードで、
> 競合のないアトミック操作をときどき行う程度なら、パフォーマンスへの影響はなく、それでいてエコシステムとの幅広い互換性が得られます。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ネイティブへのエスケープハッチを用意する (M-ESCAPE-HATCHES) { #M-ESCAPE-HATCHES }

<why>代替手段ができるまでの、サポート外のユースケースへの回避策。</why>

ネイティブハンドルをラップする型は、`unsafe` なエスケープハッチを用意してください。相互運用の場面では、ユーザがネイティブハンドルを別の場所から
得ていたり、ラップされたハンドルを FFI 越しに渡す必要があったりします。こうしたユースケースに対応するため、`unsafe` な変換メソッドを用意してください。

```rust
# type HNATIVE = *const u8;
pub struct Handle(HNATIVE);

impl Handle {
    pub fn new() -> Self {
        // API 呼び出しを通じて安全にハンドルを生成する
        # todo!()
    }

    // ユーザが別の場所で得たネイティブハンドルから新しい Handle を構築する。
    // このメソッドでは、満たさなければならない安全性の要件も
    // すべてドキュメントに書くこと。
    pub unsafe fn from_native(native: HNATIVE) -> Self {
        Self(native)
    }

    // ネイティブハンドルを恒久的または一時的に取り出すための
    // さまざまな追加メソッド。
    pub fn into_native(self) -> HNATIVE { self.0 }
    pub fn to_native(&self) -> HNATIVE { self.0 }
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 外部の型を漏らさない (M-DONT-LEAK-TYPES) { #M-DONT-LEAK-TYPES }

<why>安定した API と、長期的に低い保守コスト。</why>

可能であれば、公開 API では外部クレート由来の型よりも `std`<sup>1</sup> の型を使ってください。例外は慎重に検討してください。

公開 API に含まれる型は、どれもその API の契約の一部になります。デフォルトで同梱されるクレートは `std` とその構成要素だけであり、
しかも永続的な安定性が保証されているので、相互運用性のリスクがない型はそれらの型だけです。

あるクレートが別のクレートの型を公開することを、その型を _漏らす_ (leak する) といいます。

長期的な安定性を最大にするなら、理論上、クレートはどんな型も漏らすべきではありません。実際には、ある程度の漏れは
避けられず、ときには有益でさえあります。次の経験則に従うことをおすすめします。

- [ ] 避けられるなら、サードパーティの型を漏らさない
- [ ] アンブレラクレートの一部であれば、<sup>2</sup> 兄弟クレートの型は自由に漏らしてよい
- [ ] 関連するフィーチャフラグの背後でなら、型を漏らしてもよい (例: `serde`)
- [ ] フィーチャなしで漏らすのは、_相当な利点_ がある場合 _のみ_。最もよくあるのは、それらの型を中心に成り立っている
      Rust エコシステムの重要な部分との相互運用性である

<footnotes>

<sup>1</sup> まれに、たとえば組み込みで使われる高性能ライブラリでは、`core` だけに限定したい場合もあります。

<sup>2</sup> たとえば、`runtime` クレートが `runtime_rt`、`runtime_app`、`runtime_clock` のアンブレラになることがあります。ユーザは
アンブレラとだけやりとりすることが想定されるので、兄弟クレート同士は互いの型を漏らしてかまいません。

</footnotes>
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 要素は元のクレートから取り込む (M-FOREIGN-REEXPORTS) { #M-FOREIGN-REEXPORTS }

<why>型の同一性が曖昧にならないこと。</why>

クレートは一般に、他のクレートの要素を再エクスポートすべきではありません。たとえば、クレートに `foo::download(url: bar::Url)` というメソッドがあるとしても、`foo` の中から `pub use bar::Url` をしてはいけません。こうすることで、何十ものエイリアスが混在するのを避けられます。エイリアスが混在すると、ユーザにとってもエージェントにとってもわかりにくくなります。特に、他のクレートにある同名の本当に別の型と混ざっている場合はなおさらです。

クレートがサードパーティのクレートで定義された型を受け取ったり返したりする場合、ユーザはそのサードパーティのクレートに直接依存し、そこから型をインポートすることが想定されます。とはいえ、このルールにはいくつか正当な例外があります。

- アンブレラクレート ([M-DONT-LEAK-TYPES](./#M-DONT-LEAK-TYPES) 参照) は、定義上、他の型を再エクスポートする
- 技術的な理由で分割されたクレート (例: `foo` から `foo_core::Url` をエクスポートする)
- 安定したパスを提供するためのマクロでの利用。例: 隠された `foo::__private::Url` を通じて提供する
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 可能な場合は `impl AsRef<>` を受け取る (M-IMPL-ASREF) { #M-IMPL-ASREF }

<why>呼び出し側が自分の型を使える柔軟性。</why>

**関数** のシグネチャでは、[参照の階層が明確な](https://doc.rust-lang.org/stable/std/convert/trait.AsRef.html#implementors) 型について、
所有権を取る必要がない場合や、オブジェクトの生成が比較的安価な場合に、`impl AsRef<T>` を受け取ってください。

| 代わりに ... | 受け取るもの ... |
| --- | --- |
| `&str`, `String` | `impl AsRef<str>` |
| `&Path`, `PathBuf` | `impl AsRef<Path>` |
| `&[u8]`, `Vec<u8>` | `impl AsRef<[u8]>` |

```rust,ignore
# use std::path::Path;
// 間違いなく `AsRef` を使う。関数は所有権を必要としない。
fn print(x: impl AsRef<str>) {}
fn read_file(x: impl AsRef<Path>) {}
fn send_network(x: impl AsRef<[u8]>) {}

// さらに検討が必要。これらの場合、関数は `String` や `Vec<u8>` の
// 所有権を欲しがっている。それが「低頻度・低容量」の関数なら
// `AsRef` のほうがエルゴノミックで、そうでなければ `String` や
// `Vec<u8>` を受け取るほうがパフォーマンスが良い。
fn new_instance(x: impl AsRef<str>) -> HoldsString {}
fn send_to_other_thread(x: impl AsRef<[u8]>) {}
```

一方、**型** は一般に、こうした境界に感染させるべきではありません。

```rust,ignore
// 一般によくない。パフォーマンス上の理由で例外はありうるが、
// それがユーザに見えるべきではない。
struct User<T: AsRef<str>> {
    name: T
}

// より良い例
struct User {
    name: String
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 可能な場合は `impl RangeBounds<>` を受け取る (M-IMPL-RANGEBOUNDS) { #M-IMPL-RANGEBOUNDS }

<why>範囲を指定するときの柔軟さとわかりやすさ。</why>

数値の範囲を受け取る関数は、自前のパラメータではなく `Range` の型やトレイトを使わなければなりません。

```rust,ignore
// 悪い例
fn select_range(low: usize, high: usize) {}
fn select_range(range: (usize, usize)) {}
```

加えて、任意の範囲を扱える関数は、`Range<T>` ではなく `impl RangeBounds<T>` を受け取ってください。

```rust
# use std::ops::{RangeBounds, Range};
// 呼び出し側は `select_range(1..3)` のように呼ぶ必要がある
fn select_range(r: Range<usize>) {}

// 呼び出し側は次のように呼べる
//     select_any(1..3)
//     select_any(1..)
//     select_any(..)
fn select_any(r: impl RangeBounds<usize>) {}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 可能な場合は `impl 'IO'` を受け取る ('sans IO') (M-IMPL-IO) { #M-IMPL-IO }

<why>I/O から切り離されたビジネスロジックと、N*M 通りの合成可能性。</why>

初期化時に一度きりの I/O だけを行えばよい関数や型は、"[sans-io](https://www.firezone.dev/blog/sans-io)" で書いてください。
つまり、`T` を適切な I/O トレイトとして `impl T` を受け取り、事実上 I/O の作業を別の型に任せます。

```rust,ignore
// 悪い例。与えられたデータをパースするのに、呼び出し側が File を用意しなければ
// ならない。データがネットワークから来るなら、まずディスクに書き出す必要がある。
fn parse_data(file: File) {}
```

```rust
// ずっと良い例。次のものを受け取れる
// - File
// - TcpStream
// - Stdin
// - &[u8]
// - UnixStream
// ... ほか多数。
fn parse_data(data: impl std::io::Read) {}
```

同期関数では [`std::io::Read`](https://doc.rust-lang.org/std/io/trait.Read.html) と
[`std::io::Write`](https://doc.rust-lang.org/std/io/trait.Write.html) を使ってください。複数のランタイムを対象とする非同期 _関数_ では、
[`futures::io::AsyncRead`](https://docs.rs/futures/latest/futures/io/trait.AsyncRead.html) などを使ってください。
ランタイム固有の継続的な I/O を行う必要がある _型_ は、代わりに [M-RUNTIME-ABSTRACTED] に従ってください。

[M-RUNTIME-ABSTRACTED]: ./#M-RUNTIME-ABSTRACTED
{% endraw %}
