{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# プロジェクトのガイドライン {#project-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 共通の設定はワークスペースの Cargo.toml から取り込む (M-CARGO-WORKSPACE) { #M-CARGO-WORKSPACE }

<why>一貫性があり、保守しやすいプロジェクト設定のため。</why>

互いに何らかの関係がある2つ以上のクレートを持つリポジトリは、ワークスペースの `Cargo.toml` でそれらのクレートをまとめるべきです。そうすれば、メンバは共有のメタデータや依存のバージョンを、各クレートで値を重複させるのではなく、`[workspace.dependencies]` や `[workspace.lints]` などを通じてワークスペースのルートから継承します。

ある依存が特定のクレートだけのものであっても、ワークスペースで定義すべきです。ワークスペースの定義では、一般に依存のフィーチャを有効にすべきではありません（`["std"]` のような基本的なものは除きます）。代わりに `default-features = false` を使うべきです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ワークスペースはすべてのクレートを列挙し、バージョンを定める (M-CRATES-IN-WORKSPACE) { #M-CRATES-IN-WORKSPACE }

<why>クレート間の依存とデバッグを簡単にするため。</why>

プロジェクトが生成するすべてのクレートは、ワークスペースのメンバとして列挙すべきです。また、そのバージョンを `[workspace.dependencies]` で宣言し、ワークスペース内の依存が1つの正規のバージョンに解決されるようにすべきです。

```toml
# 悪い例。クレートが兄弟を直接リンクしている
[dependencies]
sibling.path = "../sibling"


# 良い例。ワークスペースを通している
[dependencies]
sibling.workspace = true

[workspace.dependencies]
sibling = { path = "crates/sibling", version = "0.5.2" }
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## すべてのクレートは1つのフォルダ内で兄弟として並べる (M-CRATES-FLAT-FOLDER) { #M-CRATES-FLAT-FOLDER }

<why>プロジェクト内を簡単に移動でき、標準的な Rust のレイアウトになるため。</why>

リポジトリはワークスペースの `Cargo.toml` を1つだけ持ち、すべての Rust のクレートはその下に置くべきです。そして、すべてのクレートはワークスペースの直下にある1つのサブディレクトリ（例: `crates/`）に置くべきです（クレートが1〜2ダース程度まで）。それを超える場合は、兄弟を整理するために何らかのフォルダ階層（例: `common/`、`server/`、`client/`）を使うべきです。

```bash
# ほとんどのワークスペースに理想的
Cargo.toml
crates/
  foo/Cargo.toml 
  foo_core/Cargo.toml 
  foo_proc/Cargo.toml 
  foo_tests/Cargo.toml 
  bar/Cargo.toml
  baz/Cargo.toml


# 大きなワークスペースなら可
Cargo.toml
crates/
  server/
    main/Cargo.toml 
    routes/Cargo.toml 
  client/
    foo/Cargo.toml
    bar/Cargo.toml
  common/
    error/Cargo.toml
```

クレートを他のクレートの中（その `Cargo.toml` と同じ階層かそれより下）、ましてやその `src/` フォルダの中に置くことは、決して許されません。クレートの関係を表したい場合は、代わりに共通の接頭辞を使います（例: `foo`、`foo_util`、`foo_build`）。

```bash
# 決して許されない。`src/` フォルダ内のクレート
Cargo.toml
crates/
  foo/Cargo.toml 
    src/lib.rs
       deps/bar/Cargo.toml 
```

このルールのまれな例外として、あなたのクレートがワークスペースを処理するもので、それが依存する UI テストなどの集まりを持っている場合があります。ただし、その場合でも、それらは普通ダミーのクレートです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 新しいクレートは最新のエディションを対象にする (M-LATEST-EDITION) { #M-LATEST-EDITION }

<why>最新の Rust の機能を使えるようにするため。</why>

新しいクレートやワークスペースを作るときは、`edition` を最新の安定版エディション（執筆時点では少なくとも `2024`）に設定してください。`resolver` フィールドは一般に必要ありません。

新しいプロジェクトで古いエディションを使っても、一般に利点はありません。それどころか、慣用的でなく、UX 上の扱いにくいケースが多い「古い Rust」を書くことを強いられます。特に、古いエディションを使っても、エコシステムの他の部分との互換性の面で利点は _何も_ 得られません。`2015` を元にしたアプリケーションでも、`2024` 向けに書かれたライブラリを問題なく使えます。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## MSRV は控えめに更新する (M-MSRV) { #M-MSRV }

<why>ユーザにとっての安定性を保ちつつ、新しい機能を使えるようにするため。</why>

ライブラリを最初に作るときに、最小サポート Rust バージョン（MSRV）を設定すべきです。新しい Rust の機能が必要になったら更新してかまいませんが、最新のコンパイラのリリースから数バージョン遅れた状態を保つべきです。

エコシステムでは、プロジェクトは _それなりに新しい_ Rust コンパイラでコンパイルされることが期待されています。

そのため、MSRV を上げるのにメジャーリリースは必要なく、マイナーアップデート（例: `1.3` から `1.4`）で扱えます。実際、サードパーティのクレートに依存するプロジェクトは、どれもすでに本質的にこの約束に縛られています。メジャーバージョンを上げることを強いても何の利点もなく、むしろ下流の依存を分裂させてしまうおそれがあります。
{% endraw %}
