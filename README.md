## はじめに

あなたは関数型言語を始めたばかりです。
「純粋な関数」という概念を理解するのにそれほど時間はかからないでしょう。
関数型言語を学んでいくと、関数型言語プログラマーがそのパラダイムに夢中になっていることに気が付くでしょう。
「純粋な関数はコード内を見なくても推論ができる」、「純粋な関数は戦争を起こしにくい」、「純粋な関数は参照透過性がある」など。
純粋な関数が素晴らしいのは間違いありません。
間違いありませんが、問題もあるのです...

純粋な関数とは、副作用のない関数のことです。
しかしプログラミングをやったことがある人ならば、副作用はプログラミングの目的のすべてだということを知っているでしょう。
誰も見ていないところで円周率の計算を100回やることに何の意味があるのでしょうか。
データベースにデータを保存できないなら、データベースの存在意義とは何でしょうか。
私たちは実在のデバイスからデータを読んだり、ネットワークにリクエストを送ったりする **必要があります**。
副作用なしでこれらのことを実現するのは不可能です。
それでも、関数型言語は純粋な関数で構成されます。
関数型言語プログラマーはこの問題をどのように解決するのでしょう。

簡単に言うと、数学者と同じことをします。騙すのです。
騙す、と言った時に実際にはルールには従います。
ですが、ルールの抜け道を見つけ、それを広げて象の群れを扱えるようにするのです。
騙す方法はニつあります。

1. Dependency Injectionを使う: 柵の向こう側に問題を切り分ける
2. Effect Functorを使う: 思いっきり遅延させる

## Dependency Injection

Dependency Injectionは副作用を扱う一つ目の方法です。
この方法では、関数の純粋でない部分を抜き出して、関数の引数に突っ込んでしまいます。
そして副作用を行うかどうかは他の関数の責任だ、と匙を投げるのです。
コードを見てみましょう。

```js
// logSomething :: String -> String
function logSomething(something) {
    const dt = (new Date()).toIsoString();
    console.log(`${dt}: ${something}`);
    return something;
}
```

この`logSomething()`関数には二つの純粋でない部分があります。
Dateオブジェクトを使うことと、コンソールに書き出すことです。
この関数は副作用を引き起こすだけでなく、毎ミリ秒ごとに違った結果を返すのです。
どうやってこの関数を純粋な関数にするのでしょうか。
純粋でない部分を抜き出して、関数の引数に突っ込みましょう。

```js
// logSomething: Date -> Console -> String -> *
function logSomething(d, cnsl, something) {
    const dt = d.toIsoString();
    return cnsl.log(`${dt}: ${something}`);
}
```

これを使うときは、自分で明示的に純粋ではないパーツを引数として渡す必要があります。


```js
const something = "Curiouser and curiouser!"
const d = new Date();
logSomething(d, console, something);
// ⦘ Curiouser and curiouser!
```

「一つ上のレイヤーに問題を棚上げしているだけで、前と同じだ。純粋じゃあない。」とあなたは思うかもしれません。
それは正しい考え方です。Dependency Injection はルールの抜け道です。知らない振りをしているだけなのです。
「ああ、すいません。`cnsl`オブジェクトの`log()`がIOを発生させるかどうかは知らないのです。誰かがそれをくれたのです。どこから来たのかも分かりません。」
これは少し怠惰のようにも思えます。

でも見かけよりも馬鹿なことではないのです。
`logSomething()`関数を見てみましょう。
もしこの関数を純粋ではない関数にしたいなら、あなたが関数の引数を使ってそれを設定しないといけないのです。
純粋な引数を渡せば、簡単に純粋な関数にすることができます。

```js
const d = {toISOString: () => '1865-11-26T16:00:00.000Z'};
const cnsl = {
    log: () => {
        // do nothing
    },
};
logSomething(d, cnsl, "Off with their heads!");
//  ￩ "Off with their heads!"
```

