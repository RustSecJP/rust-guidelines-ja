{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# 正しさのガイドライン {#correctness-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## unsafe には理由が必要で、避けるべき (M-UNSAFE) { #M-UNSAFE }

<why>メモリ安全性を保ち、攻撃面を最小限にするため。</why>

`unsafe` を使うには、正当な理由がなければなりません。正当な理由は次のものだけです。

1) 新しい抽象化。例: 新しいスマートポインタやアロケータ
1) 性能。例: `.get_unchecked()` を呼ぼうとする場合
1) FFI とプラットフォームの呼び出し。例: C やカーネルを呼び出す場合など

`unsafe` なコードは、コンパイラが使うガードレールを低くし、コンパイラの責任の一部をプログラマに移します。
できあがるコードが正しいかどうかは、主にコードレビューですべての誤りを見つけられるかにかかっていますが、これは間違いが起きやすいものです。
`unsafe` なコードの誤りは、深刻度の高いセキュリティ脆弱性を生むことがあります。

次の目的で、その場しのぎの `unsafe` を使ってはいけません。

- 性能が良く安全な Rust プログラムを短くする。例: `transmute` で列挙型のキャストを「簡単にする」
- `Send` などの境界を回避する。例: `unsafe impl Send ...` と書く
- `transmute` などでライフタイムの要件を回避する

ここでいう「その場しのぎ」とは、それとは関係のないコードの中に埋め込まれた `unsafe` のことです。もちろん、こうしたことを行う、きちんと設計された健全な抽象化を作るのはかまいません。

いずれの場合も、`unsafe` は以下に示すガイドラインに従わなければなりません。

### 新しい抽象化 {#novel-abstractions}

