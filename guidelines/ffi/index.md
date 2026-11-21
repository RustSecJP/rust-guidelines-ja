{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# FFI のガイドライン {#ffi-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## FFI ライブラリ間で DLL の状態を分離する (M-ISOLATE-DLL-STATE) { #M-ISOLATE-DLL-STATE }

<why>DLL の境界をまたいでデータの整合性を保ち、動作を定義されたものにするため。</why>

1 つのアプリケーションの中で Rust 製の動的ライブラリ (DLL) を複数読み込む場合、それらのライブラリの間で共有してよいのは「ポータブル」な状態だけです。
同じように、そうしたライブラリを作る場合、他の DLL から受け取ったり、他の DLL に渡したりしてよいのは「ポータブル」なデータだけです。

ここでポータブルとは、どこから来たかに関係なく、安全かつ一貫して処理できるデータを意味します。定義上、これは FFI セーフな型の一部です。
型がポータブルであるのは、`#[repr(C)]`（または同様にきちんと定義されたもの）であり、かつ次の _すべて_ を満たす場合です。

- `static` やスレッドローカルとまったく関わらないこと。
- `TypeId` とまったく関わらないこと。
- ポータブルでないデータへの値、ポインタ、参照を含まないこと（ポータブルでないデータの中にあるポータブルなデータを指すのはかまいません。たとえば、`Box` に保持された ASCII 文字列への参照を共有するのは問題ありません）。

ここでいう _関わり_ とは、計算上のあらゆる関係を意味します。したがって、その型がどう使われるかにも関係します。`u128` を DLL 間で送るのは問題ありませんが、それを使って transmute した `TypeId` をやり取りするのはいけません。

根本的な問題は、Rust コンパイラが各 DLL を、独立したアプリケーションのように、まったく新しいコンパイル成果物として扱うことにあります。つまり、各 DLL は次のようになります。

- `static` 変数とスレッドローカル変数を、それぞれ独自に持ちます。
- `#[repr(Rust)]` 型（デフォルト）の型レイアウトは、コンパイルごとに異なることがあります。
- 独自の一意な型 ID を持ち、それはほかのどの DLL とも異なります。

特に、次のものが影響を受けます。

- ⚠️ メモリを確保したあらゆるインスタンス。例: `String`、`Vec<u8>`、`Box<Foo>` など
- ⚠️ ほかの static に依存するあらゆるライブラリ。例: `tokio`、`log`
- ⚠️ `#[repr(C)]` でないあらゆる構造体
- ⚠️ 一貫した `TypeId` に依存するあらゆるデータ構造

実際には、上のどれかをライブラリ間で受け渡すと、データの消失や状態の破損が起き、たいていは未定義動作になります。

これは、FFI の境界では見えない型やメソッドにも当てはまる場合があることに、特に注意してください。

```rust,ignore
/// DLL2 の共通サービスを使いたい、DLL1 のメソッド
#[ffi_function]
fn use_common_service(common: &CommonService) {
    // これには少なくとも 2 つの問題がある:
    // - `CommonService`、またはその奥深くに入れ子になったどの型も、
    //   DLL2 では型レイアウトが異なるかもしれず、
    //   すぐに未定義動作 (UB) になる ⚠️
    // - ここでの `do_work()` は DLL2 で呼び出されるように見えるが、
    //   実際に実行されるコードは DLL1 のものになる。つまり、
    //   ここで呼び出した `do_work()` は DLL2 から来たデータ構造を見るが、
    //   DLL1 の static を使う ⚠️
    common.do_work();
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ビジネスロジックはコアクレートに置き、FFI は変換だけを行う (M-FFI-TRANSLATES) { #M-FFI-TRANSLATES }

<why>安全なコードを最大限にし、関心事をきれいに分けるため。</why>

Rust で FFI ライブラリを作るときは、_ビジネスロジック_ を担うコアクレート `foo` と、つなぎ役のクレート `foo-ffi` の間で、関心事をはっきり分けるべきです。

実際の動作に関わる機能はすべてコアクレートに置き、慣用的で、安全で、テストできる Rust で書くべきです。FFI クレートは、ネイティブの Rust と C の構成要素の間で変換するためだけに存在します。型や関数のシグネチャを繰り返し書いて少しずつ調整することになるとしても、コアクレートに相互運用の都合を持ち込んではいけません。たとえば、コアクレート `foo` に次の型があるとします。

```rust,ignore
pub struct Message {
    destination: [u8; 8],
    data: Vec<u8>,
}

impl Message {
    pub fn new(destination: [u8; 8], data: Vec<u8>) -> Self { /* ... */ }
    pub fn transmit(&self) -> Result<(), TransmitError> { /* ... */ }
}
```

関心事を適切に分けるなら、`foo-ffi` では生成と送信を 1 つの FFI エントリポイントにまとめるかもしれません。

```rust,ignore
#[no_mangle]
pub unsafe extern "C" fn transmit_message(
    destination: *const [u8; 8],
    data: *const u8,
    data_len: usize,
) -> u8 {
    let data = std::slice::from_raw_parts(data, data_len).to_vec();
    match Message::new(*destination, data).transmit() {
        Ok(()) => 0,
        Err(_) => 1,
    }
}
```

しかし、FFI の要件を `foo` 自体に漏らすのは不適切です。所有権、データモデル、シグネチャは、2 つの世界の間でそのまま通用するわけではありません。きれいに分けるのを省いて _節約した_ 時間は、後のリファクタリングで何倍にもなって返ってきます。

```rust
#[repr(C)]
pub struct Message {
    pub destination: [u8; 8],
    pub data_ptr: *mut u8,
    pub data_len: usize,
    pub data_cap: usize,
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## FFI クレートは確立された命名規則に従う (M-FFI-NAMING) { #M-FFI-NAMING }

<why>プロジェクトをまたいでクレートの役割がすぐにわかるようにするため。</why>

FFI に使うクレートは、確立された命名慣習に従うべきです。

- `-sys` は、既存の（C スタイルの）ライブラリを呼び出すためのアイテムを定義するクレート
- `-ffi` は、既存のアプリケーションから呼び出される（C スタイルの）アイテムを定義するクレート

この方式には少し違うものもあります（例: 以前の `-sys` クレートが放棄されたときの `-sys2` や、`-` と `_` のどちらを使うか）。しかし全体としては、`-ffi` は「エクスポート」ライブラリ、`-sys` は「インポート」ライブラリをはっきり示します。
{% endraw %}
