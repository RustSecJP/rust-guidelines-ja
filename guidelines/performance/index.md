{% raw %}
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

# 性能のガイドライン {#performance-guidelines}

<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## スループットを最適化し、空のサイクルを避ける (M-THROUGHPUT) { #M-THROUGHPUT }

<why>大規模での COGS 削減のため。</why>

ライブラリはスループットを重視して最適化するべきです。主要な指標の 1 つは _CPU サイクルあたりのアイテム数_ にするべきです。

これはレイテンシを軽んじるという意味ではありません&mdash;結局のところ、スループットはスケールさせて稼げますが、レイテンシはそうはいきません。しかし、
ほとんどの場合、1 件ずつの処理、競合するロック、頻繁なタスク切り替えに伴う _空のサイクル_ で、レイテンシの代償を払うべきではありません。

理想的には、次のようにするべきです。

- 前もって、妥当な大きさの作業の塊に分割する
- 個々のスレッドやタスクが、それぞれの作業の分担を独立して処理するようにする
- 作業がないときはスリープするか、yield する
- 自分の API をバッチ操作向けに設計する
- バッチ API があれば、それを使って作業する
- 長い個々のアイテムの途中や、バッチの塊の間で yield する（[M-YIELD-POINTS] を参照）
- CPU キャッシュ、時間的局所性、空間的局所性を活かす

次のことはするべきではありません。

- 個々のアイテムを少しでも早く受け取るために、ホットスピンする
- バッチ処理ができるのに、個々のアイテムごとに作業する
- 個々のアイテムを均等に分けるために、ワークスティーリングなどを行う

共有状態は、共有のコストが再計算のコストより小さい場合にだけ使うべきです。

