{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# ライブラリ / ビルドのガイドライン {#libraries--building-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ライブラリは追加の準備なしで動く (M-OOBE) { #M-OOBE }

<why>Rust エコシステム全体での導入のしやすさ。</why>

ライブラリは、サポートするすべてのプラットフォームで _そのまま動か_ なければなりません。ただし、明らかに特定のプラットフォームやターゲット専用のライブラリは除きます。

Rust のクレートは何十もの依存を持つことがよくあり、アプリケーションなら何百にもなります。ユーザは `cargo build` や `cargo install`
が _そのまま動く_ ことを期待しています。約 250 の依存を取り込む `bat` のインストールを考えてみましょう。

```text
Compiling writeable v0.5.5
Compiling strsim v0.11.1
Compiling litemap v0.7.5
Compiling crossbeam-utils v0.8.21
Compiling icu_properties_data v1.5.1
Compiling ident_case v1.0.1
Compiling once_cell v1.21.3
Compiling icu_normalizer_data v1.5.1
Compiling fnv v1.0.7
Compiling regex-syntax v0.8.5
Compiling anstyle v1.0.10
Compiling vcpkg v0.2.15
Compiling utf8parse v0.2.2
Compiling aho-corasick v1.1.3
Compiling utf16_iter v1.0.5
Compiling hashbrown v0.15.2
Building [==>                       ] 29/251: icu_locid_transform_data, serde, winnow, indexma...
```

このコンパイルは、他の実質すべてのアプリケーションやライブラリと同じように、_そのまま動き_ ます。

特定の機能を対象としたツール (例: Wayland コンポジタ) や、`windows` のようなプラットフォーム用クレートもあります。
しかし、クレートが _明らかに_ プラットフォーム固有でないかぎり、それ以外の場合は _そのまま動く_ ことが期待されます。

つまり、クレートは最終的に次の条件でビルドできなければなりません。

- [ ] すべての [Tier 1 プラットフォーム](https://doc.rust-lang.org/rustc/platform-support.html) で<sup>1</sup>
- [ ] `cargo` と `rust` 以外の前提条件なしで<sup>2</sup>

<footnotes>

<sup>1</sup> Tier 1 プラットフォームを「今のところ」サポートしないのはかまいませんが、サポートを簡単に広げられるよう抽象化は用意しておかなければなりません。これは通常、
`dummy` のフォールバックターゲットを持つ内部の `HAL` ([Hardware Abstraction Layer](https://en.wikipedia.org/wiki/HAL_(software))) モジュールを導入することで実現します。<br/>
<sup>2</sup> デフォルトの Rust のインストールには `cc` とリンカも含まれています。

</footnotes>

特に、プラットフォーム用でないクレートは、デフォルトで、追加のツールをインストールするようユーザに求めたり、コンパイルに環境変数を
必要としたりしてはなりません。何らかのツールがどうしても必要な場合 (`.proto` ファイルから Rust のコードを生成するなど)、そのツールは
公開のワークフローの一部またはそれ以前に実行し、生成された成果物 (例: `.rs` ファイル) を公開するクレートに含めてください。

依存がプラットフォーム固有だとわかっている場合、親は条件付きの (プラットフォーム) コンパイルか、オプトインのフィーチャゲートを使わなければなりません。

> **<alert></alert> ライブラリは自分の依存に責任を持つ。**
>
> あなたが `Copilot` クレートを書いていて、それが `HttpClient` を使い、さらにその `HttpClient` がコンパイルに `perl` スクリプトを必要とするとします。
>
> すると、あなたのユーザも、そのユーザのユーザも、さらにその上の全員も、_自分の_ クレートをコンパイルするために Perl をインストールしなければなりません。大規模なプロジェクトでは、
> あなたのライブラリのことも Perl のことも知らない、あるいは気にしていない何百人もの人が、不可解なコンパイルエラーに遭遇し、自分のシステムに Perl を
> インストールする方法を調べなければならなくなります。
>
> 実際のところ、こうした振る舞いは、オープンソースの世界ではほぼ自ら死刑宣告をするようなものです。代わりのものが手に入った時点で、
> 人々は _そのまま動く_ ほうに乗り換えるからです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ネイティブの `-sys` クレートは依存なしでコンパイルできる (M-SYS-CRATES) { #M-SYS-CRATES }

<why>すべてのプラットフォームでそのまま動くライブラリ。</why>

ネイティブの `foo.lib` をラップする `foo` と `foo-sys` のクレートの組を書く場合、[M-OOBE] で述べた問題に
ぶつかる可能性が高いです。

プラットフォームをまたいで _そのまま動く_ クレートを作るには、次の手順に従ってください。

- [ ] `foo.lib` のビルドは、`foo-sys` の中の `build.rs` から完全に管理する。コンパイルは
  [cc](https://crates.io/crates/cc) クレートを使って手書きしたものだけにし、Makefile や外部のビルドスクリプトは実行 _しない_。外部の依存のインストールが必要になるからである
- [ ] `nasm` のような外部ツールはすべて任意にする
- [ ] 上流のソースコードをクレートに埋め込む
- [ ] 埋め込んだソースを検証できるようにする (例: Git の URL とハッシュを含める)
- [ ] 可能なら `bindgen` のグルーコードを事前に生成しておく
- [ ] 静的リンクと、[libloading](https://crates.io/crates/libloading) による動的リンクの両方をサポートする

これらの点から外れてもうまくいくことはあり、個別に検討してかまいません。

ネイティブのビルドシステムが _OOBE_ なクレートとして手に入るなら、`cc` の呼び出しの代わりにそれを使ってもかまいません。外部ツールについても同じです。

crates.io のサイズ制限に収まらない場合は、ソースコードをダウンロードする必要があるかもしれません。その場合でも、crates.io と同程度の
可用性を持つサーバだけを使ってください。さらに、許容するダウンロードの具体的なハッシュをクレートに保存し、検証してください。

隔離されたビルド環境ではソースのダウンロードが失敗することがあるので、代わりのソースの場所も指定できるようにしてください (例: 環境変数で)。

[M-OOBE]: ./#M-OOBE
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## フィーチャは加算的にする (M-FEATURES-ADDITIVE) { #M-FEATURES-ADDITIVE }

<why>大規模で複雑なプロジェクトでの確実なコンパイル。</why>

ライブラリのフィーチャはすべて加算的でなければならず、そのフィーチャ自体が現在のプラットフォームで動くかぎり、どんな組み合わせでも動かなければなりません。つまり、次のことが言えます。

- [ ] `no-std` フィーチャを導入してはならない。代わりに `std` フィーチャを使う
- [ ] どのフィーチャ `foo` を追加しても、公開要素を無効にしたり変更したりしてはならない
  - 列挙型が `#[non_exhaustive]` であれば、列挙子の追加はかまわない
- [ ] フィーチャは、他のフィーチャが手動で有効にされることを前提にしてはならない
- [ ] フィーチャは、親が子のフィーチャを飛ばして有効にしてくれることを前提にしてはならない

参考資料

- [Feature Unification](https://doc.rust-lang.org/cargo/reference/features.html#feature-unification)
- [Mutually Exclusive Features](https://doc.rust-lang.org/cargo/reference/features.html#mutually-exclusive-features)
{% endraw %}
