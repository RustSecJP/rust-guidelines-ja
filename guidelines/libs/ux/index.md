{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# ライブラリ / UX のガイドライン {#libraries--ux-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 抽象化を目に見える形でネストさせない (M-SIMPLE-ABSTRACTIONS) { #M-SIMPLE-ABSTRACTIONS }

<why>低い認知負荷と、追加の準備なしで良い UX。</why>

公開型や主要な API の表面を設計するときは、ネストした型や複雑にパラメータ化された型をユーザに見せないようにしてください。

型パラメータは強力ですが、認知負荷を生みます。関係するトレイトがクレート固有のものならなおさらです。型パラメータは、
それらの型をフィールドに持つユーザのコードに伝染し、それ自体が複雑なトレイト階層を伴うことも多く、
わかりにくいエラーメッセージの原因にもなりえます。

`Foo` を書くユーザの立場から見てみましょう。ここで、他の構造体はあなたのクレートのものとします。

```rust,ignore
struct Foo {
    service: Service // 非常に良い
    service: Service<Backend> // 許容できる
    service: Service<Backend<Store>> // 悪い

    list: List<Rc<u32>> // 非常に良い。`List<T>` は単純なコンテナで、
                        // 他の型はユーザが用意したもの。

    matrix: Matrix4x4 // 非常に良い
    matrix: Matrix4x4<f32> // まだ問題ない
    matrix: Matrix<f32, Const<4>, Const<4>, ArrayStorage<f32, 4, 4>> // ?!?
}
```

_サービスのような_ 型 (つまり、主にスレッドやアプリケーションごとに一度だけインスタンス化され、依存としてよく渡される型) では、
_目に見える_ 型パラメータは避けてください。特に、ネストされる側の型がサービスと同じクレートから来ている場合はそうです。

コンテナ、スマートポインタなどのデータ構造は、当然ながら型パラメータを公開しなければなりません (上の `List<T>` など)。その場合でも、
パラメータの数やネストを抑えるよう注意してください。

型パラメータのネストを避けるべきかどうかを判断するには、次の要素を考えてください。

- その型をユーザが **名前で書く** か?
  - サービスレベルの型は、常に名前で書かれると想定される (例: `Library<T>`)
  - ユーティリティ型 (たとえば [`std::iter`](https://doc.rust-lang.org/stable/std/iter/index.html) にある `Chain`、`Cloned`、`Cycle` などの多くの型) は、
    名前で書かれることは想定されない
- その型は主にユーザのものでない型と組み合わされるか?
- 使われている型パラメータは複雑な境界を持つか?
- 使われている型パラメータは、他の型や関数の型推論に影響するか?

これらの要素が多く当てはまるほど、認知負荷は大きくなります。

目安として、主要なサービス API の型は _自分から_ ネストするべきではなく、ネストするとしても 1 段までにしてください。言い換えると、
これらの API は、ユーザに `Foo<Bar<FooBar>>` を扱うことを求めるべきではありません。ただし、`Foo<T>` のユーザが自分の `A<B<C>>` を `T` として
持ち込みたいなら、それは自由にできるようにしておくべきです。

> ### <tip></tip> より良い UX のための型の魔法? {#type-magic-for-better-ux}
>
> 上のガイドラインは、_普段の_ 開発作業で作るような「ありふれた」型を念頭に書かれています。その意図は、
> ユーザがあなたのコードを使うときに感じる摩擦を減らすことです。
>
> しかし、API のパターンやエコシステム全体を設計する場合には、全体として認知的な摩擦を _下げる_ ために、手の込んだ型の魔法を
> 導入する正当な理由があるかもしれません。[Bevy の ECS](https://docs.rs/bevy_ecs/latest/bevy_ecs/) や
> [Axum のリクエストハンドラ](https://docs.rs/axum/latest/axum/handler/trait.Handler.html) が思い浮かびます。
>
> ただし、それが見合うだけの効果を生むハードルは高いです。ジェネリクスの独創的な使い方が役に立つかどうか少しでも疑わしいなら、
> ユーザにとってはそれがないほうがよいかもしれません。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## API ではスマートポインタやラッパを避ける (M-AVOID-WRAPPERS) { #M-AVOID-WRAPPERS }

<why>低い認知負荷と、エルゴノミックな API。</why>

[M-ABSTRACTIONS-DONT-NEST] の特殊なケースとして、`Rc<T>`、`Arc<T>`、`Box<T>`、`RefCell<T>` のようなジェネリックなラッパや
スマートポインタは、公開 API では避けてください。

ユーザから見ると、これらはほとんどが実装の詳細であり、ユーザが解決しなければならない伝染性の複雑さを持ち込みます。実際、
必要なラッパの種類について複数のクレートの意見が食い違うと、解決できなくなることさえあります。

内部でラッパが必要な場合は、`&T`、`&mut T`、`T` のような単純な型を直接使うきれいな API の背後に隠してください。比べてみましょう。

```rust,ignore
// 良い例: 単純な API
pub fn process_data(data: &Data) -> State { ... }
pub fn store_config(config: Config) -> Result<(), Error> { ... }

// 悪い例: 実装の詳細を公開している
pub fn process_shared(data: Arc<Mutex<Shared>>) -> Box<Processed> { ... }
pub fn initialize(config: Rc<RefCell<Config>>) -> Arc<Server> { ... }
```

API でスマートポインタを使ってもよいのは、次の場合です。

- スマートポインタが API の目的の根幹である (例: 新しいコンテナライブラリ)

- ベンチマークに基づき、スマートポインタによってパフォーマンスが大きく向上し、複雑さに見合う

[M-ABSTRACTIONS-DONT-NEST]: ./#M-ABSTRACTIONS-DONT-NEST
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ジェネリクスより型を、dyn トレイトよりジェネリクスを使う (M-DI-HIERARCHY) { #M-DI-HIERARCHY }

<why>合成可能なパターンと、設計の固定化からの自由。</why>

非同期の依存を受け取るときは、ジェネリクスよりも具体的な型を、`dyn Trait` よりもジェネリクスを使ってください。

インタフェースに大きく頼る C# のような言語からコードを移植するとき、このパターンからうっかり外れてしまいがちです。
`Database` というサービスを C# から Rust に移植するとして、元の `IDatabase` インタフェースにならって、素朴に次のように訳したとします。

```rust,ignore
trait Database {
    async fn update_config(&self, file: PathBuf);
    async fn store_object(&self, id: Id, obj: Object);
    async fn load_object(&self, id: Id) -> Object;
}

impl Database for MyDatabase { ... }

// このように使うことを意図している:
async fn start_service(b: Rc<dyn Database>) { ... }
```

この方法は慣用的に感じられないだけでなく、オブジェクト安全性と相容れない他の Rust の構文を使えなくし、
非同期コードで問題を起こすことがあり、ラッパを公開してしまいます ([M-AVOID-WRAPPERS] 参照)。

代わりに、複数の実装が必要な場合は、次の _設計エスカレーションのはしご_ に従ってください。

もう 1 つの実装がテスト用の _sans-io_ な実装を提供するためだけのものなら、代わりに [M-MOCKABLE-SYSCALLS] に従って、
型を列挙型として実装してください。

ユーザが独自の実装を提供することが想定されるなら、1 つ以上のトレイトを導入し、自分の型に対しては固有の関数の _上に_
それらを実装してください。各トレイトは比較的狭くしてください。例: `StoreObject`、`LoadObject`。最終的に 1 つの
トレイトが必要になったら、サブトレイトにしてください。例: `trait DataAccess: StoreObject + LoadObject {}`。

これらのトレイトを扱うコードは、その使い方が大きなネストにつながらないかぎり、理想的にはジェネリックな型パラメータとして受け取ってください
([M-ABSTRACTIONS-DONT-NEST] 参照)。

```rust,ignore
// 良い例。ジェネリックが伝染する影響を持たず、最も具体的なトレイトだけを使う
async fn read_database(x: impl LoadObject) { ... }

// 許容できる。ただし、さらにネストしてこれが過剰になる場合は別。
struct MyService<T: DataAccess> {
    db: T,
}
```

ジェネリクスがネストの問題になったら、`dyn Trait` を検討してもかまいません。その場合でも、ラップしていることが見えるのは避け、独自のラッパを使ってください。

```rust
# use std::sync::Arc;
# trait DataAccess {
#     fn foo(&self);
# }
// こうしておけば、後で `DynamicDataAccess` を拡張したり変更したりできる。
// 必要なら `DynamicDataAccess` に対して `DataAccess` を実装し、普通の
// ジェネリックな関数で使うこともできる。
struct DynamicDataAccess(Arc<dyn DataAccess>);

impl DynamicDataAccess {
    fn new<T: DataAccess + 'static>(db: T) -> Self {
        Self(Arc::new(db))
    }
}

struct MyService {
    db: DynamicDataAccess,
}
```

このジェネリックなラッパは、[M-MOCKABLE-SYSCALLS] の列挙型による方法と組み合わせることもできます。

```rust,ignore
enum DataAccess {
    MyDatabase(MyDatabase),
    Mock(mock::MockCtrl),
    Dynamic(DynamicDataAccess)
}

async fn read_database(x: &DataAccess) { ... }
```

[M-AVOID-WRAPPERS]: ./#M-AVOID-WRAPPERS
[M-MOCKABLE-SYSCALLS]: ../resilience/#M-MOCKABLE-SYSCALLS
[M-ABSTRACTIONS-DONT-NEST]: ./#M-ABSTRACTIONS-DONT-NEST
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## エラーは標準的な形の構造体にする (M-ERRORS-CANONICAL-STRUCTS) { #M-ERRORS-CANONICAL-STRUCTS }

<why>統一されたエラー型と、一貫したエラーハンドリング。</why>

エラーは状況に応じた `struct` にし、[`Backtrace`](https://doc.rust-lang.org/stable/std/backtrace/struct.Backtrace.html)、
上流のエラー原因 (あれば)、ヘルパメソッドを持たせてください。

単純なクレートは通常 `Error` という 1 つのエラー型を公開し、複雑なクレートは複数の型を公開することもあります。たとえば
`AccessError` と `ConfigurationError` です。エラー型は、呼び出し側がエラーに対処できるよう、追加の情報を得るためのヘルパメソッドを提供してください。

単純なエラーは次のようになります。

```rust
# use std::backtrace::Backtrace;
# use std::fmt::Display;
# use std::fmt::Formatter;
pub struct ConfigurationError {
    backtrace: Backtrace,
}

impl ConfigurationError {
    pub(crate) fn new() -> Self {
        Self { backtrace: Backtrace::capture() }
    }
}

// Debug と Display を実装する
```

適切な場合は、エラー型で文脈に応じたエラー情報を提供してください。たとえば次のとおりです。

```rust,ignore
# use std::backtrace::Backtrace;
# #[derive(Debug)]
# pub struct ConfigurationError {
#    backtrace: Backtrace,
# }
impl ConfigurationError {
    pub fn config_file(&self) -> &Path { }
}
```

API がさまざまな操作を行う場合や、いろいろな上流のライブラリに依存する場合は、`ErrorKind` を保持してください。
ただし、エラーの種類 (より一般には列挙型ベースのエラー) を、エラーに重なりがないのに別々の公開エラー型を作るのを避ける目的で使ってはいけません。

```rust, ignore
// こちらを選ぶ
fn download_iso() -> Result<(), DownloadError> {}
fn start_vm() -> Result<(), VmError> {}

// あちらよりも
fn download_iso() -> Result<(), GlobalEverythingErrorEnum> {}
fn start_vm() -> Result<(), GlobalEverythingErrorEnum> {}

// ただし、すべての関数に新しいエラー型が必要なわけではない。エラーは
// 再利用できる程度に汎用的であるべき。
fn parse_json() -> Result<(), ParseError> {}
fn parse_toml() -> Result<(), ParseError> {}
```

内部に `ErrorKind` を使う場合でも、将来への備えとして、その列挙型を直接公開しないでください。
公開すると、内部的で対処しようのないものも含めて、_すべての_ 失敗の種類を呼び出し側にさらしてしまいます。
代わりに、次のように `is_xxx()` メソッドをいくつか公開してください。

```rust
# use std::backtrace::Backtrace;
# use std::fmt::Display;
# use std::fmt::Formatter;
#[derive(Debug)]
pub(crate) enum ErrorKind {
    Io(std::io::Error),
    Protocol
}

#[derive(Debug)]
pub struct HttpError {
    kind: ErrorKind,
    backtrace: Backtrace,
}

impl HttpError {
    pub fn is_io(&self) -> bool { matches!(self.kind, ErrorKind::Io(_)) }
    pub fn is_protocol(&self) -> bool { matches!(self.kind, ErrorKind::Protocol) }
}
```

上流のエラーのほとんどはバックトレースを提供しません。`Error` のインスタンスを生成するときに、バックトレースをキャプチャしてください。
いくつかある `Error::new()` のどれかで行うか、`From<UpstreamError> for Error {}` を実装するときに行います。

エラー構造体は、次のように表示する `Display` を正しく実装しなければなりません。

```rust,ignore
impl Display for MyError {
    // 何が起きたかを要約する文を表示する。
    // `self.backtrace` を表示する。
    // 持っている上流の「原因」の追加情報があれば表示する。
#   fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
#       todo!()
#   }
}
```

エラーは `std::error::Error` も実装しなければなりません。

```rust,ignore
impl std::error::Error for MyError { }
```

最後に、クレートから大量のエラーを出すなら、エラーの生成を簡単にする非公開の `bail!()` ヘルパマクロを作ることを検討してください。

> ### <tip></tip> バックトレースが得られるとき {#when-you-get-backtraces}
>
> 複雑なコードや非同期コードでは、エラーが表に出るまでにコールスタックを遠くまで _旅する_ ことがあるので、バックトレースは非常に貴重なデバッグツールです。
>
> とはいえ、バックトレースは _開発_ ツールであって _実行時_ の診断ではありません。デフォルトでは `Backtrace::capture()` はバックトレースをキャプチャ
> **しません**。オーバーヘッドが大きいためです (例: 筆者の PC ではキャプチャ 1 回あたり 4μs)。
>
> 代わりに、Rust は `RUST_BACKTRACE` などの[一連の環境変数](https://doc.rust-lang.org/stable/std/backtrace/index.html#environment-variables)を評価し、
> 明示的に求められたときにだけコールフレームをたどります。そうでなければ空のトレースをキャプチャし、そのコストはほんの数 CPU 命令だけです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 標準的なエラー変換には `map_err` ではなく `From` を使う (M-FROM-ERROR) { #M-FROM-ERROR }

<why>慣用的なエラーハンドリング。</why>

`Error` 型を自分で持っているなら、コードのあちこちで `.map_error()` によって変換するのではなく、`impl From<Other> for Error {}` を実装してください。`.map_error()` を呼ぶのが適切なのは、外部のエラー型を扱う場合か、文脈の情報を残す必要がある場合だけです。

```rust,ignore
// 悪い例。同じ変換を呼び出し箇所ごとに繰り返し、正常系の流れがわかりにくくなる。
fn load() -> Result<Config, MyError> {
    let bytes = read("config.toml").map_err(|e| MyError::Io(e))?;
    let text = str::from_utf8(&bytes).map_err(|e| MyError::Utf8(e))?;
    let cfg = toml::from_str(text).map_err(|e| MyError::Parse(e))?;
    Ok(cfg)
}

// 良い例。変換を一度だけ定義し、`?` に適用させる。
impl From<std::io::Error> for MyError { ... }
impl From<std::str::Utf8Error> for MyError { ... }
impl From<toml::de::Error> for MyError { ... }

fn load() -> Result<Config, MyError> {
    let bytes = read("config.toml")?;
    let text = str::from_utf8(&bytes)?;
    let cfg = toml::from_str(text)?;
    Ok(cfg)
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 複雑な型の構築にはビルダを用意する (M-INIT-BUILDER) { #M-INIT-BUILDER }

<why>複雑な場面でも将来にわたって使える型の構築。</why>

初期化の組み合わせが任意に 4 通り以上ありうる型は、ビルダを提供してください。言い換えると、省略可能な初期化パラメータが
2 つまでの型なら、固有メソッドで構築してかまいません。

```rust
# struct A;
# struct B;
struct Foo;

// 省略可能な構築パラメータは 2 つなので、固有メソッドで問題ない。
impl Foo {
    pub fn new() -> Self { Self }
    pub fn with_a(a: A) -> Self { Self }
    pub fn with_b(b: B) -> Self { Self }
    pub fn with_a_b(a: A, b: B) -> Self { Self }
}
```

それを超える場合、型はビルダを提供してください。

```rust, ignore
# struct A;
# struct B;
# struct C;
# struct Foo;
# struct FooBuilder;
impl Foo {
    pub fn new() -> Self { ... }
    pub fn builder() -> FooBuilder { ... }
}

impl FooBuilder {
    pub fn a(mut self, a: A) -> Self { ... }
    pub fn b(mut self, b: B) -> Self { ... }
    pub fn c(mut self, c: C) -> Self { ... }
    pub fn build(self) -> Foo { ... }
}

```

`Foo` を構築するビルダの正しい名前は `FooBuilder` です。そのメソッドはチェーンできなければならず、最後のメソッドは
`.build()` という名前にします。構築される構造体には `Foo::builder()` というショートカットがなければならず、一方でビルダ自体には公開の
`FooBuilder::new()` を持たせる _べきではありません_。値 `x` を設定するビルダのメソッドは、`set_x()` などではなく `x()` という名前にします。

### ビルダと必須パラメータ {#builders-and-required-parameters}

必須パラメータは、セッタメソッドではなく、ビルダを生成するときに渡してください。必須パラメータが複数あるビルダでは、
それらをパラメータ用の構造体にまとめ、柔軟性のために `deps: impl Into<Deps>` パターンを使ってください。

> **注:** ビルダに必須パラメータがない場合や、単純なパラメータが 1 つだけの場合は、専用の deps 構造体は必須ではありません。ただし、
> 後方互換性と API の発展のためには、単純な場合でも専用の deps 構造体を使うほうが好ましいです。既存のコードを壊さずに
> 新しい必須パラメータを追加しやすくなるからです。

```rust, ignore
#[derive(Debug, Clone)]
pub struct FooDeps {
    pub logger: Logger,
    pub config: Config,
}

impl From<(Logger, Config)> for FooDeps { ... }
impl From<Logger> for FooDeps { ... } // デフォルトの Config インスタンスを使える場合

impl Foo {
    pub fn builder(deps: impl Into<FooDeps>) -> FooBuilder { ... }
}
```

このパターンにより、次のように便利に使えます。

- `Foo::builder(logger)` - logger だけが必要なとき
- `Foo::builder((logger, config))` - 両方のパラメータが必要なとき
- `Foo::builder(FooDeps { logger, config })` - 構造体を明示的に構築する

あるいは、[`fundle`](https://docs.rs/fundle) を使って `FooDeps` の作成を簡単にすることもできます。

```rust, ignore
#[derive(Debug, Clone)]
#[fundle::deps]
pub struct FooDeps {
    pub logger: Logger,
    pub config: Config,
}
```

このパターンで「依存性注入」が可能になります。詳しくは[こちらのドキュメント](https://docs.rs/fundle/latest/fundle/attr.deps.html)を参照してください。

### ランタイム固有のビルダ {#runtime-specific-builders}

ランタイム固有の型や、ランタイム固有の設定を必要とする型には、適切なランタイムのパラメータを受け取る、専用のビルダ生成メソッドを用意してください。

```rust, ignore
#[cfg(feature="smol")]
#[derive(Debug, Clone)]
pub struct SmolDeps {
    pub clock: Clock,
    pub io_context: Context,
}

#[cfg(feature="tokio")]
#[derive(Debug, Clone)]
pub struct TokioDeps {
    pub clock: Clock,
}

impl Foo {
    #[cfg(feature="smol")]
    pub fn builder_smol(deps: impl Into<SmolDeps>) -> FooBuilder { ... }

    #[cfg(feature="tokio")]
    pub fn builder_tokio(deps: impl Into<TokioDeps>) -> FooBuilder { ... }
}
```

この方法により、コンパイル時に型安全性が保証され、ランタイムへの依存が API の表面で明示されます。こうしてできる
ビルダのメソッドは `builder_{runtime}(deps)` というパターンに従います。`{runtime}` は、特定のランタイムや実行環境を表します。

### 参考資料 {#further-reading}

- [Builder pattern in Rust: self vs. &mut self, and method vs. associated function](https://users.rust-lang.org/t/builder-pattern-in-rust-self-vs-mut-self-and-method-vs-associated-function/72892)
- [fundle](https://docs.rs/fundle)
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 複雑な型の初期化は段階的に組み立てる (M-INIT-CASCADED) { #M-INIT-CASCADED }

<why>パラメータの取り違えが起きない構築。</why>

パラメータが 4 つ以上必要な型は、ヘルパ型を使って初期化を段階的に組み立ててください。

```rust, ignore
# struct Deposit;
impl Deposit {
    // パラメータを取り違えやすく、シグネチャも全般に扱いにくい。
    pub fn new(bank_name: &str, customer_name: &str, currency_name: &str, currency_amount: u64) -> Self { }
}
```

長いパラメータリストを用意するのではなく、パラメータを意味ごとにまとめてください。このガイドラインを適用するときは、
[C-NEWTYPE] が当てはまるかどうかも確認してください。

```rust, ignore
# struct Deposit;
# struct Account;
# struct Currency
impl Deposit {
    // より良い。シグネチャがすっきりしている
    pub fn new(account: Account, amount: Currency) -> Self { }
}

impl Account {
    pub fn new_ok(bank: &str, customer: &str) -> Self { }
    pub fn new_even_better(bank: Bank, customer: Customer) -> Self { }
}
```

[C-NEWTYPE]: https://rust-lang.github.io/api-guidelines/type-safety.html#c-newtype
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## サービスは Clone を実装する (M-SERVICES-CLONE) { #M-SERVICES-CLONE }

<why>共通サービスの合成可能な共有。</why>

重量級の _サービス_ 型や「スレッドシングルトン」は、共有所有権の `Clone` セマンティクスを実装してください。`Application::init` から使われると想定するどの型も、これに含まれます。

ユーザは基本的に、スレッドごとにリソースハンドラのインスタンスを 1 つ生成し、同じスレッドの他のハンドラでそれを再利用できるべきです。

```rust,ignore
impl ThreadLocal for MyThreadState {
    fn init(...) -> Self {

        // 多くのものから使われうる共通のサービスインスタンスを生成する。
        let common = ServiceCommon::new();

        // ユーザはここで `common` を何度でも自由に渡せる
        let service_1 = ServiceA::new(&common);
        let service_2 = ServiceA::new(&common);

        Self { ... }
    }
}
```

サービスは、`ServiceCommon` が共有所有権のスマートポインタであるかのように、依存をクローンして新しい _ハンドル_ を保持するだけです。

```rust,ignore
impl ServiceA {
    pub fn new(common: &ServiceCommon) -> Self {
        // `new` の中で `common` にアクセスするだけなら、保持する
        // 必要はない。そうでなければ、クローンして `Self` に保持する。
        let common = common.clone();
    }
}
```

内部では、この `Clone` はサービス全体の重いコピーを作る **べきではありません**。代わりに `Arc<Inner>` パターンに従ってください。

```rust, ignore
// 中心となるロジックとデータを持つ、実際のサービス。
struct ServiceCommonInner {}

#[derive(Clone)]
pub ServiceCommon {
    inner: Arc<ServiceCommonInner>
}

impl ServiceCommon {
    pub fn new() {
        Self { inner: Arc::new(ServiceCommonInner::new()) }
    }

    // メソッドを転送する ...
    pub fn foo(&self) { self.inner.foo() }
    pub fn bar(&self) { self.inner.bar() }
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 重要な機能は固有実装にする (M-ESSENTIAL-FN-INHERENT) { #M-ESSENTIAL-FN-INHERENT }

<why>見つけやすい重要な機能。</why>

型は中心となる機能を固有実装として実装してください。トレイトの実装は固有の関数に転送するものとし、固有の関数の代わりにはしないでください。次のようにするのではなく

```rust
# trait Download {
#     fn download_file(&self, url: impl AsRef<str>);
# }
struct HttpClient {}

// 重要な機能をトレイトに押し込めると、ユーザはこの型を実際に使うために
// 他にどのトレイトを `use` すればよいのかを
// 調べなければならなくなる。
impl Download for HttpClient {
    fn download_file(&self, url: impl AsRef<str>) {
        // ... ファイルをダウンロードするロジック
    }
}
```

こうしてください。

```rust
# trait Download {
#     fn download_file(&self, url: impl AsRef<str>);
# }
struct HttpClient {}

impl HttpClient {
    fn download_file(&self, url: impl AsRef<str>) {
        // ... ファイルをダウンロードするロジック
    }
}

// 呼び出しを固有実装に転送する。`HttpClient` を使える
impl Download for HttpClient {
    fn download_file(&self, url: impl AsRef<str>) {
        Self::download_file(self, url)
    }
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## モジュールは大きさと範囲のバランスをとる (M-BALANCED-MODULES) { #M-BALANCED-MODULES }

<why>見つけやすい機能と、わかりやすい API の使い方。</why>

モジュールの設計は、メニュー設計で確立された UX の慣行におおむね従ってください。最も重要な要素のうち _妥当な_ 数をクレートのルートに置き、残りの機能はわかりやすくまとめて下位のモジュールに分けます。

このルールに反する例として最もよく見かけるのは次の 2 つです。はっきりした順序のない何十もの要素を含む平らなモジュールルートと、クレートのルートに要素を置かずにサブモジュールを使いすぎることです。そうするのが理にかなうクレートもありますが (例: 何百もの C の要素を定義する、自動生成された `-sys` クレートや、`std` や `tokio` のようなアンブレラクレート)、大半のライブラリクレートはそれには当たりません。

モジュールの構成を設計するときは、次の要素を考慮してください。

- クレートを使うためにユーザが見つけなければならない重要な要素は、ルートに置く。たとえば `foo_client` クレートなら、主となる `Client` 構造体はおそらくルートに置くべきである。
- その他の要素は、ユースケースごとに意味でまとめる。`traits` や `errors` という名前のモジュールは誰の役にも立たないが、`account`、`network`、`status` なら役に立つ。
- また、モジュールは、それぞれのサブシステムをさらに説明するモジュールレベルのドキュメントを書くのにうってつけの場所でもあることを考慮する。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## プレリュードを定義しない (M-NO-PRELUDE) { #M-NO-PRELUDE }

<why>きれいな名前空間と、下流での確実なビルド。</why>

クレートは `prelude` や、`use foo::*` としてインポートされることを意図した名前空間を定義してはなりません。

Rust の標準ライブラリはエディションの要素を定義するために[プレリュード](https://doc.rust-lang.org/std/prelude/index.html)をうまく使っていますが、クレートのプレリュードは益よりも害のほうが大きいです。今日の IDE のサポートがあれば必要ありませんし、異なるクレートの複数のプレリュードを使うと衝突が起きる可能性があります。

```rust,ignore
use foo::prelude::*;
use bar::prelude::*;
use baz::prelude::*;

_ = Client::new();

// error[E0659]: `Client` is ambiguous
//   --> src/lib.rs:17:13
//    |
// 17 |     _ = Client; 
//    |         ^^^^^^ ambiguous name
//    |
//    = note: ambiguous because of multiple glob imports of a name in the same module
```

特に、プレリュードはまずいモジュール設計を解決しません。プレリュードがあればクレートが使いやすく、わかりやすくなりそうに見えるなら、それはほぼ常に、既存のモジュール構成を作り直す必要があるというサインです。[M-BALANCED-MODULES](./#M-BALANCED-MODULES) を参照してください。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## パラメータの順序を一貫させる (M-PARAMETER-CONSISTENCY) { #M-PARAMETER-CONSISTENCY }

<why>開発時の摩擦の少なさ。</why>

概念的に同じパラメータが複数の関数に (クレート内でも、同じエコシステムのクレート間でも) 現れる場合、どこでも同じ順序にしてください。

- 重要なパラメータや、その呼び出しに固有のパラメータは、一般に先頭に置く
- どこにでも出てくるパラメータは、どちらかといえば最後に置く (例: `&logger`)
- クロージャは常に最後に置く (関数は 2 つ以上のクロージャを受け取るべきではない)

```rust,ignore
// 悪い例。`user_id` と `tenant_id` の順序が関数ごとに入れ替わり、
// logger も先頭に来たり最後に来たりする。
fn create_user(logger: &Logger, user_id: UserId, tenant_id: TenantId) -> Result<()> { ... }
fn delete_user(tenant_id: TenantId, user_id: UserId, logger: &Logger) -> Result<()> { ... }
fn rename_user(user_id: UserId, new_name: &str, tenant_id: TenantId, logger: &Logger) -> Result<()> { ... }

// 良い例。呼び出しに固有のパラメータを一貫した順序で先頭に置き、
// どこにでも出てくる `logger` は常に最後。
fn create_user(tenant_id: TenantId, user_id: UserId, logger: &Logger) -> Result<()> { ... }
fn delete_user(tenant_id: TenantId, user_id: UserId, logger: &Logger) -> Result<()> { ... }
fn rename_user(tenant_id: TenantId, user_id: UserId, new_name: &str, logger: &Logger) -> Result<()> { ... }
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## コレクションは適切なイテレータ関連のトレイトを実装する (M-COLLECTION-TRAITS) { #M-COLLECTION-TRAITS }

<why>合成可能なコレクション。</why>

独自のコレクションは、標準ライブラリが提供するイテレータ関連のトレイトを実装してください。

第三者に使ってもらうために新しいコレクション型 `Collection<T>` を定義するときは、次のトレイトと型も実装してください。詳しくは[こちら](https://cheats.rs/#iterators)を参照してください。

- 構造体 `IntoIter<T>`、`Iter<T>`、`IterMut<T>`
- それらすべてに対する `impl Iterator`
- メソッド `c.iter()` と `c.iter_mut()`
- `Collection<T>`、`&Collection<T>`、`&mut Collection<T>` に対する `impl IntoIterator`
- `Collection<T>` に対する `impl FromIterator`
- `Collection<T>` に対する `Extend`
- 必要に応じて `DoubleEndedIterator`、`ExactSizeIterator` など

加えて、すべてのイテレータで `size_hint()` を実装し、正確な値を返すようにしてください。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 関数は Future を返すよりも `async` にする (M-ASYNC-FN) { #M-ASYNC-FN }

<why>より単純なコードと、より理解しやすい API。</why>

どちらも使える場合は、関数を `fn foo() -> impl Future` ではなく `async fn foo()` として宣言してください。

`async` を付けた関数のほうが慣用的で読みやすいです。明示的に `Future` を返すシグネチャは、必要な場合にだけ使ってください。たとえばトレイトの中や、_重くて負荷の高い_ 非同期関数の場合です。[M-ASYNC-STACK-SIZE](../../performance/#M-ASYNC-STACK-SIZE) を参照してください。

```rust,ignore
impl Foo {
    // 悪い例。シグネチャがごちゃごちゃしていて、本体に余計な `async` ブロックが要る
    fn foo() -> impl Future<Output = Result<T, E>> { async { Ok(t) } }

    // 良い例。メソッドも実装も普通に読める
    async fn foo() -> Result<T, E> { Ok(t) }
}
```
{% endraw %}