- [ ] 確立された代替手段がないことを確かめる。あるなら、そちらを選ぶ。
- [ ] 抽象化は最小限で、テストできるものでなければならない。
- [ ] [「敵対的なコード」](https://cheats.rs/#adversarial-code)に対して強化され、テストされていなければならない。特に、
  - クロージャを受け取るなら、クロージャがパニックしたときには無効（例: ポイズン状態）にならなければならない
  - どの安全なトレイトも誤動作すると想定しなければならない。特に `Deref`、`Clone`、`Drop`
- [ ] `unsafe` を使うときは、必ずその安全性を説明する平文の理由付けを添えなければならない
- [ ] 敵対的なテストケースも含めて、[Miri](https://github.com/rust-lang/miri) を通らなければならない
- [ ] そのほかすべての [unsafe code guidelines](https://rust-lang.github.io/unsafe-code-guidelines/) に従わなければならない

### 性能 {#performance}

- [ ] 性能のために `unsafe` を使うのは、ベンチマークを取った後だけにするべき
- [ ] `unsafe` を使うときは、必ずその安全性を説明する平文の理由付けを添えなければならない。これは、
  `unsafe` なメソッドを呼ぶ場合にも、`_unchecked` なメソッドを提供する場合にも当てはまる。
- [ ] 対象のコードは [Miri](https://github.com/rust-lang/miri) を通らなければならない
- [ ] [unsafe code guidelines](https://rust-lang.github.io/unsafe-code-guidelines/) に従わなければならない

### FFI {#ffi}

- [ ] `unsafe` な構成要素を避けるため、確立された相互運用ライブラリを使うことをお勧めする
- [ ] [unsafe code guidelines](https://rust-lang.github.io/unsafe-code-guidelines/) に従わなければならない
- [ ] 生成したバインディングには、どの呼び出しパターンが許されるのかがはっきりわかるようにドキュメントを書かなければならない

### さらに学ぶには {#further-reading}

- [Nomicon](https://doc.rust-lang.org/nightly/nomicon/)
- [Unsafe Code Guidelines](https://rust-lang.github.io/unsafe-code-guidelines/)
- [Miri](https://github.com/rust-lang/miri)
- [「敵対的なコード」](https://cheats.rs/#adversarial-code)
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## すべてのコードは健全でなければならない (M-UNSOUND) { #M-UNSOUND }

<why>バグや非互換性のない、予測できる実行時の動作のため。</why>

不健全なコードとは、一見 _安全_ に見えるのに、ほかの安全なコードから呼ばれたとき、またはそれ自体で、未定義動作を起こしうるコードのことです。

> ### <tip></tip> 「安全」の意味 {#meaning-of-safe}
>
> _安全_ (safe) と `unsafe` は、Rust の専門用語です。
>
> シグネチャで `unsafe` と示されていない関数は _安全_ です。とはいえ、_安全_ な関数でも危険なことはあります
> （例: `delete_database()`）。一方、`unsafe` な関数は、正しく使えばたいていまったく無害です（例: `vec.get_unchecked()`）。
>
> したがって、関数が _安全_ に見える（つまり `unsafe` と示されていない）のに、呼び出し方の _どれか一つ_ でも
> 未定義動作を起こすなら、その関数は _不健全_ です。これは最も厳しい意味で解釈します。未定義動作を起こすのが
> 「奇妙なコード」を必要とする「ごくわずかな、理論上の可能性」にすぎなくても、その関数は不健全です。
>
> [Unsafe, Unsound, Undefined](https://cheats.rs/#unsafe-unsound-undefined) も参照してください。

```rust
// 型を「安全に」変換する
fn unsound_ref<T>(x: &T) -> &u128 {
    unsafe { std::mem::transmute(x) }
}

// `Send` 境界がないのを回避する「巧妙な技」
struct AlwaysSend<T>(T);
unsafe impl<T> Send for AlwaysSend<T> {}
unsafe impl<T> Sync for AlwaysSend<T> {}
```

不健全な抽象化は決して許されません。何かを安全にカプセル化できないなら、代わりに `unsafe` な関数を公開し、正しい使い方をドキュメントに書かなければなりません。

<div class="warning">

例外なし

十分な理由があれば、ほとんどのガイドラインは破ってもかまいません。しかしこの場合には例外はありません。不健全なコードは決して受け入れられません。

</div>

> ### <tip></tip> モジュールの境界がすべて {#its-the-module-boundaries}
>
> 健全性の境界はモジュールの境界と同じであることに注意してください! ほかの部分は安全な抽象化の中で、
> **同じモジュール内** のどこか別の場所で保証された動作に頼る安全な関数を持つのは、まったく問題ありません。
>
> ```rust
> struct MyDevice(*const u8);
>
> impl MyDevice {
>     fn new() -> Self {
>        // インスタンスを正しく初期化する ...
>        # todo!()
>     }
>
>     fn get(&self) -> u8 {
>         // この関数自体には `self.0` が有効かどうかを確かめる手段がないが、
>         // それが有効であることに頼ってまったく問題ない。
>         unsafe { *self.0 }
>     }
> }
>
> ```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## unsafe は未定義動作を意味する (M-UNSAFE-IMPLIES-UB) { #M-UNSAFE-IMPLIES-UB }

<why>警告疲れを起こさず、意味の一貫性を保つため。</why>

`unsafe` という目印を関数やトレイトに付けてよいのは、誤用すると未定義動作 (UB) の危険がある場合だけです。
ほかの理由で呼び出すのが危険な関数に付けてはいけません。

```rust
// 正しい unsafe の使い方
unsafe fn print_string(x: *const String) { }

// 誤った unsafe の使い方
unsafe fn delete_database() { }
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## パニックは「プログラムを止める」という意味 (M-PANIC-IS-STOP) { #M-PANIC-IS-STOP }

<why>健全性と予測可能性のため。</why>

パニックは例外ではありません。パニックは、プログラムを直ちに終了させることを意味します。

コードは [_最低限_ パニック安全](https://doc.rust-lang.org/nomicon/exception-safety.html) でなければなりません（つまり、パニックから生き残っても未定義の状態になってはいけません）。それでも、パニックを起こすことは _このプログラムは今すぐ止まるべきだ_ という意味です。次のことは正当ではありません。

- パニックを使って（エラーを）上流に伝える
- パニックを使って、自分で招いたエラー状態を処理する
- パニックが捕まえられると想定する（自分のコードによってであっても）

たとえば、あなたのコードを呼び出すアプリケーションが、次の内容を含む `Cargo.toml` でコンパイルされているとします。

```toml
[profile.release]
panic = "abort"
```

すると、パニックを起こすたびに、本来は正常に動いているプログラムが無駄にアボートします。パニックを起こす正当な理由は次のとおりです。

- プログラミングの誤りに出会ったとき。例: `x.expect("must never happen")`
- const コンテキストから呼び出されるもの。例: `const { foo.unwrap() }`
- ユーザが求めたとき。例: 自分で `unwrap()` メソッドを提供する場合
- ポイズンに出会ったとき。例: ロックの結果に `unwrap()` を呼ぶ場合（ポイズンされたロックは、別のスレッドがすでにパニックを起こしたことを示します）

どれも、直接または間接にプログラミングの誤りと結びついています。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 検出したプログラミングのバグはエラーではなくパニックにする (M-PANIC-ON-BUG) { #M-PANIC-ON-BUG }

<why>エラー処理を扱いやすくし、実行時の一貫性を保つため。</why>

上の [M-PANIC-IS-STOP] を発展させたものとして、回復できないプログラミングの誤りを検出したときは、ライブラリもアプリケーションもパニックを起こさなければなりません。つまり、プログラムの終了を要求しなければなりません。

このような場合、`Error` 型を導入したり返したりするべきではありません。そうしたエラーには、実行時に対処しようがないからです。

契約違反、つまりライブラリ内部や呼び出し側による不変条件の破れは、プログラミングの誤りなので、パニックにしなければなりません。

ただし、何が違反に当たるかは状況によります。そうした検査は不可能だったりコストが高かったりするので、API がわざわざ違反を検出しにいくことは期待されていません。すでにある検査の中で `must_be_even == 3` に出会ったなら、明らかにパニックにするべきです。一方、関数 `parse(&str)` は明らかに `Result` を返さなければなりません。迷ったときは、標準ライブラリを参考にすることをお勧めします。

```rust, ignore
// 一般に、不正なパラメータを受け取った関数は、次のどれかをしなければならない
// - パラメータを無視する、または誤った結果を返す、あるいはその両方
// - Result などで問題を知らせる
// - パニックする
// この `divide_by` で y == 0 だとわかったなら、
// パニックするのが正しいやり方。
fn divide_by(x: u32, y: u32) -> u32 { ... }

// ただし、こうした検査を省き、規定されていない（ただし未定義ではない）
// 結果を返すことが許される場合もある。
fn divide_by_fast(x: u32, y: u32) -> u32 { ... }

// ここでは、不正な URI を渡すことは契約違反ではない。
// パースは本質的に失敗しうるので、Result を返さなければならない。
fn parse_uri(s: &str) -> Result<Uri, ParseError> { };

```

> ### <tip></tip> 「構築によって正しく」する {#make-it-correct-by-construction}
>
> 検出したプログラミングの誤りでパニックを起こすのは「いちばんましな選択肢」ですが、そのパニックが誰かの一日を台無しにするかもしれません。
> 本来ならパニックになるようなユーザ入力や呼び出し順序については、型システムを使って、パニックするコードパスをそもそもなくせないかも検討するべきです。

[M-PANIC-IS-STOP]: ./#M-PANIC-IS-STOP
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## パニック後の実行継続は最後の手段 (M-PANIC-CONTINUATION) { #M-PANIC-CONTINUATION }

<why>状態の整合性を保ち、わかりにくいバグを避けるため。</why>

`catch_unwind()` によるパニックからの回復は最後の手段であり、原則として、その後に制御されたアプリケーションの再起動を行わなければなりません。

パニックは、プログラムが回復できない状態に達したことを示します（[M-PANIC-IS-STOP](./#M-PANIC-IS-STOP) と [M-PANIC-ON-BUG](./#M-PANIC-ON-BUG) を参照）。特にライブラリのコードは、パニックを捕まえて実行を続けようとするべきではありません。本来ありえない状態を目にする危険があるからです。

```rust,ignore
thread_local! {
    static ALWAYS_EQUAL: RefCell<(i32, i32)> = RefCell::new((0, 0));
}

fn main() {
    let _ = panic::catch_unwind(|| {
        ALWAYS_EQUAL.with_borrow_mut(|p| {
            p.0 += 1;        
            panic!("Assume some user-provided closure failed here");  
            p.1 += 1;
        });
    });

    ALWAYS_EQUAL.with_borrow(|p| {
        assert_eq!(p.0, p.1);  // 壊れている!
    });
}
```

上の例は少し作為的ですが、捕まえたパニックの副作用や相互作用は、もっと見つけにくく、影響範囲が広く、わかりにくいことがあります。

無関係なタスクが多数同時に処理されているシステム（例: サーバのリクエストハンドラ）では、リクエストごとに `catch_unwind` を使ってもかまいません。それでも、リクエストハンドラがパニックを起こしたら、アプリケーションの再起動を促すべきです。ここでの `catch_unwind` の目的は、実行をいつまでも続けることではなく、ほかのすべてのリクエストを穏やかに終わらせることです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 独自のパニックには役に立つメッセージを付ける (M-PANIC-MESSAGE) { #M-PANIC-MESSAGE }

<why>バグをすばやく診断するため。</why>

コードが意図的にパニックを起こす場合（`panic!`、`assert!`、`unreachable!`、`todo!` など）、何が悪かったのかをはっきり述べ、必要に応じて関連する値を含むメッセージを付けなければなりません。

```rust,ignore
// 悪い例: このパニックでは、開発者が対処する手がかりがほとんどない。
assert!(buffer.len() >= HEADER_SIZE);

// 良い例: メッセージに理由と実際の値が含まれている。
assert!(buffer.len() >= HEADER_SIZE, "buffer too small for header: got {} bytes, need {HEADER_SIZE}", buffer.len());
```

API の誤用に関するメッセージは、エンドユーザの役に立つものにするべきです。バグを示すメッセージは、作者であるあなた自身や、あなたの後でプロジェクトを保守する人が、根本原因をすばやく特定できるものにするべきです。

テストの中のパニックメッセージは、一般には必要ありません。
{% endraw %}