今この関数は(`undefined`を返す以外)何もしません。
しかし、この関数は完全に純粋です。
同じ引数で呼び出せば、毎回同じ結果を返します。
このことはとても重要です。
純粋ではない関数にするためには恣意的な行動が求められます。
言い換えるなら、関数が依存しているものはすべて引数にあるのです。
`Date`や`console`といったグローバルなものには依存しません。

また重要なことは、以前の純粋でない関数にも関数を渡すことが可能です。
違う例を見てみましょう。
フォームにユーザー名を入力することを想像してください。
私たちはこのユーザー名を取得したいものだと仮定します。

```js
// getUserNameFromDOM :: () -> String
function getUserNameFromDOM() {
    return document.querySelector('#username').value;
}

const username = getUserNameFromDOM();
username;
// ￩ "mhatter"
```

この例では、DOMに情報を求めています。
`document`はグローバルオブジェクトでいつでも変わる可能性があるため、このコードは純粋ではありません。
一つの解決策は`document`を引数として渡すことですが、今回の例では以下のように
`querySelector()`関数を引数として渡すことが可能です。

```js
// getUserNameFromDOM :: (String -> Element) -> String
function getUserNameFromDOM($) {
    return $('#username').value;
}

// qs :: String -> Element
const qs = document.querySelector.bind(document);

const username = getUserNameFromDOM(qs);
username;
// ￩ "mhatter"
```

あなたはまた文句を言うかもしれません。
今やったことと言えば、`getUsernameFromDOM()`から純粋でない部分を外に出しただけです。
それはなくなっていません。ただ`qs()`という別の関数に張り付けただけなのです。
一つの純粋でない関数を、一つの純粋な関数と一つの純粋でない関数の二つに分離したのですから、
コードを長くしただけにも思えます。

もう少し我慢してください。
`getUserNameFromDOM()`のテストをすることを考えてみましょう。
純粋なバージョンとそうでないバージョン、どちらが簡単でしょう。
純粋でないバージョンの場合、テストのために`document`オブジェクトが必要です。
その上、その`document`が内部に`username`というIDを持つ要素が存在しなければなりません。
ブラウザの外でテストしたい時には、ただ一つの小さな機能のテストのためであってもjsdom やヘッドレスブラウザといったものが必要になってしまいます。
しかし純粋なバージョンの場合、この問題は避けられます。

```js
const qsStub = () => ({value: 'mhatter'});
const username = getUserNameFromDOM(qsStub);
assert.strictEqual('mhatter', username, `Expected username to be ${username}`);
```

実際のブラウザやjsdom での統合テストをしなくて良い、と言っているのではありません。
この例が示しているのは、`getUserNameFromDOM()`の挙動が完全に予測できるということです。
`qsStub`を渡すと、いつでも`mhatter`を返します。
予測不可能な部分をより小さな`qs`に移動したのです。

望むならこの予測不可能な部分をどんどんと上に押しやることができます。
これを続けていくと、最終的にはアプリケーションのエントリーポイントに到達します。
この時のアプリケーションは薄い純粋ではないコードの殻の中に、純粋なコードの核があるといったものになります。
アプリケーションが大きくなればなるほど、予測可能な部分を広げるというのは大事になってきます。


### Dependency Injectionの欠点

この方法で、大きくて複雑なアプリケーションを作ることはできます。
テストは簡単になり、関数の依存は明確になります。
しかし欠点として関数の引数が下のように長くなって今います。

```js
function app(doc, con, ftch, store, config, ga, d, random) {
    // Application code goes here
 }

app(document, console, fetch, store, config, ga, (new Date()), Math.random);
```

引数はかなり下のレイヤーにある関数が使うものかもしれません。
この場合、下に到達するまでのすべてのレイヤーの関数にその引数を渡さなければなりません。
中間のレイヤーがその引数を全く必要としていなくても、です。
この「引数のドリルダウン」の問題を除いて、上のコードはそんなに悪いコードではありません。
依存性を切り離して明確にできるのは良いことです。
ただ、面倒くさいのです。

もう一つの方法があります。

