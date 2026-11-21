{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# ライブラリ / 堅牢性のガイドライン {#libraries--resilience-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## I/O とシステムコールはモック可能にする (M-MOCKABLE-SYSCALLS) { #M-MOCKABLE-SYSCALLS }

<why>普段は起こしにくいエッジケースをテストできること。</why>

I/O や副作用のあるシステムコールを行う、ユーザ向けの型はすべて、それらの作用をモックできるようにしてください。これにはファイルや
ネットワークへのアクセス、時計、エントロピー源やシードなどが含まれます。より一般的には、次のような操作はすべて

- 非決定的である
- 外部の状態に依存する
- ハードウェアや環境に依存する
- その他の理由で壊れやすい、または常に再現できるとはかぎらない

モックできるようにしてください。

> ### <tip></tip> アロケーションもモックする? {#mocking-allocations}
>
> カーネルのコードなどを書いているのでないかぎり、アロケーションは決定的で、ハードウェアに依存せず、実質的に失敗しないものと
> みなしてかまいません。そのため、このガイドラインの対象外です。
>
> ただし、これは無制限にメモリが使えると考えてよいという意味では _ありません_。ライブラリのメモリ計算量が _妥当_ であれば、
> 呼び出し側が渡した入力をそのまま受け取ってもかまいませんが、メモリを大量に使うライブラリや外部からの入力を扱うコードは、
> 上限付きの操作や分割処理の操作 (またはその両方) を提供してください。

このガイドラインは、ライブラリにいくつかのことを求めます。ライブラリは

- その場限りの I/O を行わない。つまり `read("foo.txt")` を呼び出さない
- モックできない I/O やシステムコールに依存しない
- I/O やシステムコールの _コア_ を自分で作らない
- `MyIoLibrary::default()` のようなコンストラクタを提供しない

代わりに、I/O やシステムコールを行うライブラリは、すでにモック可能な I/O の _コア_ を受け取るか、自分でモックの機能を提供してください。

```rust, ignore
let lib = Library::new_runtime(runtime_io); // モック可能な I/O 機能を渡す
let (lib, mock) = Library::new_mocked(); // 固有のモック機能をサポートする
```

固有のモック機能をサポートするライブラリは、次のように実装してください。

```rust, ignore
pub struct Library {
    some_core: LibraryCore // システムコールや I/O などをカプセル化する。下記参照。
}

impl Library {
    pub fn new() -> Self { ... }
    pub fn new_mocked() -> (Self, MockCtrl) { ... }
}
```

内部では、`LibraryCore` は [M-RUNTIME-ABSTRACTED] と同様の非公開の列挙型で、呼び出しを該当するシステムコールか
モックコントローラのどちらかにディスパッチします。

```rust, ignore
// 呼び出しを OS か
// モックコントローラのどちらかにディスパッチする。
enum LibraryCore {
    Native,

    #[cfg(feature = "test-util")]
    Mocked(mock::MockCtrl)
}

impl LibraryCore {
    // OS に転送する何らかの関数。
    fn random_u32(&self) {
        match self {
            Self::Native => unsafe { os_random_u32() }
            Self::Mocked(m) => m.random_u32()
        }
    }
}


#[cfg(feature = "test-util")]
mod mock {
    // M-SERVICES-CLONE のパターンに従っているので、`LibraryCore` と
    // ユーザの両方が同じ `MockCtrl` インスタンスを保持できる。
    pub struct MockCtrl {
        inner: Arc<MockCtrlInner>
    }

    // 必要なロジックをそれに合わせて実装する。たいていは下の
    // `MockCtrlInner` に転送する。
    impl MockCtrl {
        pub fn set_next_u32(&self, x: u32) { ... }
        pub fn random_u32(&self) { ... }
    }

    // 実際のロジックを持つ。例: 次に返すべき乱数。
    struct MockCtrlInner {
        next_call: u32
    }
}
```

すでに [M-RUNTIME-ABSTRACTED] パターンの上に構築されている、ランタイムを意識したライブラリは、代わりにランタイムの列挙型を拡張してください。

```rust, ignore
enum Runtime {
    #[cfg(feature="tokio")]
    Tokio(tokio::Tokio),

    #[cfg(feature="smol")]
    Smol(smol::Smol)

    #[cfg(feature="test-util")]
    Mock(mock::MockCtrl)
}
```

上で示したように、モックをサポートするほとんどのライブラリは、モックコントローラを受け取るのではなく、タプルで返してください。
最初の要素をライブラリのインスタンス、2 番目をモックコントローラにします。こうするのは、複数のインスタンスが
1 つのコントローラを共有した場合に状態が曖昧になるのを防ぐためです。

```rust, ignore
impl Library {
    pub fn new_mocked() -> (Self, MockCtrl) { ... } // 良い
    pub fn new_mocked_bad(&mut MockCtrl) -> Self { ... } // 誤用されやすい
}
```

[M-RUNTIME-ABSTRACTED]: ../ux/#M-RUNTIME-ABSTRACTED
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## テスト用ユーティリティはフィーチャで制限する (M-TEST-UTIL) { #M-TEST-UTIL }

<why>安全性検査を迂回できない本番ビルド。</why>

テスト用の機能は、フィーチャフラグの背後に隠さなければなりません。これには次のものが含まれます。

- モックの機能 ([M-MOCKABLE-SYSCALLS])
- 機密データを調べる機能
- 安全性検査の上書き
- 偽データの生成

フラグは `test-util` という名前の 1 つだけにすることをおすすめします。いずれにしても、そのフィーチャがテスト目的であることがはっきりわかるようにしなければなりません。

```rust, ignore
impl HttpClient {
    pub fn get() { ... }

    #[cfg(feature = "test-util")]
    pub fn bypass_certificate_checks() { ... }
}
```

[M-MOCKABLE-SYSCALLS]: ./#M-MOCKABLE-SYSCALLS
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 結合テストは `tests/` の下に置く (M-INTEGRATION-TESTS) { #M-INTEGRATION-TESTS }

<why>すっきりしたコードファイル。</why>

公開 API の表面だけに触れるテストは _結合テスト_ であり、`mod tests {}` ではなく `tests/` の下に置きます。

カバレッジの目標があるプロジェクトでは、`src/` のファイルに実際のビジネスロジックよりも多くのテストコードが含まれることも珍しくありません。そうなると、IDE でも PR でもコードを見て理解するのが難しくなります。同様に、テストの目的が結合テストでも単体テストでも達成できるなら、常に前者を選んでください。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 適切な型族を使う (M-STRONG-TYPES) { #M-STRONG-TYPES }

<why>正しいデータと安全性の不変条件を、正しいタイミングで。</why>

作業に合った `std` の型を使ってください。一般に、使える中で最も強い型を、API の流れのできるだけ早い段階で使ってください。よくある違反は次のとおりです。

| 使わないもの ... | 代わりに使うもの ... | 説明 |
| --- | --- | --- |
| `String`* | `PathBuf`* | OS を扱うものはすべて `Path` のような型にする |

とはいえ、Rust の `std` の一般的な慣習にも従ってください。公開 API の境界にある純粋に数値の型 (例: `window_size()`) は、
`Saturating<usize>` や `NonZero<usize>` などではなく、普通の数値であることが期待されます。

<footnotes>

<sup>*</sup> その兄弟の型も含みます。例: `&str`、`Path` など。

</footnotes>
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ニュータイプは自身の不変条件を守る (M-STRONG-TYPES-GUARD) { #M-STRONG-TYPES-GUARD }

<why>正しさの不変条件を 1 か所に集めること。</why>

不変条件 (空でない文字列、パーセンテージ、ポート番号、サニタイズ済みのパスなど) を表すために強い型やニュータイプを導入するときは、該当する場合、型そのものがその不変条件を強制しなければなりません。

構築は失敗しうるものにし、不変条件を守れないときは適切なエラーを返してください。その責任をすべての利用者に押しつけてはいけません。

```rust,ignore
// 悪い例。新しい型を作っているが何も強制していない。呼び出し側はみな
// 値が本当に有効な月かどうかを確認し直さなければならず、専用の型を
// 持つ意味がなくなる。
pub struct Month(pub u8);

impl Month {
    pub fn new(value: u8) -> Self { ... }
}


// 良い例。不変条件 (1..=12) は境界で一度だけ確認され、
// 以降の `Month` の使用はすべてそれを当てにできる。
pub struct Month(u8);

impl Month {
    pub fn from_u8(value: u8) -> Result<Self, DateError> { ... }
}
```

つまり、全域的でない (non-total) ニュータイプについては次のとおりです。

- 失敗しうるコンストラクタを少なくとも 1 つ持たなければならない (例: `fn from_foo(...) -> Result<Self, _>`)
- パニックするコンストラクタを追加で持ってもよく (例: `new`)、できれば `const` にする
- より弱い型からニュータイプへの変換は、失敗しうるものでなければならない (`TryFrom`/`FromStr`)
- 失敗しない `From` の実装を提供してはならない

> ### <tip></tip> なぜ `const` なのか? {#why-const}
>
> コンストラクタを const にすると `const {}` ブロックの中で使えるので、こうした違反がエラーとして表に出ます。これにより、
> ユーザは `let month_due = const { Month::new(14) }` のように書けて、実行時にこうしたパスに入ることを避けられます。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ビルダは最後の `.build()` で検証する (M-BUILD-RESULT) { #M-BUILD-RESULT }

<why>すっきりしたビルダのエラーハンドリング。</why>

ビルダのフィールドごとのセッタは失敗せずに入力を受け取り、最終的な検証は `.build()` で行ってください。

失敗しうるセッタはノイズを増やすうえ、相互に依存するエラー条件を防ぐこともできません。ビルダが失敗しうる場合は、代わりに `Result` を返す `.build()` を提供してください。

```rust,ignore
// 悪い例。何の価値もないエラーチェックを繰り返させる。
Foo::builder()
    .name("Foo")?
    .distance(42)?
    .build();

// 良い例。妥当性チェックを 1 か所にまとめ、プロパティ間の
// 相互チェックもできる。
Foo::builder()
    .name("Foo")
    .distance(42)
    .build()?;
```

とはいえ、該当する場合は、個々の設定値に独自の検証を持つ強い型を使うようにしてください。M-STRONG-TYPES-GUARD を参照してください。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 要素をグロブで再エクスポートしない (M-NO-GLOB-REEXPORTS) { #M-NO-GLOB-REEXPORTS }

<why>意図して決めた公開範囲。</why>

他のモジュールから `pub use foo::*` をしないでください。他のクレートからは特にそうです。意図したより多くのものをうっかりエクスポートしてしまうかもしれませんし、
グロブは PR でレビューしにくいものです。代わりに、要素を個別に再エクスポートしてください。

```rust,ignore
pub use foo::{A, B, C};
```

技術的な理由があれば、グロブでのエクスポートも認められます。たとえば、一連の HAL (hardware abstraction layer) モジュールからプラットフォーム固有の再エクスポートを行う場合です。

```rust,ignore
#[cfg(target_os = "windows")]
mod windows { /* ... */ }

#[cfg(target_os = "linux")]
mod linux { /* ... */ }

// グロブ再エクスポートの許容できる使い方。これはよくあるパターンで、
// すべてが 1 つのプラットフォームからそのまま転送されているだけなのが
// 明らかである。

#[cfg(target_os = "windows")]
pub use windows::*;

#[cfg(target_os = "linux")]
pub use linux::*;
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## static を避ける (M-AVOID-STATICS) { #M-AVOID-STATICS }

<why>クレートのバージョンをまたいだ一貫性と正しさ。</why>

要素の一貫した見え方が正しさにかかわる場合、ライブラリは `static` やスレッドローカルな要素を避けてください。
要するに、static が _魔法のように_ 別の値を持っていたら正しく動かなくなるコードでは、static を使ってはいけません。
パフォーマンスの最適化のためだけに使う static なら問題ありません。

Rust における static の根本的な問題は、状態がひそかに複製されることです。

次の関数を持つ `core` というクレートを考えます。

```rust
# use std::sync::atomic::AtomicUsize;
# use std::sync::atomic::Ordering;
static GLOBAL_COUNTER: AtomicUsize = AtomicUsize::new(0);

pub fn increase_counter() -> usize {
    GLOBAL_COUNTER.fetch_add(1, Ordering::Relaxed)
}
```

ここで、`main` というクレートが `library_a` と `library_b` という 2 つのライブラリを呼び出し、それぞれがそのカウンタを呼び出すとします。

```rust,ignore
// グローバルな static カウンタを 2 回増やす
library_a::count_up();
library_a::count_up();

// グローバルな static カウンタをさらに 3 回増やす
library_b::count_up();
library_b::count_up();
library_b::count_up();
```

最終的に、それぞれが結果を報告します。

```rust,ignore
library_a::print_counter();
library_b::print_counter();
main::print_counter();
```

この時点で、カウンタの _本当の_ 値はいくつでしょうか。`0`、`2`、`3`、`5` のどれでしょう?

答えは、クレートのバージョン解決によっては、上のどれにもなりえます (複数同時にさえ!)。

Rust は、宣言された依存を満たすために、同じクレートの複数のバージョンを、それぞれ独立にインスタンス化してリンクすることがあります。
これは特にクレートの `0.x` のバージョン期間中に起きやすく、この期間はそれぞれの `x` が別々の _メジャー_ バージョンになります。

`main`、`library_a`、`library_b` がすべて同じバージョンの `core` (例: `0.5`) を宣言していれば、報告される結果は `5` になります。
すべてのクレートが実際に同じバージョンの `GLOBAL_COUNTER` を _見て_ いるからです。

しかし、`library_a` が代わりに `0.4` を宣言していたら、`library_a` は別のバージョンの `core` にリンクされます。そのため `main` と `library_b` は
`3` という値で一致し、一方 `library_a` は `2` を報告します。

`static` な要素は便利なこともありますが、ライブラリが安定化する前や、static な変数と非 static な変数の使い方が絡み合う中で
_ひそかな複製_ が一貫性の問題を起こしうる状態に対しては、特に危険です。さらに static は単体テストの邪魔になり、
thread-per-core の設計では競合の原因にもなります。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 本番コードは println ではなくテレメトリを使う (M-LOG-NOT-PRINT) { #M-LOG-NOT-PRINT }

<why>必要な場所で診断情報が得られること。</why>

本番のコードパスでは、`println!` や `dbg!` ではなく、プロジェクトのテレメトリフレームワークを通じて診断情報を出してください。コンソール出力は、ユーザインタフェースとして意図的に標準出力に書き込む CLI のためのものです。
{% endraw %}
