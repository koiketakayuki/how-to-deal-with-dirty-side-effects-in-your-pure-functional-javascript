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