## 遅延関数
関数型プログラマーが使う、もう一つの抜け道を見てみましょう。
その考え方は「実際に副作用が起きるまで、それは副作用ではない」というものです。
よく分からないので、掘り下げてみましょう。
次のコードを見てください。

```js
// fZero :: () -> Number
function fZero() {
    console.log('Launching nuclear missiles');
    // Code to launch nuclear missiles goes here
    return 0;
}
```

馬鹿げたコードです。
0が欲しいなら、直接そう書けばいいからです。
また、読者がJavaScriptで核兵器をコントロールするプログラムを書いている訳ではないことは知っています。
でも、ポイントを説明するのには役立ちます。
上の関数は明確に純粋な関数ではありません。
まずコンソールにメッセージを書き出して、それから核戦争を始めるのかもしれません。
その後で0を返します。
ミサイルを撃った後に、何か計算したいのだと想像してください。
このようなシナリオでは事前にどのように計算をするかの予定を立てるのは非常に合理的なことです。
ミサイルがどのタイミングで発射されるかには気を付けるべきです。
計算の途中で間違ってミサイルを撃ってしまうことがないようにしなければなりません。
なので`fZero()`を返すだけのラッパーを作ってみてはどうでしょう。
安全のためのラッパーのようなものです。


```js
// fZero :: () -> Number
function fZero() {
    console.log('Launching nuclear missiles');
    // Code to launch nuclear missiles goes here
    return 0;
}

// returnZeroFunc :: () -> (() -> Number)
function returnZeroFunc() {
    return fZero;
}
```

その返り値が実行されない限り、何回でも`returnZeroFunc()`を呼ぶことができます。
このコードは(理論的には)安全です。ミサイルが発射さえることはありません。

```js
const zeroFunc1 = returnZeroFunc();
const zeroFunc2 = returnZeroFunc();
const zeroFunc3 = returnZeroFunc();
// No nuclear missiles launched.
```

純粋な関数をもう少し形式的に定義してみましょう。
その後で、`returnZeroFunc()`を議論します。

純粋な関数である、とは

1. 観察可能な副作用がない。
2. 参照透過性がある。すなわち、同じ引数を入力した時にはいつでも同じ結果が出力される。

`returnZeroFunc()`の場合を見てみましょう。

`returnZeroFunc()`は副作用があるでしょうか。
先ほど確認したように`returnZeroFunc()`の呼び出しは
ミサイルを発射しません。その返り値を実行しない限りは何も起きません。
したがってここでは副作用はありません。

それでは`returnZeroFunc()`は参照透過でしょうか。
つまり、いつも同じ値を返すでしょうか。
次のコードでテストできます。

```js
zeroFunc1 === zeroFunc2; // true
zeroFunc2 === zeroFunc3; // true
```

しかし、まだ純粋な関数とは言えません。
`returnZeroFunc()`はそのスコープの外の値を参照しているからです。
この問題は次のように書けば避けられます。

```js
// returnZeroFunc :: () -> (() -> Number)
function returnZeroFunc() {
    function fZero() {
        console.log('Launching nuclear missiles');
        // Code to launch nuclear missiles goes here
        return 0;
    }
    return fZero;
}
```

この関数は純粋な関数ですが、JavaScriptは私たちの期待と少し違う動き方をします。
参照透過性を確認するために`===`演算子を使うことはできません。
これは`returnZeroFunc()`が実行する度に違う関数を作るからです。
しかし、コードを見れば参照透過だということを確認できます。

`returnZeroFunc()`は毎回同じ関数を返すということ以外していません。
それらの関数は参照されているメモリこそ違うものの、本質的には同じものです。

これらは小さく、きれいな抜け道です。
でも実際のコードでこのようなテクニックが使えるのでしょうか。
答えは「Yes」です。
でも、実際のコードでどのようにするかを見る前にもう少し上のコードを掘り下げてみましょう。
危険な`fZero()`に戻ってみましょう。

```js
// fZero :: () -> Number
function fZero() {
    console.log('Launching nuclear missiles');
    // Code to launch nuclear missiles goes here
    return 0;
}
```

