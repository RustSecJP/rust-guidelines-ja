{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# 共通ガイドライン {#universal-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 上流のガイドラインに従う (M-UPSTREAM-GUIDELINES) { #M-UPSTREAM-GUIDELINES }

<why>コミュニティの教訓を反映し、ユーザやコントリビュータを驚かせないコードベース。</why>

この本のガイドラインは、既存の Rust のガイドラインを補うものです。特に次のものです。

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html)
- [Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/)
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns//intro.html)
- [Rust Reference - Undefined Behavior](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)

これらにも目を通し、この本の項目に加えて適用することをおすすめします。次の項目は忘れられがちなので、特に注意してください。

- [ ] [C-CONV](https://rust-lang.github.io/api-guidelines/naming.html#ad-hoc-conversions-follow-as_-to_-into_-conventions-c-conv) - その場限りの変換は
  `as_`、`to_`、`into_` の慣習に従う
- [ ] [C-GETTER](https://rust-lang.github.io/api-guidelines/naming.html#getter-names-follow-rust-convention-c-getter) - ゲッタの名前は Rust の慣習に従う
- [ ] [C-COMMON-TRAITS](https://rust-lang.github.io/api-guidelines/interoperability.html#c-common-traits) - 型は一般的なトレイトを積極的に実装する
  - `Copy`, `Clone`, `Eq`, `PartialEq`, `Ord`, `PartialOrd`, `Hash`, `Default`, `Debug`
  - 型を表示したい場合は `Display`
- [ ] [C-CTOR](https://rust-lang.github.io/api-guidelines/predictability.html?highlight=new#constructors-are-static-inherent-methods-c-ctor) -
  コンストラクタは静的な固有メソッドにする
  - 特に、`Foo::default()` があっても `Foo::new()` を用意する
- [ ] [C-FEATURE](https://rust-lang.github.io/api-guidelines/naming.html#feature-names-are-free-of-placeholder-words-c-feature) - フィーチャ名には
  意味のない語を含めない
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 静的検証を使う (M-STATIC-VERIFICATION) { #M-STATIC-VERIFICATION }

<why>一貫性と、よくある問題がないこと。</why>

コードの品質を保つため、プロジェクトでは次の静的検証ツールを使ってください。これらのツールは、普段の作業中に開発者のマシンで
実行するよう設定できます。また、チェックインゲートの一部としても使ってください。

* [コンパイラのリント](https://doc.rust-lang.org/rustc/lints/index.html) は、バグを防ぎコードの品質を高める多くのリントを提供します。
* [clippy のリント](https://doc.rust-lang.org/clippy/) には、バグを防ぎコードの品質を高める何百ものリントがあります。
* [rustfmt](https://github.com/rust-lang/rustfmt) は、ソースのフォーマットを一貫させます。
* [cargo-audit](https://crates.io/crates/cargo-audit) は、依存クレートにセキュリティ脆弱性がないか検査します。
* [cargo-hack](https://crates.io/crates/cargo-hack) は、クレートのフィーチャのすべての組み合わせが正しく動くことを確かめます。
* [cargo-udeps](https://crates.io/crates/cargo-udeps) は、Cargo.toml ファイル内の使われていない依存を検出します。
* [miri](https://github.com/rust-lang/miri) は、`unsafe` なコードの正しさを検証します。

### コンパイラのリント {#compiler-lints}

Rust のコンパイラは、一般にきわめて質の高い診断を出します。デフォルトの診断に加えて、プロジェクトでは
次のコンパイラのリントを明示的に有効にしてください。

```toml
[lints.rust]
ambiguous_negative_literals = "warn"
missing_debug_implementations = "warn"
redundant_imports = "warn"
redundant_lifetimes = "warn"
trivial_numeric_casts = "warn"
unsafe_op_in_unsafe_fn = "warn"
unused_lifetimes = "warn"
```

### Clippy のリント {#clippy-lints}

clippy については、主要なリントカテゴリをすべて有効にし、さらに `restriction` リントグループからいくつかのリントを有効にしてください。
望ましくないリント (例: 数値のキャスト) は、個別に判断して無効に戻せます。

```toml
[lints.clippy]
cargo = { level = "warn", priority = -1 }
complexity = { level = "warn", priority = -1 }
correctness = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
perf = { level = "warn", priority = -1 }
style = { level = "warn", priority = -1 }
suspicious = { level = "warn", priority = -1 }
# nursery = { level = "warn", priority = -1 }  # 任意。誤検知が増える可能性がある

# 追加のリントは `restriction` と `nursery` グループのもの。一貫性、品質、簡潔さを
# 高めるために、ソースコード中で特定の構文が使われるのを
# 防ぐ
allow_attributes_without_reason = "warn"
as_pointer_underscore = "warn"
assertions_on_result_states = "warn"
clone_on_ref_ptr = "warn"
deref_by_slicing = "warn"
disallowed_script_idents = "warn"
empty_drop = "warn"
empty_enum_variants_with_brackets = "warn"
empty_structs_with_brackets = "warn"
fn_to_numeric_cast_any = "warn"
if_then_some_else_none = "warn"
map_err_ignore = "warn"
redundant_type_annotations = "warn"
renamed_function_params = "warn"
semicolon_outside_block = "warn"
undocumented_unsafe_blocks = "warn"
unnecessary_safety_comment = "warn"
unnecessary_safety_doc = "warn"
unneeded_field_pattern = "warn"
unused_result_ok = "warn"
too_long_first_doc_paragraph = "warn"

# そうしないと構造化ログで問題が起きることがある。
literal_string_with_formatting_args = "allow"

# 独自の無効化はここで定義する
# ...
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## リントの上書きには `#[expect]` を使う (M-LINT-OVERRIDE-EXPECT) { #M-LINT-OVERRIDE-EXPECT }

<why>最新で整ったリント設定。</why>

プロジェクト全体のリントをサブモジュールや要素の中で上書きするときは、`#[allow]` ではなく `#[expect]` を使ってください。

`#[expect]` を付けたリントは、想定したウォーニングが発生しなかった場合にウォーニングを出します。そのため、不要になったリント指定がたまっていくのを防げます。
とはいえ、生成されたコードに付ける場合には `#[allow]` も役に立ちますし、マクロの中に現れることもあります。

上書きには `reason` を付けてください。

```rust,edition2021
#[expect(clippy::unused_async, reason = "API fixed, will use I/O later")]
pub async fn ping_server() {
  // 今のところはスタブ
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 公開型は Debug を実装する (M-PUBLIC-DEBUG) { #M-PUBLIC-DEBUG }

<why>機密データを漏らさない、簡単なデバッグ。</why>

クレートが公開するすべての公開型は `Debug` を実装してください。ほとんどの型は `#[derive(Debug)]` で実装できます。

```rust
#[derive(Debug)]
struct Endpoint(String);
```

機密データを保持するための型も `Debug` を実装してください。ただし、独自の実装を書きます。
その実装には単体テストを用意して、機密データが実際に漏れていないこと、将来も漏れないことを確かめなければなりません。

```rust
use std::fmt::{Debug, Formatter};

struct UserSecret(String);

impl Debug for UserSecret {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "UserSecret(...)")
    }
}

#[test]
fn test() {
    let key = "552d3454-d0d5-445d-ab9f-ef2ae3a8896a";
    let secret = UserSecret(key.to_string());
    let rendered = format!("{:?}", secret);

    assert!(rendered.contains("UserSecret"));
    assert!(!rendered.contains(key));
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 人が読むための公開型は Display を実装する (M-PUBLIC-DISPLAY) { #M-PUBLIC-DISPLAY }

<why>使いやすさ。</why>

型が上流の利用者 (開発者であれエンドユーザであれ) に読まれることを想定しているなら、その型は `Display` を実装してください。特に次のものが含まれます。

- エラー型。`std::error::Error` によって `Display` の実装が必須とされています
- 文字列のようなデータのラッパ

`Display` の実装は Rust の慣習に従ってください。これには改行やエスケープシーケンスの表示も含まれます。
[M-PUBLIC-DEBUG] で述べた機密データの扱いは、ここでも同様に当てはまります。

[M-PUBLIC-DEBUG]: ./#M-PUBLIC-DEBUG
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 迷ったらクレートを分割する (M-SMALLER-CRATES) { #M-SMALLER-CRATES }

<why>速いコンパイルと良いモジュール性。</why>

クレートは少なすぎるよりも多すぎるくらいのほうがよいです。そうすることで、コンパイル時間が劇的に改善し (特にそれらのクレートを開発している間)、
コンポーネント間の循環依存も防げます。

基本的には、サブモジュールが単独で使えるなら、その中身を別のクレートに移してください。

クレートを分割すると、一部の `pub(crate)` フィールドやメソッドにアクセスできなくなることがあります。多くの場合、これは望ましい副作用です。
ユーザに同じような使い勝手を提供できる、より柔軟な抽象化を設計するきっかけにしてください。

場合によっては、個々のクレートを 1 つの _アンブレラクレート_ にまとめ直すのが望ましいこともあります。手続き的マクロやランタイムを扱う場合などです。
技術的な理由で分割した機能 (例: `foo_proc` という手続き的マクロのクレート) は、常に再エクスポートしてください。それ以外の場合、再エクスポートは控えめに使ってください。

> ### <tip></tip> フィーチャかクレートか {#features-vs-crates}
>
> 目安として、クレートは単独でも無理なく使える要素のためのものです。フィーチャは、単独では成り立たない追加機能を有効にするためのものです。
> アンブレラクレートの場合 (下記参照) は、構成要素を有効にするためにフィーチャを使ってもかまいません (ただしその場合、その機能はすでにクレートとして切り出されています)。
>
> たとえば、次のモジュールを持つ `web` クレートを定義すると、クライアントの呼び出しだけが必要なユーザも、サーバのコードのコンパイルの負担を負うことになります。
>
> ```text
> web::server
> web::client
> web::protocols
> ```
>
> 代わりに、ユーザが必要なものを選べるよう、個別のクレートを用意してください。
>
> ```text
> web_server
> web_client
> web_protocols
> ```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 名前に曖昧語を含めない (M-WEASEL-WORDS) { #M-WEASEL-WORDS }

<why>読みやすさ。</why>

シンボル名、特に型やトレイトの名前には、意味のある情報を加えない曖昧語 (weasel word) を含めないでください。よくある例は `Service`、`Manager`、`Factory` です。

ライブラリが予約サービスを含んでいたり、予約サービスとやりとりしたりすることは十分ありえます&mdash;`booking_service` という名前の `HttpClient`
インスタンスを持つことさえあるでしょう&mdash;が、コードの中で `BookingService` という _型_ を目にすることはめったにないはずです。

多くの予約を扱う要素は、単に `Bookings` と呼べば十分です。もっと具体的なことをするなら、代わりにその性質を名前に付け加えてください。
それらの予約をどこかへ送信する? それなら `BookingDispatcher` と呼ぶほうが役に立ちます。

`Manager` も同じです。どんなコードも _何か_ を管理しているので、この呼び名が役に立つことはまずありません。ごく一部の例外を除き、
ライフサイクルの問題も、何かのマネージャの担当にすべきではありません。要素は必要に応じたやり方で生成され、その破棄は `Drop` が、
そして `Drop` だけが管理します。

ファクトリについては、少なくともその用語は避けてください。`FooFactory` という概念には使い道がありますが、Rust での標準的な名前は
`Builder` です ([M-INIT-BUILDER](../libs/ux/#M-INIT-BUILDER) 参照)。要素を繰り返し生成できるビルダも、やはりビルダです。

加えて、ファクトリ (ビルダ) を引数として受け取るのは、オブジェクト指向の概念を Rust に持ち込んだ慣用的でないやり方です。
繰り返しインスタンスを生成する必要があるなら、関数は `FooBuilder` などではなく `impl Fn() -> Foo` を受け取るべきです。
一方、単体のビルダには使い道があります。ただし主な用途は、省略可能な値をめぐるパラメータの組み合わせの複雑さを減らすことです
(これも [M-INIT-BUILDER](../libs/ux/#M-INIT-BUILDER) 参照)。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 要素の名前は短くする (M-SHORT-NAMES) { #M-SHORT-NAMES }

<why>慣用的なコード。</why>

要素の識別子は短くする、という Rust の慣習に従ってください。

- 識別子は、短い単語を 2 つより多く組み合わせない (`GlobalApplicationConfig` より `AppConfig`)
- モジュールやクレートの情報を接頭辞に埋め込まない (`foo::FooId` より `foo::Id`)。特に、直接の「親」の要素だけで十分に説明がつく場合はそうする。こうした場合、必要に応じて、ユーザがその場で修飾子を使って要素を区別することが期待される (`fn convert(foo::Id) -> bar::Id`)
- 略語を使う (`CallbackFunction` より `CallbackFn`)

その場で理にかなうなら、これらのルールはどれも破ってかまいません。ただしクレート単位で見て、そうした例外は _例外的_ なものにとどめ、十分な理由があるべきです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 関連関数よりも普通の関数を使う (M-REGULAR-FN) { #M-REGULAR-FN }

<why>読みやすさ。</why>

関連関数は主にインスタンスの生成に使い、汎用的な計算には使わないでください。

一部のオブジェクト指向言語と違い、Rust では普通の関数が第一級市民であり、それを置くためのモジュールや _クラス_ は必要ありません。
したがって、レシーバに明確に属さない機能は、型の `impl` ブロックの中に置くべきではありません。

```rust, ignore
struct Database {}

impl Database {
    // OK。関連関数がインスタンスを生成している
    fn new() -> Self {}

    // OK。`&self` をレシーバに持つ普通のメソッド
    fn query(&self) {}

    // よくない。この関数は `Database` と直接関係がないので、
    // 関連関数として `Database` の下に置くべきではない。
    fn check_parameters(p: &str) {}
}

// 普通の関数としてなら問題ない
fn check_parameters(p: &str) {}
```

普通の関数のほうが慣用的で、呼び出し側の余計なノイズも減ります。ただし、トレイトの関連関数はまったく問題なく慣用的です。

```rust
pub trait Default {
    fn default() -> Self;
}

struct Foo;

impl Default for Foo {
    fn default() -> Self { Self }
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## マジック値はドキュメント化する (M-DOCUMENTED-MAGIC) { #M-DOCUMENTED-MAGIC }

<why>保守性と安全なリファクタリング。</why>

本番コードにハードコードした _マジック_ 値には、コメントを付けなければなりません。コメントには次のことを書きます。

- その値を選んだ理由
- 値を変えたときに起きる、わかりにくい副作用
- その定数とやりとりする外部システム

インラインの値よりも、名前付きの定数を使うようにしてください。

```rust, ignore
// 悪い例: 1日待つことは割と明らかだが、その理由がわからない
wait_timeout(60 * 60 * 24).await // 最大で1日待つ

// より良い例
wait_timeout(60 * 60 * 24).await // サーバが処理を終えられるよう十分に
                                 // 大きな値。小さすぎると、正当な
                                 // リクエストをアボートしかねない。
                                 // `api.foo.com` のタイムアウトポリシーに基づく。

// 最も良い例

/// サーバを待つ時間。
///
/// サーバが処理を終えられるよう十分に
/// 大きな値。小さすぎると、正当な
/// リクエストをアボートしかねない。
/// `api.foo.com` のタイムアウトポリシーに基づく。
const UPSTREAM_SERVER_TIMEOUT: Duration = Duration::from_secs(60 * 60 * 24);
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## メッセージテンプレートを使った構造化ログを使う (M-LOG-STRUCTURED) { #M-LOG-STRUCTURED }

<why>低コストなログと強力なフィルタリング。</why>

ログには、名前付きプロパティを持つ構造化イベントと、[message templates](https://messagetemplates.org/) 仕様に従ったメッセージテンプレートを使ってください。

> **注:** 例では [`tracing`](https://docs.rs/tracing/) クレートの `event!` マクロを使っていますが、
ここでの原則は構造化ログをサポートするどのログ API (例: `log`、`slog`、独自のテレメトリシステム) にも当てはまります。

### 文字列のフォーマットを避ける {#avoid-string-formatting}

文字列のフォーマットは実行時にメモリを確保します。メッセージテンプレートなら、フォーマットを閲覧時まで遅らせられます。
閲覧時に調べやすいよう、メッセージテンプレートにはすべての名前付きプロパティを含めることをおすすめします。

```rust,ignore
// 悪い例: 文字列のフォーマットがアロケーションを引き起こす
tracing::info!("file opened: {}", path);
tracing::info!(format!("file opened: {}", path));

// 良い例: 名前付きプロパティを持つメッセージテンプレート
event!(
    name: "file.open.success",
    Level::INFO,
    file.path = path.display(),
    "file opened: {{file.path}}",
);
```

> **注**: メッセージテンプレートでは `{{property}}` 構文を使ってください。この構文は Rust のフォーマット構文をエスケープしつつ、
> リテラルのテキストをそのまま残します。文字列のフォーマットはログを閲覧するときまで遅らされます。

### イベントに名前を付ける {#name-your-events}

階層的なドット記法 `<component>.<operation>.<state>` を使います。

```rust,ignore
// 悪い例: 名前のないイベント
event!(
    Level::INFO,
    file.path = file_path,
    "file {{file.path}} processed successfully",
);

// 良い例: 名前付きイベント
event!(
    name: "file.processing.success", // イベント識別子
    Level::INFO,
    file.path = file_path,
    "file {{file.path}} processed successfully",
);
```

名前付きイベントにすると、ログエントリをまたいでグループ化やフィルタリングができます。

### OpenTelemetry のセマンティック規約に従う {#follow-opentelemetry-semantic-conventions}

必要であれば、一般的な属性には [OTel のセマンティック規約](https://opentelemetry.io/docs/specs/semconv/) を使ってください。
これにより標準化と相互運用性が得られます。

```rust,ignore
event!(
    name: "file.write.success",
    Level::INFO,
    file.path = path.display(),         // 標準の OTel 名
    file.size = bytes_written,          // 標準の OTel 名
    file.directory = dir_path,          // 標準の OTel 名
    file.extension = extension,         // 標準の OTel 名
    file.operation = "write",           // 独自の名前
    "{{file.operation}} {{file.size}} bytes to {{file.path}} in {{file.directory}} extension={{file.extension}}",
);
```

よく使う規約:

- HTTP: `http.request.method`, `http.response.status_code`, `url.scheme`, `url.path`, `server.address`
- ファイル: `file.path`, `file.directory`, `file.name`, `file.extension`, `file.size`
- データベース: `db.system.name`, `db.namespace`, `db.operation.name`, `db.query.text`
- エラー: `error.type`, `error.message`, `exception.type`, `exception.stacktrace`

### 機密データを伏せる {#redact-sensitive-data}

機密データをそのままログに出してはいけません。プライバシやセキュリティのインシデントにつながるおそれがあります。

```rust,ignore
// 悪い例: 機密データかもしれないものをログに出している
event!(
    name: "file.operation.started",
    Level::INFO,
    user.email = user.email,  // 機密データ
    file.name = "license.txt",
    "reading file {{file.name}} for user {{user.email}}",
);

// 良い例: 機密部分を伏せる
event!(
    name: "file.operation.started",
    Level::INFO,
    user.email.redacted = redact_email(user.email),
    file.name = "license.txt",
    "reading file {{file.name}} for user {{user.email.redacted}}",
);
```

機密データには、メールアドレス、ユーザの身元がわかるファイルパス、シークレットやトークンを含むファイル名、
個人を特定できる情報 (PII) を含むファイルの中身、セッション ID を含む一時ファイルのパスなどがあります。一貫した伏せ字処理には [`data_privacy`](https://crates.io/crates/data_privacy) クレートの利用を検討してください。

### 参考資料 {#further-reading}

- [Message Templates Specification](https://messagetemplates.org/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
{% endraw %}
