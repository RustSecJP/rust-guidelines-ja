{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# アプリケーションのガイドライン {#application-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## アプリケーションでは mimalloc を使う (M-MIMALLOC-APPS) { #M-MIMALLOC-APPS }

<why>コストなしで大きな性能向上が得られるため。</why>

アプリケーションは、[mimalloc](https://crates.io/crates/mimalloc) をグローバルアロケータに設定するべきです。これにより、メモリ確保の多いホットパスで、たいてい目立った性能向上が得られます。私たちは、ベンチマークで最大 25% の改善を確認しています。

アロケータの変更は数行のコードで済みます。`Cargo.toml` に次のように mimalloc を追加します。

```toml
[dependencies]
mimalloc = { version = "0.1" } # 新しいバージョンがあればそれを使う
```

そして `main.rs` から使います。

```rust,ignore
use mimalloc::MiMalloc;

#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## アプリケーションは Anyhow やその派生を使ってよい (M-APP-ERROR) { #M-APP-ERROR }

<why>アプリケーションレベルのエラー処理を単純にするため。</why>

> 注: このガイドラインは主に、[M-ERRORS-CANONICAL-STRUCTS] を緩め、明確にするものです。

アプリケーションと、自分のリポジトリにあってそのアプリケーションからだけ使われるクレートは、独自の型を実装する代わりに、[ohno::AppError](https://docs.rs/crate/ohno/latest#structs)、[anyhow](https://github.com/dtolnay/anyhow)、[eyre](https://github.com/eyre-rs/eyre) などのアプリケーションレベルのエラークレートを使ってかまいません。

たとえば、アプリケーションのクレートでは、eyre の共通の `Result` 型をそのまま再エクスポートして使ってもかまいません。この型は、サードパーティのライブラリのエラー、特に [M-ERRORS-CANONICAL-STRUCTS] に従ったエラーを、すべて自動的に扱えるはずです。

```rust,ignore
use ohno::AppError;

fn start_application() -> Result<(), AppError> {
    start_server()?;
    Ok(())
}
```

アプリケーションのエラークレートを選んだら、アプリケーションレベルのエラーはすべてその型に切り替えるべきです。複数のアプリケーションレベルのエラー型を混ぜて使うべきではありません。

ライブラリ（複数のクレートから使われるクレート）は、代わりに常に [M-ERRORS-CANONICAL-STRUCTS] に従うべきです。

[M-ERRORS-CANONICAL-STRUCTS]: ../libs/ux/#M-ERRORS-CANONICAL-STRUCTS
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## アプリケーションは使える範囲で最も高い target-cpu を対象にする (M-TARGET-CPU) { #M-TARGET-CPU }

<why>サーバ群全体の性能のため。</why>

サーバアプリケーションは、汎用のベースラインをそのまま使うのではなく、デプロイ環境が確実にサポートする最も高い `target-cpu` に向けてコンパイルするべきです。

たとえば、`.cargo/config.toml` に次のように設定すれば実現できます。

```toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "target-cpu=x86-64-v3"]

[target.x86_64-pc-windows-msvc]
rustflags = ["-C", "target-cpu=x86-64-v3"]

# 必要に応じて、ここにほかのプラットフォームを追加する ...
```

このガイドラインはアプリケーションにだけ当てはまることに注意してください。ライブラリではターゲットの設定は無視されます。
{% endraw %}