[M-YIELD-POINTS]: ./#M-YIELD-POINTS
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ホットパスを早めに特定し、プロファイルし、最適化する (M-HOTPATH) { #M-HOTPATH }

<why>高性能なコードのため。</why>

開発の早い段階で、自分のクレートが性能や COGS に関わるものかどうかを見極めるべきです。関わるなら、次のことをしてください。

- ホットパスを特定し、それを対象にしたベンチマークを作る
- CPU とメモリ確保の情報を集めるプロファイラを定期的に実行する
- 最も性能に敏感な部分をドキュメントに書くか、周りに伝える

ベンチマークには [criterion](https://crates.io/crates/criterion) か [divan](https://crates.io/crates/divan) をお勧めします。
できれば、ベンチマークでは経過した実時間だけでなく、全スレッドで使った CPU 時間も計測するべきです（残念ながら、これには手作業が必要で、一般的なベンチマークツールではそのままではサポートされていません）。

Windows での Rust のプロファイリングは、[Intel VTune](https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html)
と [Superluminal](https://superluminal.eu/) でそのまま行えます。ただし、意味のある CPU の情報を得るには、`Cargo.toml` でベンチマーク用のデバッグシンボルを有効にするべきです。

```toml
[profile.bench]
debug = 1
```

最も性能に敏感な部分をドキュメントに書いておくと、ほかの貢献者がより良い判断をしやすくなります。最新のプロファイリングで見つかったホットスポットのスクリーンショットを共有するだけでもかまいません。

### さらに学ぶには {#further-reading}

- [Performance Tips](https://cheats.rs/#performance-tips)

> ### <tip></tip> どれくらい速くなる? {#how-much-faster}
>
> 私たちがよく目にした「言語に関係する」問題には、次のようなものがあります。
>
> - 頻繁な再アロケーション。特に、クローンされたり、伸びていったり、`format!` で組み立てられたりする文字列
> - バンプアロケーションなどで済むところでの、短命なアロケーション
> - String やコレクションのクローンによるメモリコピーのオーバーヘッド
> - 同じデータ構造の繰り返しの再ハッシュ
> - 衝突耐性が問題でない場面での、Rust のデフォルトのハッシャの使用
>
> 経験上、こうした `String` の問題の一部に対処しただけのホットパスで、ベンチマークが約 15% 向上したのを見ています。高度に最適化した版では、最大 50% に達しうるようです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 長時間実行されるタスクには yield ポイントを置く (M-YIELD-POINTS) { #M-YIELD-POINTS }

<why>すべてのタスクに CPU 時間を公平に配分するため。</why>

長時間かかる計算を行う場合、その中に `yield_now().await` のポイントを入れるべきです。

あなたのフューチャは、ブロックするタスクや長時間実行されるタスクにうまく対処できないランタイムで実行されるかもしれません。対処できるランタイムであっても、そうしたタスクは悪い設計とみなされ、ランタイムのオーバーヘッドを生みます。複雑なタスクが定期的に I/O を行うなら、その await ポイントが使われて自然に横取り (preempt) されます。

```rust, ignore
async fn process_items(items: &[items]) {
    // アイテムの処理を続ける。ランタイムが自動的に横取りしてくれる。
    for i in items {
        read_item(i).await;
    }
}
```

タスクが I/O を挟まずに長時間の CPU 処理を行うなら、並行する処理を飢えさせないように、代わりに一定の間隔で協調的に yield するべきです。

```rust, ignore
async fn process_items(zip_file: File) {
    let items = zip_file.read().async;
    for i in items {
        decompress(i);
        yield_now().await;
    }
}
```

個々の処理の数や時間が予測できないなら、`has_budget_remaining()` やそれに関連する API を使って、ホストしているランタイムに問い合わせるべきです。

> ### <tip></tip> どれくらいの頻度で yield する? {#yield-how-often}
>
> スレッド・パー・コアのモデルでは、タスク切り替えのオーバーヘッドと、無関係なタスクを飢えさせることによるシステム全体への影響とを、釣り合わせなければなりません。
>
> ランタイムのタスク切り替えに数百 ns かかり、さらに CPU キャッシュが失われるオーバーヘッドもあると仮定すると、
> 切り替えの間の連続した実行は、切り替えコストが無視できる（1% 未満になる）ほど長くするべきです。
>
> したがって、yield ポイントの間で 10 - 100μs の CPU バウンドな作業を行うのが、良い出発点になるでしょう。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## できるところではアロケーションを再利用する (M-MEM-REUSE) { #M-MEM-REUSE }

<why>アロケーションのオーバーヘッドを減らし、ホットパスを速くするため。</why>

API を設計するときは、ユーザが再利用できるリソースを持ち続けられるようにするべきです。自分のコードの中でも、そうしたリソースがあれば使うべきです。

ホットなループの中で繰り返しアロケーションを行うコストは大きくなりえます。しかも、ユーザから見ると、プロファイルしない限り気づかないことがあります。

```rust,ignore
// 悪い例: API の設計上、要素ごとに新しいアロケーションが必要になる。
for id in ids {
    let value = db.get(id);
}
```

このような API は便利さのためにあってもかまいませんが、補助的なものにするべきです。代わりに、中心となる API では、ユーザが元になるオブジェクトを所有し、再利用できるようにするべきです。

```rust,ignore
// 良い例: 新しいアロケーションが必要かどうかをユーザが決められる。
let mut value = Value::new();
for id in ids {
    db.get_in(id, &mut value);
}
```

再利用できる型を再利用するための定番のメソッドは `.clear()` で、多くの `std` のアイテムにあります。このパターンにはいくつかの種類があります。単純な場合は、ユーザが所有する型が、既存の再利用できるコレクションを直接持てます。

```rust
struct Value {
    data: Vec<u8>
}
```

重くて深く入れ子になったライブラリでは、バンプ方式の `Arena` を渡すか、ユーザの型の中に `Arena` をカプセル化して、呼び出しスタック全体で使えるようにする価値があるかもしれません。

```rust,ignore
struct Query {
    arena: Arena,
    request: Request,
    data: Vec<u8>    
}

fn client_do_work(query: &mut Query) {
    let request = rewrite_request(&query.request, &query.arena);
    get_in(request, &mut query.data);
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ライブラリのテレメトリで性能を落とさない (M-LOG-OVERHEAD) { #M-LOG-OVERHEAD }

<why>診断中もテレメトリのオーバーヘッドを小さく保つため。</why>

テレメトリを出力するライブラリのコードは、それがホットパスのスループットやレイテンシに目立った影響を与えないようにするべきです。

ログやメトリクスを出力する、サードパーティ向けのクレートは、テレメトリが常に有効であること、あるいは高負荷の下で有効であることを想定するべきです。したがって、出力するイベントの量とオーバーヘッドが妥当で、過度な性能低下を起こさないように注意するべきです。

ホットな内側のループでは、できればテレメトリの出力を完全になくすべきです。どうしても避けられないなら、出力するイベントは軽量にし、アロケーション（例: `format!` による文字列の連結）を避けるべきです。

```rust,ignore
// 悪い例: メッセージごとにログを出し、アロケーションを伴う整形を行う。
for m in messages {
    log(format!("Emitting message {}", m.id()))
}

// より良い例: メッセージごとのアロケーションを避ける。
for m in messages {
    log(("Emitting message", m.id()))
}

// 最も良い例: できるなら、何が起きたかをテレメトリの利用者が後でオフラインで再構成できるようにする
log(("Processing message batch", messages.batch_id()))
for m in messages { ... }
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 入れ子の型階層では不要な間接参照を避ける (M-AVOID-INDIRECTION) { #M-AVOID-INDIRECTION }

<why>キャッシュに優しい、速いメモリアクセスのため。</why>

ホットな型は、ヒープを介した入れ子の間接参照を避け、キャッシュ利用を改善するために、ホットでキャッシュしやすい深いフィールドを上の階層に引き上げることを検討するべきです。

一番良いのはベンチマークを取ることですが、C# のコードを Rust に移植するときに何度も現れるパターンがあります。入れ子の型を反射的に `Arc` で包み、それが何層にも重なることが多いのです。とても幅が広い型や重い型で、本当に複数の所有者が共有する必要があるなら、これは理にかなうこともあります。しかし、DRAM の参照を何回も順番に行わなければならない場合、このパターンはアクセスのレイテンシを台無しにすることがあります。

入れ子の共有所有権がどうしても必要というわけでないなら、たいていはローカルに埋め込んだデータから始め、キャッシュしやすいフィールドを引き上げるほうが良いでしょう。

```rust,ignore
// 悪い例: `print`（それなりにホットだとする）は、有効かどうかを
// 問い合わせるのに 2 回の間接参照が必要。
struct Item {
    config: Arc<Config>,
    payload: Payload,
}

struct Config {
    feature: Arc<Feature>
}

impl Item {
    fn print(&self) {
        if self.config.feature.is_enabled() { ... }
    }
}

// より良い例: `enabled` は近くにあり、`print` が呼ばれたときには
// すぐに使える可能性が高い。
struct Item {
    config: Arc<Config>,
    payload: Payload,
    enabled: bool,
}

impl Item {
    fn print(&self) {
        if self.enabled { ... }
    }
}

```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 不変で所有権を持ったシーケンスにはボックス化されたスライスと文字列を使う (M-BOX-DST) { #M-BOX-DST }

<why>メモリ消費を減らし、キャッシュをうまく使うため。</why>

頻繁に使われ、内部的で、生成後にサイズが変わらない不変のシーケンスは、元の `Vec<T>` や `String` ではなく、`Box<[T]>` や `Arc<str>` などとして保持するべきです。

普通の伸長可能なコレクションは、`(ptr, len, capacity)` の 3 つ組でできています。これをボックス化されたスライスに変換すると、不変になり、[shrink-to-fit](./#M-SHRINK-TO-FIT) が行われ、`capacity` の部分がなくなります。その結果、ハンドルのサイズが 1/3 減ります。このパターンが役に立つには、次の前提条件が当てはまるべきです。

- シーケンスが頻繁にインスタンス化されること（例: 数千個を超えるインスタンス）
- 不変でなければならないこと
- ユーザから見えないこと。つまり、普通のユーザは `&str` などを扱うだけであること

コレクションによっては、このための専用メソッドがあります。例: `String::into_boxed_str`。

```rust,ignore
// 悪い例: 要素が多いと、領域を無駄にし、
// 結果として走査も遅くなる。
struct Data {
    ids: Vec<String>
}

// 良い例: メモリ消費を減らし、より多くの要素が
// キャッシュに収まる。
struct Data {
    ids: Vec<Box<str>>
}
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## 構築後にコレクションを必要なサイズまで縮める (M-SHRINK-TO-FIT) { #M-SHRINK-TO-FIT }

<why>メモリ使用量を最小限にするため。</why>

`Vec` や `String` のような、大きくて長生きする伸長可能なコレクションを、正確なサイズを予約せずに構築した場合（[M-INITIAL-CAPACITY](./#M-INITIAL-CAPACITY) を参照）、できたコレクションは保存する前に `shrink_to_fit` で縮めるべきです。

Rust のコレクションの多くは、要素を繰り返し追加すると 2 のべき乗で大きくなります。そのため、最悪の場合、コレクションは必要なメモリの約 2 倍を使うことがあります。

```rust,ignore
// 悪い例: 長生きするオブジェクトが、必要なメモリの 2 倍を使うことになるかもしれない。
let mut long_lived = Vec::new();
for x in large_iter {
    long_lived.push(x);
}

// 良い例: 余分なメモリを解放する。
long_lived.shrink_to_fit();
```

これは、`into_boxed_*` などによる変換には当てはまらないことに注意してください（[M-BOX-DST](./#M-BOX-DST) を参照）。こうした変換は、一般に変換の前にすでに縮めているからです。
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## できるところでは高速なハッシャを使う (M-FAST-HASHER) { #M-FAST-HASHER }

<why>ハッシュの性能のため。</why>

信頼できる内部のキーをハッシュするときは、標準ライブラリのデフォルトよりも、暗号用でない高速なハッシャ（例: `foldhash`、`FxHash`）を選んでください。

Rust のデフォルトのハッシャは、信頼できないユーザ入力に対してそれなりに DoS に強くなっていますが、その代わりに性能が犠牲になっています。特定のバケットをあふれさせるように悪意をもってキーが作られることはないと信頼できるなら、独自の高速なハッシャで大きな性能向上が得られることがあります。

```rust,ignore
// 悪い例: 自分で管理しているキーにデフォルトのハッシャを使っている。
let lookup = HashMap::<UserID, Data>::with_capacity(1024);

// 良い例: 内部のキーに、より高速な foldhash を使っている。
let lookup = foldhash::HashMap<UserID, Data>::with_capacity(1024);
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## コレクションは十分な初期容量で生成する (M-INITIAL-CAPACITY) { #M-INITIAL-CAPACITY }

<why>コレクションを効率よく生成するため。</why>

コレクション（`Vec`、`String`、`HashMap`、`HashSet` など）の最終的なサイズやおおよそのサイズが生成時にわかっているなら、`new` や `default` ではなく `with_capacity` で生成するべきです。

容量を指定せずに生成したコレクションは、初期化の間に何度も再アロケーションされることがあり、そのたびに中身のコピーも起きます。十分な容量で生成すれば、この無駄なオーバーヘッドを完全に避けられます。

```rust,ignore
// 悪い例: おそらく何度も再アロケーションし、中身をコピーする。
let mut rval = Vec::new();
for x in &other {
    rval.push(convert(x));
}

// より良い例: 最初から十分な容量でコレクションを生成する。
let mut rval = Vec::with_capacity(other.len());
for x in &other {
    rval.push(convert(x));
}
```

イテレータによる生成 (`collect`) は、`size_hint` を通してこの動作を引き継ぎます。できるなら、手動の `push` ループよりもこちらを選ぶべきです。

```rust,ignore
// 理想的な例: 見た目も良く、性能も良い
let rval: Vec<_> = other.iter().map(convert).collect();
```
<!-- Copyright (c) Microsoft Corporation. Licensed under the MIT license. -->

## ホットな `async` 関数はスタックサイズを減らす (M-ASYNC-STACK-SIZE) { #M-ASYNC-STACK-SIZE }

<why>async のスタックサイズを小さくし、memcpy のオーバーヘッドを減らすため。</why>

ホットパスにある `async` 付きの関数は、そのフューチャのサイズを追跡し、影響を減らすために次の手段を 1 つ以上取るべきです。

- パラメータと戻り値の型のサイズを減らす
- `.await` ポイントをまたいで保持されるアイテムの型のサイズを減らす
- `impl Future` を返し、準備のロジックを `async {}` のキャプチャから切り出す

> ### <tip></tip> フューチャの「スタック」サイズ {#future-stack-sizes}
>
> フューチャでは、素朴に _そのスタック_ と考えられるものが、実際には内部のはるかに複雑な仕組みの一部になっています。
>
> 2 つの `.await` ポイントの間でだけ一時的に生きる普通のローカル変数は、ランタイムのスレッドの普通のスタックに残ります。しかし、`.await` ポイントをまたいで生きるローカル変数や、生成時に渡されるパラメータは、そのフューチャのステートマシン型の一部になります。そして、この型のレイアウトは、現在のところ最適化しきれていません。
>
> これにより、フューチャを生成したりボックス化したりするときに、スタックからヒープへの memcpy が起きることがあります。それだけでなく、関係する async 関数の、async をまたいだ呼び出しスタックのうち、想定しうる最も深い入れ子の分だけ、大きなスタックサイズを前もって必要とすることもあります（ちなみに、これが async 関数を単純に再帰できない理由です）。
>
> ```rust,ignore
> async fn foo(_large: Large) {
>     let within_future = [0_u8; 1024]; // 下の .await をまたぐので、`foo` の型に埋め込まれる
>     let on_stack = [0_u8; 1024]; // .await ポイントをまたがないので、スタック上に置かれる
>     let sneaky = Droppable::with_size(1024); // こっそり .await ポイントをまたぐ!
>     dbg!(&on_stack, &sneaky);
>     bar(&within_future).await;
>     dbg!(&within_future);
>     // <- ほかでは使われていないのに、`sneaky` はここでドロップされる!
> }
> 
> let future = foo(Large::new()); // `Large` は使われてすらいないのに、
>                                 // `foo` の型に埋め込まれて、
>                                 // そのサイズを膨らませる。
> 
> // ここでは、`foo` がまだ実行されていないのに、このスレッドのスタックメモリを
> // 最大で `Large` + 2kb 消費するかもしれない。spawn すると、これがランタイムの
> // Task 構造体に memcpy される:
> rt.spawn(future);
>```

多くの async 関数では、これは問題になりません。関連する `Future` のコストが無視できるほど小さいからです。しかし、ホットパスで使われ、頻繁に呼び出されたりインスタンス化されたりする関数（例: 毎秒数千回の呼び出しや、並行タスク）では、監視と最適化が役に立つかもしれません。

ホットなフューチャは、`size_of_val` で追跡するべきです。

```rust,ignore
async fn hot() { ... }

#[test]
fn has_reasonable_size() {
    let f = hot();
    assert!(size_of_val(&f) < ...); // 値や上限は、最初に実行したときに決める。
}
```

そのうえで、次の方法を組み合わせることを検討してください。

```rust,ignore
// 1) 代わりに `impl Future` を返す。これにより、特に大きな引数が
//    フューチャのサイズを膨らませるのを防げる。
fn hot(args: Args) -> impl Future<Output = Result<T>> { 
    // 2) 引数の処理に async の機能が要らないなら、
    //    async のコンテキストの外で処理する。
    let args = args.do_something(); 

    if args.invalid() {
        // 3) `Either` を使って 1 つの `impl Future` 型を返す。
        //    そうしないと、新しい型を作らなければならなくなる。
        async { Err(InvalidArgs) }.left_future() 
    } else {
        // 4) フューチャのヘルパを使ってフューチャの呼び出しを連結する。
        //    これも、重いローカル変数がステートマシンを通して
        //    受け渡されるのを防ぐ。
        read(args).then(|x| foo(x)).right_future() 
    }
}
```
{% endraw %}
