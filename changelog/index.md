{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# 変更履歴 {#changelog}

このページでは、Pragmatic Rust Guidelines の主な変更を記録しています。

## 2026.6 {#20266}

この更新ではガイドラインを大幅に拡充し、マクロ、パフォーマンス、プロジェクトのレイアウト、FFI、AI を活用した開発などにわたる
41 の新しい項目を追加しました。

| ガイドライン | タイトル |
|---|---|
| [M-ASYNC-FN](../guidelines/libs/ux/#M-ASYNC-FN) | 関数は Future を返すのではなく `async` にする |
| [M-ASYNC-STACK-SIZE](../guidelines/performance/#M-ASYNC-STACK-SIZE) | ホットな `async` 関数はスタックサイズを減らす |
| [M-AVOID-INDIRECTION](../guidelines/performance/#M-AVOID-INDIRECTION) | 入れ子の型階層では不要な間接参照を避けるべき |
| [M-BALANCED-MODULES](../guidelines/libs/ux/#M-BALANCED-MODULES) | モジュールは大きさと範囲のバランスが取れている |
| [M-BOX-DST](../guidelines/performance/#M-BOX-DST) | 所有権を持った不変のシーケンスにはボックス化されたスライスと文字列を使う |
| [M-BUILD-RESULT](../guidelines/libs/resilience/#M-BUILD-RESULT) | ビルダは最後の `.build()` で検証する |
| [M-CARGO-WORKSPACE](../guidelines/project/#M-CARGO-WORKSPACE) | 共通の設定はワークスペースの Cargo.toml から取り込む |
| [M-COLLECTION-TRAITS](../guidelines/libs/ux/#M-COLLECTION-TRAITS) | コレクションは適切なイテレータのトレイトを実装する |
| [M-CRATES-FLAT-FOLDER](../guidelines/project/#M-CRATES-FLAT-FOLDER) | すべてのクレートは1つのフォルダ内で兄弟として並べる |
| [M-CRATES-IN-WORKSPACE](../guidelines/project/#M-CRATES-IN-WORKSPACE) | ワークスペースはすべてのクレートを列挙し、バージョンを定める |
| [M-EXAMPLE-OVER-PROC](../guidelines/macros/#M-EXAMPLE-OVER-PROC) | 手続き的マクロより「例によるマクロ」を優先する |
| [M-FAST-HASHER](../guidelines/performance/#M-FAST-HASHER) | 可能なら高速なハッシャを使う |
| [M-FFI-NAMING](../guidelines/ffi/#M-FFI-NAMING) | FFI クレートは確立された命名規則に従う |
| [M-FFI-TRANSLATES](../guidelines/ffi/#M-FFI-TRANSLATES) | ビジネスロジックはコアクレートに置き、FFI は変換だけを行う |
| [M-FOREIGN-REEXPORTS](../guidelines/libs/interop/#M-FOREIGN-REEXPORTS) | アイテムは元のクレートから取り込む |
| [M-FROM-ERROR](../guidelines/libs/ux/#M-FROM-ERROR) | 標準的なエラー変換には `map_err` ではなく `From` を使う |
| [M-INITIAL-CAPACITY](../guidelines/performance/#M-INITIAL-CAPACITY) | コレクションは十分な初期容量で作成する |
| [M-INTEGRATION-TESTS](../guidelines/libs/resilience/#M-INTEGRATION-TESTS) | 結合テストは `tests/` に置く |
| [M-LATEST-EDITION](../guidelines/project/#M-LATEST-EDITION) | 新しいクレートは最新のエディションを対象にする |
| [M-LOG-NOT-PRINT](../guidelines/libs/resilience/#M-LOG-NOT-PRINT) | 本番コードは println ではなくテレメトリを使う |
| [M-LOG-OVERHEAD](../guidelines/performance/#M-LOG-OVERHEAD) | ライブラリのテレメトリはパフォーマンスを大きく落とさない |
| [M-MACRO-HELPERS](../guidelines/macros/#M-MACRO-HELPERS) | サードパーティのアイテムは隠された `_private` モジュールから取り込む |
| [M-MACRO-LAST-RESORT](../guidelines/macros/#M-MACRO-LAST-RESORT) | マクロは最後の手段 |
| [M-MACRO-MAIN-CRATE](../guidelines/macros/#M-MACRO-MAIN-CRATE) | マクロはメインクレートを前提とする |
| [M-MACROS-DONT-LIE](../guidelines/macros/#M-MACROS-DONT-LIE) | マクロはシグネチャを偽らない |
| [M-MEM-REUSE](../guidelines/performance/#M-MEM-REUSE) | 可能ならアロケーションを再利用する |
| [M-MSRV](../guidelines/project/#M-MSRV) | MSRV は控えめに更新する |
| [M-NO-META-DESIGN-DOCUMENTATION](../guidelines/ai/#M-NO-META-DESIGN-DOCUMENTATION) | メタ設計ドキュメントを避ける |
| [M-NO-PRELUDE](../guidelines/libs/ux/#M-NO-PRELUDE) | プレリュードを定義しない |
| [M-PANIC-CONTINUATION](../guidelines/correctness/#M-PANIC-CONTINUATION) | パニック後の続行は最後の手段 |
| [M-PANIC-MESSAGE](../guidelines/correctness/#M-PANIC-MESSAGE) | 独自のパニックには役立つメッセージを付ける |
| [M-PARAMETER-CONSISTENCY](../guidelines/libs/ux/#M-PARAMETER-CONSISTENCY) | パラメータの順序は一貫している |
| [M-PROC-IMPL](../guidelines/macros/#M-PROC-IMPL) | 手続き的マクロはテストを含む別の実装クレートを持つべき |
| [M-PROC-IMPLIED-ITEMS](../guidelines/macros/#M-PROC-IMPLIED-ITEMS) | 手続き的マクロは暗黙のアイテムや隠れたアイテムを生成しない |
| [M-RUST-SHAPED](../guidelines/ai/#M-RUST-SHAPED) | Rust のコードは Rust の問題を解く |
| [M-SHORT-NAMES](../guidelines/universal/#M-SHORT-NAMES) | アイテムの名前は短い |
| [M-SHRINK-TO-FIT](../guidelines/performance/#M-SHRINK-TO-FIT) | 構築後にコレクションをサイズに合わせて縮める |
| [M-SINGLE-ITEM-PATH](../guidelines/ai/#M-SINGLE-ITEM-PATH) | アイテムは1つのパスからのみ見える |
| [M-STRONG-TYPES-GUARD](../guidelines/libs/resilience/#M-STRONG-TYPES-GUARD) | ニュータイプは自身の不変条件を守る |
| [M-TARGET-CPU](../guidelines/apps/#M-TARGET-CPU) | アプリケーションは実行可能な最も高い target-cpu を対象にする |
| [M-TAUTOLOGICAL-TESTS](../guidelines/ai/#M-TAUTOLOGICAL-TESTS) | テストは自明な事実をアサートしない |

また、ガイドラインごとのバージョン管理を廃止しました。

## 2025 {#2025}

Pragmatic Rust Guidelines の最初のリリースです。
{% endraw %}