試しに`fZero()`が返す0をミサイルを発射せずに使ってみましょう。
`fZero()`が最終的に返す0に対して、1を足して返す関数を作ります。

```js
// fIncrement :: (() -> Number) -> Number
function fIncrement(f) {
    return f() + 1;
}

fIncrement(fZero);
// ⦘ Launching nuclear missiles
// ￩ 1
```

おっと。核戦争が始まってしまいました。
もう一度やってみましょう。
でも今度は数字を返す関数ではなくて、最終的には数字を返す関数を返り値とする関数として
`fIncrement()`を定義します。


```js
// fIncrement :: (() -> Number) -> (() -> Number)
function fIncrement(f) {
    return () => f() + 1;
}

fIncrement(zero);
// ￩ [Function]
```

ふぅ。核戦争の危機を免れました。
このまま進めてみましょう。
この二つの関数があれば、すべての「遅延された数」を表すことが可能です。

```js
const fOne   = fIncrement(zero);
const fTwo   = fIncrement(one);
const fThree = fIncrement(two);
// And so on…
```

また、遅延された値を操作する演算を定義できます。

```js
// fMultiply :: (() -> Number) -> (() -> Number) -> (() -> Number)
function fMultiply(a, b) {
    return () => a() * b();
}

// fPow :: (() -> Number) -> (() -> Number) -> (() -> Number)
function fPow(a, b) {
    return () => Math.pow(a(), b());
}

// fSqrt :: (() -> Number) -> (() -> Number)
function fSqrt(x) {
    return () => Math.sqrt(x());
}

const fFour = fPow(fTwo, fTwo);
const fEight = fMultiply(fFour, fTwo);
const fTwentySeven = fPow(fThree, fThree);
const fNine = fSqrt(fTwentySeven);
// No console log or thermonuclear war. Jolly good show!
```

ここで何をしたか分かりましたか？
普通の数で行っていることを、遅延された数でも行っているのです。
数学者はこれを「同型」と呼んでいます。
すべての普通の数は、関数と組み合わせることで遅延された数に変換できます。

遅延された数はそれをラップする関数を実行することで取り出せます。
つまり、普通の数と遅延された数との間に完全な1対1対応があるということです。
これは思っているよりも、かなりわくわくさせる概念です。約束します。
すぐ後にこの概念に戻ってきます。

この関数でラップする、というのは正当な戦略です。
どんなものでも望む限り、関数の後ろ側に隠せます。
そして、その関数が呼び出されるまでは「純粋」なコードとして扱えます。
誰も核戦争は始めたくはありません。
普通のコード(核ミサイルを発射するコードではない)では、私たちはどこかで副作用が必要になります。
すべてのものを関数でラップすれば、私たちは正確に副作用をコントロールできます。
どこで副作用を発生させるかを、正確に決めることができるのです。
しかし毎回関数を書いて値をラップするのは骨が折れます。
また、すべての関数の「遅延バージョン」を作るのも大変に面倒くさいことです。
言語に組み込まれている`Math.sqrt()`のような関数は完璧です。
このような関数が我々の「遅延された値」に対して適応できたら素晴らしいのではないでしょうか。
どうしたらよいのでしょう。Effect Functor の世界へようこそ。

## Effect Functor

Effect Functor は関数による遅延そのものです。
なので、先ほど議論していた`fZero()`をEffect オブジェクトによる実装を見ていきます。
その前に、一度おさらいしておきましょう。

```js
// zero :: () -> Number
function fZero() {
    console.log('Starting with nothing');
    // Definitely not launching a nuclear strike here.
    // But this function is still impure.
    return 0;
}
```

Effect オブジェクトを作るコンストラクタを定義します。

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {};
}
```

このコードはそんなに見ないでください。
もう少しちゃんと作りこみます。
`fZero()`関数をEffect オブジェクトと使いたいのですがどうすればよいでしょうか。
「普通の」関数を受け取って、最終的には遅延された値に適用したいのです。
実際に副作用を起こさず、遅延されたままの状態でこれを行いたいのです。
この操作を`map`と呼びます。
この理由としては「普通」の関数を「遅延された値にも適用可能な」関数にmapするからです。
実装すると下のようになります。

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        }
    }
}
```

