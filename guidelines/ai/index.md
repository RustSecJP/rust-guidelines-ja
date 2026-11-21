{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# AI のガイドライン {#ai-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## AI での利用を念頭に置いて設計する (M-DESIGN-FOR-AI) { #M-DESIGN-FOR-AI }

<why>コードベースで作業するエージェントから最大限の効果を得るため。</why>

一般的なルールとして、API を人間にとって使いやすくすれば、AI にとっても使いやすくなります。
この本のガイドラインに従っていれば、おおむね問題ありません。

Rust の強力な型システムはエージェントにとって大きな恵みです。エージェントには本当の理解が欠けていますが、
それは多くの場合、包括的なコンパイラのチェックで補えます。そして Rust はそうしたチェックを豊富に提供します。

とはいえ、Rust での AI によるコーディングをより効果的にするために、特に重要なガイドラインがいくつかあります。

* **慣用的な Rust の API パターンを作る**。API が公開か内部かにかかわらず、世の中の大多数の Rust のコードに見た目も使い心地も
近いほど、AI にとって良くなります。[Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html)
と、[ライブラリ / UX](../libs/ux) のガイドラインに従ってください。

* **詳しいドキュメントを用意する**。エージェントは詳しく良いドキュメントを好みます。クレートのすべてのモジュールと公開アイテムにドキュメントを付けてください。
読み手は Rust についてしっかりした、ただし専門家ほどではない理解を持ち、標準ライブラリを理解していると想定してください。
次のものに従ってください。
[C-CRATE-DOC](https://rust-lang.github.io/api-guidelines/checklist.html#c-crate-doc)、
[C-FAILURE](https://rust-lang.github.io/api-guidelines/checklist.html#c-failure)、
[C-LINK](https://rust-lang.github.io/api-guidelines/checklist.html#c-link)、
[M-MODULE-DOCS](../docs/#M-MODULE-DOCS)、
[M-CANONICAL-DOCS](../docs/#M-CANONICAL-DOCS)。

* **十分な例を用意する**。ドキュメントにはそのまま使える例を載せ、リポジトリにはもっと手の込んだ例を含めるべきです。
次のものに従ってください。
[C-EXAMPLE](https://rust-lang.github.io/api-guidelines/checklist.html#c-example)、
[C-QUESTION-MARK](https://rust-lang.github.io/api-guidelines/checklist.html#c-question-mark)。

* **強い型を使う**。厳密でよく文書化されたセマンティクスを持つ強い型を使って、[基本データ型への執着（primitive obsession）](https://refactoring.guru/smells/primitive-obsession) を避けてください。
次のものに従ってください。
[C-NEWTYPE](https://rust-lang.github.io/api-guidelines/checklist.html#c-newtype)。

* **API をテスト可能にする**。顧客が、あなたの API の使い方をユニットテストでテストできるように API を設計してください。そのためには、モックやフェイク、
cargo のフィーチャを導入する必要があるかもしれません。AI エージェントは、あなたの API を呼び出すコードが正しく動いていることを証明するために、
素早く繰り返し試せる必要があります。

* **テストカバレッジを確保する**。あなた自身のコードは、観察できる振る舞いについて十分なテストカバレッジを持つべきです。
そうすれば、エージェントはリファクタリングのときに、ほとんど手を借りずに作業できます。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## アイテムは1つのパスからのみ見える (M-SINGLE-ITEM-PATH) { #M-SINGLE-ITEM-PATH }

<why>各型への、ごちゃごちゃしない1つのパスのため。</why>

クレート内の公開アイテムは、1つのパスからのみ到達できるようにすべきです。たとえば、ある `crate::db::Connection` が `crate::Connection` としても見えるべきではありません。

```rust,ignore
// よくない
pub mod db {
    pub struct Connection;
}

pub use db::Connection;
```

このルールは、大きなコードベースを何度も繰り返して作成したりリファクタリングしたりするエージェントによって、よく破られます。エージェントは自分の作業を _簡単にする_ ために、意味のある場面で構造をきれいに設計し直すのではなく、アイテムを複数のパスで再エクスポートします。それは多くの場合、何らかの変更より前の古いパスです。

これはユーザ向けのアイテムの重複だけを対象にしていることに注意してください。クレート内では、エクスポートの木を組み立てる過程で同じアイテムが何度も現れることは許容されます（し、多くの場合避けられません）。

```rust,ignore
// よい
pub(crate) mod db {
    pub struct Connection;
}

pub use db::Connection;
```

同様に、外部のアイテムの再エクスポートはこのルールの対象外です。ただし、それらは [M-FOREIGN-REEXPORTS](../libs/interop/#M-FOREIGN-REEXPORTS) に従うべきです。

また、このルールは、マクロが必要とする、公開されているが隠された `_private` モジュールにも当てはまりません。[M-MACRO-HELPERS](../macros/#M-MACRO-HELPERS) を参照してください。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## メタ設計ドキュメントを避ける (M-NO-META-DESIGN-DOCUMENTATION) { #M-NO-META-DESIGN-DOCUMENTATION }

<why>ユーザに関係のあることに絞ったドキュメントにするため。</why>

クレートやモジュールのドキュメントには、クレートの作成中にだけ意味のあったメタ設計の話を含めてはなりません。つまり、ドキュメント化すべきなのは最終的な状態であり、設計の道のりではありません。

エージェントは、変更をどう設計したかを説明するセクションや、「なぜ Y ではなく X を選んだのか」という小論、設計日誌を、ユーザ向けのドキュメントの中によく書きます。こうした成果物は、プロジェクトで作業している間は興味深い診断情報かもしれませんが、エンドユーザにとってはほとんど意味がありません。

たとえば、エージェントは次のような自己報告を付け加えて、どのガイドラインに従ったと主張するかをまとめることがあります。

```text
| Rule | Applied | Where |
| --- | :---: | --- |
| M-SHORT-NAMES | ✅ | Shortened method names across the data-access and HTTP handler layers. |
| M-WEASEL-WORDS | ✅ | Removed weasel words from type and field names throughout the public API. |
| M-PUBLIC-DISPLAY | ✅ | Added `Display` impls for all user-facing identifier and error types. |
| M-ASYNC-FN | ✅ | Migrated I/O-facing APIs from `impl Future` returns to `async fn`. |
```

この種の内容は振る舞いではなく過程を説明するもので、時間とともに古くなります。とはいえ、プロジェクトの README に _設計原則_ などのセクションを設け、エンドユーザに関係のある、長く続くアーキテクチャ上の目標（例: クレートがアロケーションを行わないこと、OSI アーキテクチャを持つこと、`#[no_std]` を念頭に置いて設計されていること）を大まかに説明するのは、もちろんまったく妥当です。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## テストは自明な事実をアサートしない (M-TAUTOLOGICAL-TESTS) { #M-TAUTOLOGICAL-TESTS }

<why>ノイズではなく、価値を生むテストのため。</why>

ユニットテストは、基礎となる定義を繰り返すのではなく、意味のある振る舞いを検証すべきです。

エージェントは、テスト対象のコードが使うのと同じロジックから期待値を書き直したり、実装の分岐をそのままなぞったりするテストをよく作ります。こうしたテストは作りからして必ず成功し、ほとんど何の価値も生みませんが、その後の変更でのノイズの底上げになります。

```rust
const CHECKPOINTS: [u32; 4] = [0, 90, 180, 270];

#[test]
fn checkpoints_are_correct() {
    assert_eq!(CHECKPOINTS, [0, 90, 180, 270]);
}
```

これらがミューテーションテストを満たすために使われているなら、代わりにそのミューテーションテストをスキップすべきです。

代わりに、意味のあるテストは、その定数が満たすべき性質をチェックします。たとえば、等間隔であること、単調増加であること、関連するロジックで何らかの方向を強制することなどです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## Rust のコードは Rust の問題を解く (M-RUST-SHAPED) { #M-RUST-SHAPED }

<why>慣用的なコードのため。</why>

C#、Java、C++ などのコードを（自動的に）Rust に移植するとき、技術的な構造を1対1でそのまま写してはなりません。

ドメインの側面と言語の側面を分けて考えるのが賢明です。ドメインの側面はビジネス上の問題を扱います。素数を計算するアルゴリズムや、顧客テーブルを処理するロジックは、言語間で翻訳するときも同じように動作できます（し、そうすべきです）。

しかし、多くのパターンは、それが生まれたエコシステムに特有の問題を解くために存在します。Rust のエコシステムには独自の問題があり、それらには Rust でうまく機能するイディオムで対処する必要があります。これには次のものが含まれます。

- エラーハンドリング
- タスクとスレッドの管理
- コンポーネントの抽象化とそのライフタイム
- パラメータの所有権
- 「オブジェクト指向」プログラミングがないこと
- インタフェースとトレイトの構造的な違い
- その他多数

言語の構造の中には、まったく翻訳できないもの（例: C# と比べて、Rust には意味のあるリフレクションがありません）もあれば、見かけ上よく似ていて、何か月も後になって痛い目を見るかもしれないもの（例: static。[M-AVOID-STATICS](../libs/resilience/#M-AVOID-STATICS) を参照）もあります。

目安として、構造体とそのメソッドは、ビジネス上の機能に関する限り、名前、流れ、入力、出力がおおまかに似ていてもかまいません。しかし、Rust と { C#、Java、Python、... } の実装の間に技術的に目立つ類似点があれば、それはより深いアーキテクチャ上の問題を示しています。`throw_if_null()` が意味をなすことは決してありません。
{% endraw %}
