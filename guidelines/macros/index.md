{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# マクロのガイドライン {#macros-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## マクロは最後の手段 (M-MACRO-LAST-RESORT) { #M-MACRO-LAST-RESORT }

<why>複雑さを最小限にするため。</why>

マクロは、ほかに有効な解決策がない場合にだけ使うべきです。次の言葉を参考にしてください。

> @littlecalculist がいつも言っていたように、「マクロは言語を使い切ったときのためのもの」です。まだ言語が残っているなら（そして Rust にはたくさんの言語があります）、まず言語を使ってください。
>
> @pcwalton

マクロは強力ですが、いくつかの欠点があります。マクロは

- 魔法のようなもので、何をするのか、どうやってするのかを予測できないことがあります。
- それまでマクロに頼っていなかったプロジェクトでは、コンパイル時間を不釣り合いに増やします。
- Rust の構文やセマンティクスが変わりうるエディションの境目で、わかりにくい破損を起こすことがあります。

直感に反して、マクロの展開結果が構造的に複雑であるほど、そもそもそれにマクロを使うのは悪い考えになります。理想的なマクロとは、ユーザに「_何が生成されるかは正確にわかるけれど、それを全部手で書きたくはない_」と思わせるものです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 手続き的マクロより「例によるマクロ」を選ぶ (M-EXAMPLE-OVER-PROC) { #M-EXAMPLE-OVER-PROC }

<why>マクロを調べやすく、コンパイルが速いため。</why>

「例によるマクロ」(macro by example) で用が足りるなら、手続き的マクロよりもそちらを選ぶべきです。

手続き的マクロのほうが強力ですが、その展開結果は簡単には調べられません。この柔軟性が要らない場面では、単純な「例によるマクロ」のほうが良い選択です。

```rust,ignore
// 悪い例: アトリビュートマクロには手続き的マクロの仕組みが必要で、
// IDE によっては中身を調べにくい。しかもここでは必要ない。
#[make_new_id]
struct MyId;

// 良い例: 書きやすく、保守しやすく、調べやすい。コンパイルも速い。
make_new_id!(MyId);
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## マクロはシグネチャを偽らない (M-MACROS-DONT-LIE) { #M-MACROS-DONT-LIE }

<why>ユーザと LLM にとってわかりやすくするため。</why>

マクロは、シグネチャやアイテムの形を偽ってはいけません（ユーザに偽らせてもいけません）。

マクロはトークンストリームを自由に書き換えられます。構造体を列挙型に、トレイトを関数に変えるなど、想像できるどんな変換もできます。しかし、そのどれもすべきではありません。できあがるコードは非常にわかりにくく、予測することも理解することもほぼ不可能になるからです。

特に、マクロは次のことをしてはいけません。

- データ型の性質を目に見える形で変える（例: 構造体を列挙型に、など）
- 関数のシグネチャを変える
- アイテムが `async` かどうかを変える
- そのほか、_書かれていること_ と _起きていること_ を大きく食い違わせること

```rust,ignore
// 悪い例: 余分なパラメータを追加し、関数を `async` にしている。
// コードを読んでも予測できない。
#[magic_transform]
fn foo() { }

foo(token).await
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## マクロはメインクレートを前提にする (M-MACRO-MAIN-CRATE) { #M-MACRO-MAIN-CRATE }

<why>マクロのロジックを単純にするため。</why>

手続き的マクロは、メインクレートを通して使われることを前提にしてよく（そうすべきで）、そのためのパスを出力します。

手続き的マクロを含むクレートは、技術的な理由から、次の 3 つに分けて提供するのが一般的です。

- `foo` - `foo_proc` のマクロを、追加のトレイトや型とともに再エクスポートするメインクレート
- `foo_proc` - `proc-macro = true` を持ち、`foo_proc_impl` のマクロを再エクスポートするファサード
- `foo_proc_impl` - マクロの実際の実装と単体テスト

場合によっては、さらに別のクレートが関わることもあります。作者は `foo` も `foo_proc` もその兄弟も、すべて使えるようにしたくなるかもしれません。その結果、再エクスポートの階層が複雑になったり、サードパーティのヘルパを使ったりすることになります。実際には、得られる UX の改善はわずかで、増える複雑さ（やコンパイル時間のオーバーヘッド）に見合わないのが普通です。そもそもエコシステムでは、こうした使い方をほとんどサポートしないのが通例だからです。

これはまた、クレートが別の名前でインポートされるような使い方をサポートしようとすべきではない、ということも意味します。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 補助の手続き的マクロクレートのバージョンを固定する (M-MACRO-VERSION-PIN) { #M-MACRO-VERSION-PIN }

<why>生成されたコードを、そのライブラリと互換に保つため</why>

対になる手続き的マクロクレートからマクロを再エクスポートするクレートは、それらを `=x.y.z` で自分自身とまったく同じバージョンに固定しなければなりません。各マクロクレートも、自分の対になるクレートを同じように固定しなければなりません。公開するときは、これらのクレートを 1 つのクレートとして扱い、まったく同じバージョンでまとめて公開しなければなりません。

バージョンを厳密に固定しないと、新しいマクロが古いバージョンのクレートと互換性のないコードを生成することがあります。その結果、生成されたコードの中で、予想外で原因を突き止めにくいコンパイルエラーが起きます。

M-MACRO-VERSION-PIN は、独立して使われるマクロライブラリには当てはまりません。

例:

```toml
# foo/Cargo.toml
[dependencies]
foo_macros = "=1.2.3"
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## サードパーティのアイテムは隠し `_private` モジュールから参照する (M-MACRO-HELPERS) { #M-MACRO-HELPERS }

<why>コンパイルの結果を予測しやすくするため。</why>

マクロの展開結果がサードパーティのアイテムを参照する必要がある場合、ホストクレートはそれらを隠しモジュールから再エクスポートし、マクロはそのモジュールを通したフルパスを出力するべきです。ユーザのクレートがサードパーティのクレートに直接依存していることを前提にしてはいけません。

たとえば、`bar` のトレイトを必要とするクレート `foo` は次のようにします。

```rust,ignore
#[doc(hidden)]
pub mod _private {
    pub use ::bar::Bar;
}

pub use foo_proc::my_macro;
```

そして `my_macro!` の実装は、このモジュールがあることを前提にしたコードを出力します。

```rust,ignore
impl ::foo::_private::Bar for MyType { ... }
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 手続き的マクロは、テストを含む別の実装クレートを持つべき (M-PROC-IMPL) { #M-PROC-IMPL }

<why>手続き的マクロを十分にテストできるようにするため。</why>

手続き的マクロは、`foo_proc` のようなクレートの中の薄いシムにするべきです。実際のトークンストリーム変換のロジックとそのテストは、別の普通のライブラリクレート（通常 `foo_proc_impl` と呼ばれます）に置き、そこに処理を委ねます。

手続き的マクロのクレートは特殊なので、`foo_proc` から単体テストやスナップショットテストを行うには、たいてい回避策が必要です。代わりに、`foo_proc_impl` クレートを用意することを検討してください。

```rust,ignore
use proc_macro2::TokenStream;

pub fn my_macro(attr: TokenStream, item: TokenStream) -> TokenStream { ... }
```

これらには普通の [insta](https://insta.rs/) などのスナップショットテストを付けられます。そのうえで、次のように `foo_proc` クレートを通して本物の手続き的マクロとしてエクスポートします。

```rust,ignore
#[proc_macro_attribute]
pub fn my_macro(attr: TokenStream, item: TokenStream) -> TokenStream {
    foo_proc_impl::my_macro(attr.into(), item.into()).into()
}
```

そして、マクロをコアクレートから再エクスポートします。

```rust,ignore
pub use foo_proc::my_macro;
```

コアクレートの中には、エラーメッセージが一貫していることを確かめるため、失敗する例を含む [trybuild](https://docs.rs/trybuild/latest/trybuild/) の UI テストを追加することもお勧めします。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 手続き的マクロは暗黙のアイテムや隠れたアイテムを生成しない (M-PROC-IMPLIED-ITEMS) { #M-PROC-IMPLIED-ITEMS }

<why>わかりやすいエラーと、正しい衛生性 (hygiene) と可視性のため。</why>

マクロは魔法のような型を勝手に定義するべきではありません。特に、公開された型や、名前空間の工夫に頼らない型を定義するべきではありません。

型を定義したいマクロもあります。たとえば次のようなものです。

```rust,ignore
#[my_macro]
struct UserType;

// これが次のように展開される

struct UserType;
struct ExtraType; 
impl UserType {
    fn foo() -> ExtraType { ... };
}
```

これはほぼ常に悪い考えです。理由はいくつかあります。

- 同じモジュール内のユーザ定義の型と衝突することがあります。
- 素朴に実装すると、同じマクロの別の展開結果と衝突することがあります。
- ユーザの命名規則とぶつかることがあります。
- ソースコード上では見えないので、必要な場所で再エクスポートし忘れやすくなります。

ユーザはこうした制約をある程度回避できますが、それはユーザが付き合わなければならない小さな悩みの種になります。しかも、無関係なコードをリファクタリングしているときに、何か月も経ってから表面化するかもしれません。

ただし、この規則には 1 つ例外があり、その UX は一般に許容できるものです。Rocket などのクレートで知られるようになった、[名前空間](https://doc.rust-lang.org/reference/names/namespaces.html)を重ねて使う方法です。

```rust,ignore
#[my_macro]
fn foo() { ... }

// これが次のように展開される

fn foo() { ... }

struct foo;
impl SomeTrait for foo { ... }
```

ここでは、関数 `foo` と同じ名前の新しい型 `foo` を導入しています。Rust の名前空間の規則により、両者は共存でき、親と一緒に自動的に再エクスポートされます。また、[Rust の大文字小文字の規則 (C-CASE)](https://rust-lang.github.io/api-guidelines/naming.html#casing-conforms-to-rfc-430-c-case) により、ユーザ定義の型とぶつかる可能性はとても低くなります。それでも、_公開_ 型としては見栄えが良くないので、主にルートクレートの中で、リクエストハンドラや FFI 関数を定義するのに使われます。

> ### <tip></tip> 名前空間 != モジュール {#namespaces--modules}
>
> Rust の名前空間は、ほかの言語の名前空間とは関係ありません。C# の名前空間は、Rust ではおおよそモジュールに当たります。Rust の名前空間は名前（例: `fn foo`、`struct Bar {}`、`moo!`）が持つ難解な性質で、モジュール内でその名前がどの「名前の入れ物」に属するかを決めます。
{% endraw %}