注意深く見ると疑問に思ううかもしれませんが、このコードは関数合成にそっくりです。
この点は後ほど見ていきます。
とりあえず今は動作を試してみましょう。


```js
const zero = Effect(fZero);
const increment = x => x + 1; // A plain ol' regular function.
const one = zero.map(increment);
```

うーん。副作用が遅延されているので、実際に何が起こっているかを見る方法がありませんね。
Effect オブジェクトを修正して遅延されたものの「トリガーを引ける」ようにしましょう。

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
    }
}

const zero = Effect(fZero);
const increment = x => x + 1; // Just a regular function.
const one = zero.map(increment);

one.runEffects();
// ⦘ Starting with nothing
// ￩ 1
```

お望みとあらば`map`を繫ぎ続けることもできます。

```js
const double = x => x * 2;
const cube = x => Math.pow(x, 3);
const eight = Effect(fZero)
    .map(increment)
    .map(double)
    .map(cube);

eight.runEffects();
// ⦘ Starting with nothing
// ￩ 8
```

ここからが面白いところです。
私たちは上のようなオブジェクトを「Functor」と呼びます。
Functorと言った時には、それが`map`という演算を持ち、いくつかのルールに従うということを表します。
このルールはできないことのルールではなく、何ができるかのルールです。
ルールというよりも、特権です。
Effect はFunctor なのでいくつかやれることがあります。
この内の一つは「合成」です。
それは次のようなルールです。

```
eがFunctorで,関数f, gがあるなら
e.map(g).map(f) は e.map(x => f(g(x))) と等しい 
```

言い換えると、二つmapが並んでいた場合は
関数合成したものをmapした場合に等しい、ということです。
つまり、Effect は次のような操作ができるのです。

```js
const incDoubleCube = x => cube(double(increment(x)));
// If we're using a library like Ramda or lodash/fp we could also write:
// const incDoubleCube = compose(cube, double, increment);
const eight = Effect(fZero).map(incDoubleCube);
```

この結果は`map()`を三つ並べた時と同じになることが保証されています。
つまりリファクタする際に、振る舞いを変えずにコードを変えられます。
これはパフォーマンスを改善する際などに有効かもしれません。
数字の例では別にどちらでも大差ありません。
より現実に近いコードを見ていきましょう。

### Effect を作るためのショートカット

Effect のコンストラクタは関数を引数に取ります。
私たちが遅延したい副作用も関数、例えば`Math.random()`や`console.log`を遅延したい時などは非常に便利です。
ですが、たまに遅延する必要のないものをEffect 内部で扱いたい時があります。
例えば、グローバルの`window`オブジェクトにconfigを置く場合を考えてみましょう。
その値を取得したい時はありますが、その操作は純粋ではありません。
この操作を単純にするために次のようなショートカットを書きます。

```js
// of :: a -> Effect a
Effect.of = function of(val) {
    return Effect(() => val);
}
```

これが便利かどうか調べるためにWebアプリを作っていることを考えましょう。
このアプリは記事一覧やプロフィールといった基本的な機能を持っています。
ユーザーが変わった際にコンテンツを動的に変更するにはどうしたらよいでしょう。
私たちは賢いエンジニアなので、下のように設定ファイルをグローバルに置くことにしました。

```js
window.myAppConf = {
    selectors: {
        'user-bio':     '.userbio',
        'article-list': '#articles',
        'user-name':    '.userfullname',
    },
    templates: {
        'greet':  'Pleased to meet you, {name}',
        'notify': 'You have {n} alerts',
    }
};
```

`Effect.of()` を使えば、これらの設定の値をEffect の中に突っ込むことができます。

```js
const win = Effect.of(window);
userBioLocator = win.map(x => x.myAppConf.selectors['user-bio']);
// ￩ Effect('.userbio')
```

### Effectの入れ子と取り出し

Effectを関数でマッピングした際には
その関数がまたEffectを返すような時もあります。

すでに`getElementLocator()`は定義しました。
文字列を遅延したEffect です。
もし実際にDOMを見つけたい時は、別の純粋でない関数である`document.querySelector()`を呼ばなければなりません。
これをEffect化しましょう。

```js
// $ :: String -> Effect DOMElement
function $(selector) {
    return Effect.of(document.querySelector(s));
}
```
二つを組み合わせると下のようになります。

```js
const userBio = userBioLocator.map($);
// ￩ Effect(Effect(<div>))
```

こいつは少し扱いずらいオブジェクトです。
中のdivにアクセスするためにはマッピングする関数の内部で
もう一度マップピングする必要があります。

```js
const innerHTML = userBio.map(eff => eff.map(domEl => domEl.innerHTML));
// ￩ Effect(Effect('<h2>User Biography</h2>'))
```

少しなんとかしてみましょう。
一度`userBio`まで戻ります。
退屈かもしれませんが、何が起こっているのかをしっかり説明したいのです。
`Effect('user-bio')`という記法は誤解を招きます。
より正確に書くなら次のようになります。

```js
Effect(() => '.userbio');
```

これも完全に正確ではありません。
さらに正確に書くと

```js
Effect(() => window.myAppConf.selectors['user-bio']);
```

これを`$`を使ってマッピングします。
マッピングは関数合成と同じなので、次のようになります。

```js
Effect(() => $(window.myAppConf.selectors['user-bio']));
```

これは次のコードと同義です。

```js
Effect(
    () => Effect.of(document.querySelector(window.myAppConf.selectors['user-bio'])))
);
```

さらに次のように変形できます。

```js
Effect(
    () => Effect(
        () => document.querySelector(window.myAppConf.selectors['user-bio'])
    )
);
```

実際に副作用を働かせているのは内側のコードだけで
外側にそのコードが漏れ出ていないことが分かると思います。

### Join

入れ子のEffectをタイピングするのは面倒くさいことです。
Effectの入れ子は解消したいのですが、その過程で副作用が発生しないようにしなければなりません。
Effectの場合にこれを実現するために、外側のEffectが`runEffect()`関数を実行すればよいのですが
これは少し良くないやり方です。
目的が副作用を発生させることではないからです。
なので、`runEffect()`と同じことをする`join()`を実装します。
入れ子のEffectを解消する際には`join()`、副作用をトリガーする際には`reunEffect()`と使い分けるようにします。

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
        join(x) {
            return f(x);
        }
    }
}
```

