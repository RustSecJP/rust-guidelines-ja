{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# チェックリスト {#checklist}

- **共通**
  - [ ] 上流のガイドラインに従う ([M-UPSTREAM-GUIDELINES])
  - [ ] 静的検証を使う ([M-STATIC-VERIFICATION])
  - [ ] リントの上書きには `#[expect]` を使うべき ([M-LINT-OVERRIDE-EXPECT])
  - [ ] 公開型は Debug である ([M-PUBLIC-DEBUG])
  - [ ] 読まれることを意図した公開型は Display である ([M-PUBLIC-DISPLAY])
  - [ ] 迷ったらクレートを分割する ([M-SMALLER-CRATES])
  - [ ] 名前に曖昧な言葉を含めない ([M-WEASEL-WORDS])
  - [ ] アイテムの名前は短い ([M-SHORT-NAMES])
  - [ ] 関連関数より通常の関数を優先する ([M-REGULAR-FN])
  - [ ] マジック値はドキュメント化されている ([M-DOCUMENTED-MAGIC])
  - [ ] メッセージテンプレートを使った構造化ロギングを使う ([M-LOG-STRUCTURED])
- **ライブラリ / 相互運用性**
  - [ ] 型は Send である ([M-TYPES-SEND])
  - [ ] ネイティブなエスケープハッチ ([M-ESCAPE-HATCHES])
  - [ ] 外部の型を漏らさない ([M-DONT-LEAK-TYPES])
  - [ ] アイテムは元のクレートから取り込む ([M-FOREIGN-REEXPORTS])
  - [ ] 可能なら `impl AsRef<>` を受け付ける ([M-IMPL-ASREF])
  - [ ] 可能なら `impl RangeBounds<>` を受け付ける ([M-IMPL-RANGEBOUNDS])
  - [ ] 可能なら `impl 'IO'` を受け付ける（'sans IO'） ([M-IMPL-IO])
- **ライブラリ / UX**
  - [ ] 抽象化は目に見える形で入れ子にしない ([M-SIMPLE-ABSTRACTIONS])
  - [ ] API ではスマートポインタやラッパを避ける ([M-AVOID-WRAPPERS])
  - [ ] ジェネリクスより型を、dyn トレイトよりジェネリクスを優先する ([M-DI-HIERARCHY])
  - [ ] エラーは標準的な構造体である ([M-ERRORS-CANONICAL-STRUCTS])
  - [ ] 標準的なエラー変換には `map_err` ではなく `From` を使う ([M-FROM-ERROR])
  - [ ] 複雑な型の構築にはビルダを用意する ([M-INIT-BUILDER])
  - [ ] 複雑な型の初期化階層はカスケードさせる ([M-INIT-CASCADED])
  - [ ] サービスは Clone である ([M-SERVICES-CLONE])
  - [ ] 不可欠な機能は固有実装にすべき ([M-ESSENTIAL-FN-INHERENT])
  - [ ] モジュールは大きさと範囲のバランスが取れている ([M-BALANCED-MODULES])
  - [ ] プレリュードを定義しない ([M-NO-PRELUDE])
  - [ ] パラメータの順序は一貫している ([M-PARAMETER-CONSISTENCY])
  - [ ] コレクションは適切なイテレータのトレイトを実装する ([M-COLLECTION-TRAITS])
  - [ ] 関数は Future を返すのではなく `async` にする ([M-ASYNC-FN])
- **ライブラリ / 堅牢性**
  - [ ] I/O とシステムコールはモック可能にする ([M-MOCKABLE-SYSCALLS])
  - [ ] テスト用ユーティリティはフィーチャでゲートする ([M-TEST-UTIL])
  - [ ] 結合テストは `tests/` に置く ([M-INTEGRATION-TESTS])
  - [ ] 適切な型族を使う ([M-STRONG-TYPES])
  - [ ] ニュータイプは自身の不変条件を守る ([M-STRONG-TYPES-GUARD])
  - [ ] ビルダは最後の `.build()` で検証する ([M-BUILD-RESULT])
  - [ ] アイテムをグロブで再エクスポートしない ([M-NO-GLOB-REEXPORTS])
  - [ ] static を避ける ([M-AVOID-STATICS])
  - [ ] 本番コードは println ではなくテレメトリを使う ([M-LOG-NOT-PRINT])
- **ライブラリ / ビルド**
  - [ ] ライブラリはそのまますぐに動く ([M-OOBE])
  - [ ] ネイティブの `-sys` クレートは依存なしでコンパイルできる ([M-SYS-CRATES])
  - [ ] フィーチャは加算的である ([M-FEATURES-ADDITIVE])
- **マクロ**
  - [ ] マクロは最後の手段 ([M-MACRO-LAST-RESORT])
  - [ ] 手続き的マクロより「例によるマクロ」を優先する ([M-EXAMPLE-OVER-PROC])
  - [ ] マクロはシグネチャを偽らない ([M-MACROS-DONT-LIE])
  - [ ] マクロはメインクレートを前提とする ([M-MACRO-MAIN-CRATE])
  - [ ] 補助の手続き的マクロクレートのバージョンを固定する ([M-MACRO-VERSION-PIN])
  - [ ] サードパーティのアイテムは隠された `_private` モジュールから取り込む ([M-MACRO-HELPERS])
  - [ ] 手続き的マクロはテストを含む別の実装クレートを持つべき ([M-PROC-IMPL])
  - [ ] 手続き的マクロは暗黙のアイテムや隠れたアイテムを生成しない ([M-PROC-IMPLIED-ITEMS])
- **アプリケーション**
  - [ ] アプリには mimalloc を使う ([M-MIMALLOC-APPS])
  - [ ] アプリケーションは Anyhow やその派生を使ってよい ([M-APP-ERROR])
  - [ ] アプリケーションは実行可能な最も高い target-cpu を対象にする ([M-TARGET-CPU])
- **FFI**
  - [ ] FFI ライブラリ間で DLL の状態を分離する ([M-ISOLATE-DLL-STATE])
  - [ ] ビジネスロジックはコアクレートに置き、FFI は変換だけを行う ([M-FFI-TRANSLATES])
  - [ ] FFI クレートは確立された命名規則に従う ([M-FFI-NAMING])
- **正しさ**
  - [ ] unsafe には理由が必要であり、避けるべき ([M-UNSAFE])
  - [ ] unsafe は未定義動作を意味する ([M-UNSAFE-IMPLIES-UB])
  - [ ] すべてのコードは健全でなければならない ([M-UNSOUND])
  - [ ] パニックは「プログラムを停止する」ことを意味する ([M-PANIC-IS-STOP])
  - [ ] 検出したプログラミングのバグはエラーではなくパニックにする ([M-PANIC-ON-BUG])
  - [ ] パニック後の続行は最後の手段 ([M-PANIC-CONTINUATION])
  - [ ] 独自のパニックには役立つメッセージを付ける ([M-PANIC-MESSAGE])
- **パフォーマンス**
  - [ ] スループットを最適化し、空のサイクルを避ける ([M-THROUGHPUT])
  - [ ] ホットパスを早期に特定、プロファイル、最適化する ([M-HOTPATH])
  - [ ] 長時間実行されるタスクには yield ポイントを設けるべき ([M-YIELD-POINTS])
  - [ ] 可能ならアロケーションを再利用する ([M-MEM-REUSE])
  - [ ] ライブラリのテレメトリはパフォーマンスを大きく落とさない ([M-LOG-OVERHEAD])
  - [ ] 入れ子の型階層では不要な間接参照を避けるべき ([M-AVOID-INDIRECTION])
  - [ ] 所有権を持った不変のシーケンスにはボックス化されたスライスと文字列を使う ([M-BOX-DST])
  - [ ] 構築後にコレクションをサイズに合わせて縮める ([M-SHRINK-TO-FIT])
  - [ ] 可能なら高速なハッシャを使う ([M-FAST-HASHER])
  - [ ] コレクションは十分な初期容量で作成する ([M-INITIAL-CAPACITY])
  - [ ] ホットな `async` 関数はスタックサイズを減らす ([M-ASYNC-STACK-SIZE])
- **プロジェクト**
  - [ ] 共通の設定はワークスペースの Cargo.toml から取り込む ([M-CARGO-WORKSPACE])
  - [ ] ワークスペースはすべてのクレートを列挙し、バージョンを定める ([M-CRATES-IN-WORKSPACE])
  - [ ] すべてのクレートは1つのフォルダ内で兄弟として並べる ([M-CRATES-FLAT-FOLDER])
  - [ ] 新しいクレートは最新のエディションを対象にする ([M-LATEST-EDITION])
  - [ ] MSRV は控えめに更新する ([M-MSRV])
- **ドキュメント**
  - [ ] 最初の文は1行、およそ15語 ([M-FIRST-DOC-SENTENCE])
  - [ ] 包括的なモジュールのドキュメントがある ([M-MODULE-DOCS])
  - [ ] ドキュメントには標準的なセクションがある ([M-CANONICAL-DOCS])
  - [ ] `pub use` のアイテムには `#[doc(inline)]` を付ける ([M-DOC-INLINE])
- **AI**
  - [ ] AI での利用を念頭に置いて設計する ([M-DESIGN-FOR-AI])
  - [ ] アイテムは1つのパスからのみ見える ([M-SINGLE-ITEM-PATH])
  - [ ] メタ設計ドキュメントを避ける ([M-NO-META-DESIGN-DOCUMENTATION])
  - [ ] テストは自明な事実をアサートしない ([M-TAUTOLOGICAL-TESTS])
  - [ ] Rust のコードは Rust の問題を解く ([M-RUST-SHAPED])

<!-- Universal  -->
[M-UPSTREAM-GUIDELINES]: ../universal/#M-UPSTREAM-GUIDELINES
[M-STATIC-VERIFICATION]: ../universal/#M-STATIC-VERIFICATION
[M-LINT-OVERRIDE-EXPECT]: ../universal/#M-LINT-OVERRIDE-EXPECT
[M-PUBLIC-DEBUG]: ../universal/#M-PUBLIC-DEBUG
[M-PUBLIC-DISPLAY]: ../universal/#M-PUBLIC-DISPLAY
[M-SMALLER-CRATES]: ../universal/#M-SMALLER-CRATES
[M-WEASEL-WORDS]: ../universal/#M-WEASEL-WORDS
[M-SHORT-NAMES]: ../universal/#M-SHORT-NAMES
[M-REGULAR-FN]: ../universal/#M-REGULAR-FN
[M-DOCUMENTED-MAGIC]: ../universal/#M-DOCUMENTED-MAGIC
[M-LOG-STRUCTURED]: ../universal/#M-LOG-STRUCTURED
[M-LOG-NOT-PRINT]: ../libs/resilience/#M-LOG-NOT-PRINT

<!-- Libs -->
[M-TYPES-SEND]: ../libs/interop/#M-TYPES-SEND
[M-DONT-LEAK-TYPES]: ../libs/interop/#M-DONT-LEAK-TYPES
[M-FOREIGN-REEXPORTS]: ../libs/interop/#M-FOREIGN-REEXPORTS
[M-ESCAPE-HATCHES]: ../libs/interop/#M-ESCAPE-HATCHES
[M-STRONG-TYPES]: ../libs/resilience/#M-STRONG-TYPES
[M-STRONG-TYPES-GUARD]: ../libs/resilience/#M-STRONG-TYPES-GUARD
[M-NO-GLOB-REEXPORTS]: ../libs/resilience/#M-NO-GLOB-REEXPORTS
[M-ESSENTIAL-FN-INHERENT]: ../libs/ux/#M-ESSENTIAL-FN-INHERENT
[M-MOCKABLE-SYSCALLS]: ../libs/resilience/#M-MOCKABLE-SYSCALLS
[M-SIMPLE-ABSTRACTIONS]: ../libs/ux/#M-SIMPLE-ABSTRACTIONS
[M-AVOID-WRAPPERS]: ../libs/ux/#M-AVOID-WRAPPERS
[M-DI-HIERARCHY]: ../libs/ux/#M-DI-HIERARCHY
[M-ERRORS-CANONICAL-STRUCTS]: ../libs/ux/#M-ERRORS-CANONICAL-STRUCTS
[M-FROM-ERROR]: ../libs/ux/#M-FROM-ERROR
[M-INIT-BUILDER]: ../libs/ux/#M-INIT-BUILDER
[M-BUILD-RESULT]: ../libs/resilience/#M-BUILD-RESULT
[M-INIT-CASCADED]: ../libs/ux/#M-INIT-CASCADED
[M-SERVICES-CLONE]: ../libs/ux/#M-SERVICES-CLONE
[M-IMPL-ASREF]: ../libs/interop/#M-IMPL-ASREF
[M-IMPL-RANGEBOUNDS]: ../libs/interop/#M-IMPL-RANGEBOUNDS
[M-IMPL-IO]: ../libs/interop/#M-IMPL-IO
[M-BALANCED-MODULES]: ../libs/ux/#M-BALANCED-MODULES
[M-NO-PRELUDE]: ../libs/ux/#M-NO-PRELUDE
[M-PARAMETER-CONSISTENCY]: ../libs/ux/#M-PARAMETER-CONSISTENCY
[M-COLLECTION-TRAITS]: ../libs/ux/#M-COLLECTION-TRAITS
[M-ASYNC-FN]: ../libs/ux/#M-ASYNC-FN
[M-TEST-UTIL]: ../libs/resilience/#M-TEST-UTIL
[M-INTEGRATION-TESTS]: ../libs/resilience/#M-INTEGRATION-TESTS
[M-AVOID-STATICS]: ../libs/resilience/#M-AVOID-STATICS
[M-OOBE]: ../libs/building/#M-OOBE
[M-SYS-CRATES]: ../libs/resilience/#M-SYS-CRATES
[M-FEATURES-ADDITIVE]: ../libs/building/#M-FEATURES-ADDITIVE

<!-- Apps -->
[M-APP-ERROR]: ../apps/#M-APP-ERROR
[M-MIMALLOC-APPS]: ../apps/#M-MIMALLOC-APPS
[M-TARGET-CPU]: ../apps/#M-TARGET-CPU

<!-- FFI -->
[M-ISOLATE-DLL-STATE]: ../ffi/#M-ISOLATE-DLL-STATE
[M-FFI-TRANSLATES]: ../ffi/#M-FFI-TRANSLATES
[M-FFI-NAMING]: ../ffi/#M-FFI-NAMING

<!-- Correctness -->
[M-UNSAFE]: ../correctness/#M-UNSAFE
[M-UNSAFE-IMPLIES-UB]: ../correctness/#M-UNSAFE-IMPLIES-UB
[M-UNSOUND]: ../correctness/#M-UNSOUND
[M-PANIC-IS-STOP]: ../correctness/#M-PANIC-IS-STOP
[M-PANIC-ON-BUG]: ../correctness/#M-PANIC-ON-BUG
[M-PANIC-CONTINUATION]: ../correctness/#M-PANIC-CONTINUATION
[M-PANIC-MESSAGE]: ../correctness/#M-PANIC-MESSAGE

<!-- Performance -->
[M-HOTPATH]: ../performance/#M-HOTPATH
[M-THROUGHPUT]: ../performance/#M-THROUGHPUT
[M-YIELD-POINTS]: ../performance/#M-YIELD-POINTS
[M-MEM-REUSE]: ../performance/#M-MEM-REUSE
[M-LOG-OVERHEAD]: ../performance/#M-LOG-OVERHEAD
[M-AVOID-INDIRECTION]: ../performance/#M-AVOID-INDIRECTION
[M-BOX-DST]: ../performance/#M-BOX-DST
[M-SHRINK-TO-FIT]: ../performance/#M-SHRINK-TO-FIT
[M-FAST-HASHER]: ../performance/#M-FAST-HASHER
[M-INITIAL-CAPACITY]: ../performance/#M-INITIAL-CAPACITY
[M-ASYNC-STACK-SIZE]: ../performance/#M-ASYNC-STACK-SIZE

<!-- Project -->
[M-CARGO-WORKSPACE]: ../project/#M-CARGO-WORKSPACE
[M-CRATES-IN-WORKSPACE]: ../project/#M-CRATES-IN-WORKSPACE
[M-CRATES-FLAT-FOLDER]: ../project/#M-CRATES-FLAT-FOLDER
[M-LATEST-EDITION]: ../project/#M-LATEST-EDITION
[M-MSRV]: ../project/#M-MSRV

<!-- Docs -->
[M-FIRST-DOC-SENTENCE]: ../docs/#M-FIRST-DOC-SENTENCE
[M-MODULE-DOCS]: ../docs/#M-MODULE-DOCS
[M-CANONICAL-DOCS]: ../docs/#M-CANONICAL-DOCS
[M-DOC-INLINE]: ../docs/#M-DOC-INLINE

<!-- Macros -->
[M-MACRO-LAST-RESORT]: ../macros/#M-MACRO-LAST-RESORT
[M-EXAMPLE-OVER-PROC]: ../macros/#M-EXAMPLE-OVER-PROC
[M-MACROS-DONT-LIE]: ../macros/#M-MACROS-DONT-LIE
[M-MACRO-MAIN-CRATE]: ../macros/#M-MACRO-MAIN-CRATE
[M-MACRO-VERSION-PIN]: ../macros/#M-MACRO-VERSION-PIN
[M-MACRO-HELPERS]: ../macros/#M-MACRO-HELPERS
[M-PROC-IMPL]: ../macros/#M-PROC-IMPL
[M-PROC-IMPLIED-ITEMS]: ../macros/#M-PROC-IMPLIED-ITEMS

<!-- AI -->
[M-DESIGN-FOR-AI]: ../ai/#M-DESIGN-FOR-AI
[M-SINGLE-ITEM-PATH]: ../ai/#M-SINGLE-ITEM-PATH
[M-NO-META-DESIGN-DOCUMENTATION]: ../ai/#M-NO-META-DESIGN-DOCUMENTATION
[M-TAUTOLOGICAL-TESTS]: ../ai/#M-TAUTOLOGICAL-TESTS
[M-RUST-SHAPED]: ../ai/#M-RUST-SHAPED
{% endraw %}
