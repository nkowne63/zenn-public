---
title: "TypeScript 5.2で予告されているusingをいじってみる"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "babel", "react"]
published: false
publication_name: ventus
---

:::message
この記事でのusing宣言の動作はbabelのpolyfill実装に依存しており、実際のv8エンジンやTypeScriptのトランスパイル出力の挙動とは異なる可能性があります。
以下の挙動がusing宣言に対応している処理系の実際の挙動と異なる場合はコメントをいただけると幸いです。
:::

# 導入

先日、Twitterでこんなツイートが回ってきました。

https://twitter.com/mattpocockuk/status/1669630994280849408

TypeScript 5.2で新しい「using宣言」が追加されるというものです。

しかも、TypeScriptの独自構文かと思いきや、JavaScriptのstage 3の変更をTypeScriptで先行実装するというものでした。
新しい変数宣言の追加はES 2015(ES6)の「let」「const」以来でなんと8年ぶりであり、JavaScript/TypeScriptの常識を変えてしまう可能性がある機能追加と言えるでしょう。

この記事ではTypeScript 5.2のiteration planに先回りして、実際にusingがどのような挙動なのかを[tc39のプロポーザル](https://github.com/tc39/proposal-explicit-resource-management)やbabelの下でいじって確かめてみます。

# using宣言の例と動作

早速Babelで動かしてみます。
[Babelのプラグイン](https://babeljs.io/docs/babel-plugin-proposal-explicit-resource-management
)だけではシンボルが生えないので自力で生やしてあげる必要があるので注意が必要です。

```js
Symbol.dispose = Symbol.for("Symbol.dispose");
const g = () => ({
    [Symbol.dispose]() {
        console.log('dispose self');
    }
})

{
    console.log("disposable is not using")
    using disposable = g()
    console.log("disposable is using")
}
console.log("disposable is disposed")
```

```output
disposable is not using
disposable is using
dispose self
disposable is disposed
```

disposable変数がスコープを抜けたときに、変数の`@@dispose`メソッドが呼ばれているのがわかります。

基本的な仕組みはこれだけで、Babelの出力も見る限りだと非常にシンプルなコードになっていることがわかります。

```js
var g = function g() {
  return _defineProperty({}, Symbol.dispose, function () {
    console.log('dispose self');
  });
};
try {
  var _stack = [];
  console.log("disposable is not using");
  var disposable = _using(_stack, g());
  console.log("disposable is using");
} catch (_) {
  var _error = _;
  var _hasError = true;
} finally {
  _dispose(_stack, _error, _hasError);
}
console.log("disposable is disposed");
```

リソースの確保をしたあとに自動的に解放コードを呼ぶという使い道が主に想定されており、確保と解放をコールバックとして実装するよりかは扱いやすく素直なコードになることが期待できます。

# await using宣言

また、リソースの破棄を非同期に行うために、`await using`と`@@asyncDispose`も用意されています。

```js
Symbol.asyncDispose = Symbol.for("Symbol.asyncDispose");

const g = () => ({
    [Symbol.asyncDispose]() {
        return new Promise(resolve => {
            setTimeout(() => {
                console.log('dispose self');
                resolve();
            }, 1000);
        })
    }
})

const main = async () => {
    {
        await using disposable = g()
        console.log(disposable)
        console.log("disposable is using")
        console.log("wait 1000 ms")
    }
    console.log("disposable is disposed")
}

main()
```

```output
{ [Symbol(Symbol.asyncDispose)]: [Function (anonymous)] }
disposable is using
wait 1000 ms
dispose self
disposable is disposed
```

注意しなければいけないのは、**`await using`のタイミングではPromiseの解決は行われない**ことです。あくまでも`@@asyncDispose`が非同期になり、スコープを抜けるタイミングで解決が行われます。

また、`@@asyncDispose`がない場合はフォールバックとして`@@dispose`が使われます。

構文に引きずられてPromiseを返すと実行時エラーになります。（後述）

# その他for文との組み合わせ

他にも`for (using x of y)`, `for (await using x of y)`, `for await (using x of y)`, `for await (await using x of y)`という構文がfor文との組み合わせで使えるようになっています。usingに関する基本的な挙動は先程説明した通りなので割愛します。

（こういうfor文やそれに対する装飾（await, using）が1つ増えるたびに可能な構文が2倍になっていくので見ていてやや心配になる作りです...）

# DisposableStack

# using宣言の効用

# 意地悪してみる

## グローバルなusing

## disposableじゃないやつをusingする

## usingした変数を外部スコープの変数に代入

## usingした変数をreturnしてみる

## disposable stackを循環させてみる

# Reactでの挙動