ユーザープロフィールの要素取得の例は次のように簡略化されます。

```js
const userBioHTML = Effect.of(window)
    .map(x => x.myAppConf.selectors['user-bio'])
    .map($)
    .join()
    .map(x => x.innerHTML);
// ￩ Effect('<h2>User Biography</h2>')
```

### Chain
`map()`の後に`join()`が続くというのは良くみられるパターンです。
実際にかなり多いので、ショートカットを定義するのは大変便利です。
ショートカットとして`map()`してから`join()`をする`chain()`を定義します。
Effectを返す関数でEffectをマッピングする時には、この`chain()`を用います。
コードは次にようになります。

```js
// Effect :: Function -> Effect
function Effect(f) {
    return {
        map(g) {
            return Effect(x => g(f(x)));
        },
        runEffects(x) {
            return f(x);
        }
        join(x) {
            return f(x);
        }
        chain(g) {
            return Effect(f).map(g).join();
        }
    }
}
```

これを使えば二つのEffectを「チェイン」することができます。
HTML要素取得の例は次のように簡略化されます。

```js
const userBioHTML = Effect.of(window)
    .map(x => x.myAppConf.selectors['user-bio'])
    .chain($)
    .map(x => x.innerHTML);
// ￩ Effect('<h2>User Biography</h2>')
```
