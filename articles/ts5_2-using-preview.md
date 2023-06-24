---
title: "TypeScript 5.2で予告されているusingをいじってみる"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
publication_name: ventus
---

:::message
この記事でのusing宣言の動作はBabelのtransform及びes-shimsのpolyfill実装に依存しており、実際のV8エンジンやTypeScriptのトランスパイル出力の挙動とは異なる可能性があります。
以下の挙動がusing宣言に対応している処理系の実際の挙動と異なる場合はコメントをいただけると幸いです。
:::

# 導入

先日、Twitterでこんなツイートが回ってきました。

https://twitter.com/mattpocockuk/status/1669630994280849408

TypeScript 5.2で新しい「using宣言」が追加されるというものです。

しかも、TypeScriptの独自構文かと思いきや、JavaScriptのStage 3のProposalをTypeScriptで先行実装するという通常のTypeScriptの実装プロセスに則ったものでした。
新しい変数宣言の追加はES 2015(ES6)の「let」「const」以来でなんと8年ぶりで、JavaScript/TypeScriptの常識を変えてしまう可能性がある機能追加と言えるでしょう。

この記事では[TypeScript 5.2のIteration Plan](https://github.com/microsoft/TypeScript/issues/54298)の先回りをして、実際にusingや、その周辺のExplicit Resource Managementはどのような挙動なのかを[tc39のProposal](https://github.com/tc39/proposal-explicit-resource-management)やBabelの下でいじって確かめてみます。

# using宣言の例と動作

早速Babelで動かしてみます。
[Babelのプラグイン](https://babeljs.io/docs/babel-plugin-proposal-explicit-resource-management
)だけではシンボルが生えないので、[es-shims/DisposableStack](https://github.com/es-shims/DisposableStack)などで自力で生やしてあげる必要があります。

```js
const g = () => ({
    [Symbol.dispose]() {
        console.log("dispose self");
    }
});

{
    console.log("disposable is not using");
    using disposable = g();
    console.log("disposable is using");
}
console.log("disposable is disposed");
```

```output
disposable is not using
disposable is using
dispose self
disposable is disposed
```

処理系がdisposable変数のスコープを抜けたときに、disposable変数の`Symbol.dispose`メソッドが呼ばれているのがわかります。

基本的な仕組みはこれだけで、Babelの出力も見る限りだと非常にシンプルなコードになっていることがわかります。

```js
var g = function g() {
  return _defineProperty({}, Symbol.dispose, function () {
    console.log("dispose self");
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

確保したリソースについて自動的に解放コード呼ぶという使い道が主に想定されており、確保と解放のインターフェースをコールバックとして提供するよりは扱いやすく素直なコードになることが期待できます。



# await using宣言

リソースの破棄を非同期に行うために`await using`と`Symbol.asyncDispose`も用意されています。
（なお、`Symbol.asyncDispose`がない場合はフォールバックとして`Symbol.dispose`が使われます。）

```js
const g = () => ({
    [Symbol.asyncDispose]() {
        return new Promise(resolve => {
            setTimeout(() => {
                console.log("dispose self");
                resolve();
            }, 1000);
        });
    }
});

const main = async () => {
    {
        await using disposable = g();
        console.log(disposable);
        console.log("disposable is using");
        console.log("wait 1000 ms");
    }
    console.log("disposable is disposed");
};

main();
```

```output
{ [Symbol(Symbol.asyncDispose)]: [Function (anonymous)] }
disposable is using
wait 1000 ms
dispose self
disposable is disposed
```

**`await using`の直後に宣言されている`disposable`はPromiseではなく、したがってPromiseの解決待ちをすることもありません。**
非同期の`Symbol.asyncDispose`メソッドは、処理系が`disposable`のスコープを抜けるときに実行され、処理の完了を待ってから次の行に移ります。
構文に引きずられてPromiseを`disposable`に代入すると実行時エラーになります。（後述）

以上の性質から、`await using`がある箇所とは別の場所で、暗黙的にPromiseの解決待ちが発生します。
具体的には、以下のようになります。（Proposalの[Implicit Async Interleaving Points ("implicit await")](https://github.com/tc39/proposal-explicit-resource-management#implicit-async-interleaving-points-implicit-await)から引用）

```js
async function f() {
  {
    a();
  } // exit block
  b(); // same microtask as call to `a()`
}
```

```js
async function f() {
  {
    await using x = ...;
    a();
  } // exit block, implicit `await`
  b(); // different microtask from call to `a()`.
}
```

# その他for文との組み合わせ

他にも`for (using x of y)`, `for (await using x of y)`, `for await (using x of y)`, `for await (await using x of y)`という構文が定義されています。usingに関する基本的な挙動は先程説明した通りなので割愛します。

詳しくは[Proposalのawait using in for-of and for-await-of Statements](https://github.com/tc39/proposal-explicit-resource-management#await-using-in-for-of-and-for-await-of-statements)を参照してください。

（こういうfor文やそれに対する装飾（await, using）が1つ増えるたびに認知負荷がゴリゴリ上がっていきそうで見ていてやや心配になる作りです...）

# DisposableStack

少し本筋とは外れますが、using宣言以外にもこのProposalではグローバルにオブジェクトが2つ追加されていいます。
`DisposableStack`と`AsyncDisposableStack`です。


`DisposableStack`は文字通り、1つのオブジェクトに複数のdisposableなオブジェクトをまとめているもので、それ自体もdisposableです。
なので、リソースなどをまとめて管理し、まとめて解放したいときに使うことができます。
また、stackと名前に入っている通り、解放は登録したときとは逆順に行われます。
`AsyncDisposableStack`は`DisposableStack`の非同期版です。

（コードはProposalの[Implicit Async Interleaving Points ("implicit await")](https://github.com/tc39/proposal-explicit-resource-management#aggregation)から引用）

```js
// sync
const stack = new DisposableStack();
const resource1 = stack.use(getResource1());
const resource2 = stack.use(getResource2());
const resource3 = stack.use(getResource3());
stack[Symbol.dispose](); // disposes of resource3, then resource2, then resource1
```

解放の途中でリソースの1つが例外を投げても、他のリソースの解放が終了してから例外が投げられます。（リソースの投げた例外がそのまま投げられます。）
また、解放時に例外を投げたリソースが複数ある場合は、`SupressedError`に例外がネストされてまとめて投げられます。

`DisposeStack`には`Symbol.dispose`以外にも便利なメソッドがあり、`Symbol.dispose`メソッドを持たないオブジェクトを解放するためのコールバックとともに登録できる`adopt`メソッドや、解放時に呼ばれるコールバックを登録する`defer`メソッド、新しい`DiposeStack`にリソースの所有権を移す`move`メソッドなどがあります。

```js
// 以下の2つは等価
// stack.adoptによる実装
stack.adopt(
    { sample: "resource" }, // Symbol.disposeメソッドを持たない
    () => { releaseObject(); }
);
stack[Symbol.dispose]();

// using宣言による実装
using g = {
    sample: "resource",
    [Symbol.dispose]() {
        releaseObject();
    }
};
```

# using宣言の利点

usingの主な利点は、リソースの解放操作をオブジェクト生成時に定義できることですが、他に以下の利点もProposalでは指摘されています。

- インターフェース統一：DOM API、NodeJS APIなどで様々な方法で書かれていたリソース解放のインターフェースを統一できる
- バグ防止：間違えてリソースを解放したり、リソースの解放順番を間違えたり、解放したあとのリソースにアクセスしたりといったなミスを予防できる

順に見ていきましょう。

## インターフェース統一

DOMやNodeJSのAPIにはリソースを扱うものが多くあります。
DOM APIにおいては、AudioContextやFileReader、WebSocketといったOS・ネットワーク関連のリソースだけでなく、ResizeObserverといったフロントエンド特有のものもリソースに含まれます。
NodeJSのAPIだとchild_process.ChildProcessやhttps.Serverだけでなく、stream.Readableといったやや抽象的なものもあります。
そして、これらのリソースの解放は全て違うメソッドでした。
具体的には以下のものが思い思いに使われていました。

- close()
- abort()
- disconnect()
- cancel()
- stop()
- releaseLock()
- unregister()
- terminate()
- pauseAnimations()
- unref()
- disable()
- kill()
- final()
- closeSync()
- destroy()
- end()

...自由ですね。
これらのリソース解放系のメソッドのラッパーとして`Symbol.dispose`が用意されます。
書くのが楽になりそうですね。

## バグ防止

リソース管理はリソースが1つだけなら大したことないように見えますが、複数を同時管理する場合は以下のことに気をつけなければなりません。

- 必要なリソースを間違って解放しない
- 解放したリソースにアクセスしない
- 依存関係にある複数のリソースを間違った順番で解放しない

リソースを間違って解放するコードが機能改修などにより挿入された場合、その行が実行される後に実行されるリソースを使うコードは基本的にエラーになってしまいます。

また、厄介なことに、今までのJSではリソースを特定の変数スコープに閉じ込めることは困難でした。
具体的には以下のようなことが起こる危険性がつきまといました。

```js
const handle = ...;
try {
  ... // handleを使ってもよい
}
finally {
  handle.close();
}
// handleは解放済みなのにまだアクセスできる
```

これは意図しないバグにつながる危険性があるため、処理系がスコープを抜けるとリソースが解放された状態になるのが望ましいです。
さらに、依存関係にあるリソースを間違った順番で解放しようとして解放に失敗するということがありえます。

これらの問題はこのProposalによって解決することができ、リソース周りのバグを予防して生産性を向上させる効果が見込めます。

# 意地悪してみる

ここからは、仕様書に書いてなかったり、仕様書からは読み取りにくい挙動について実際に動かして試してみます。

## グローバルなusing

処理系がスコープを抜けるとdiposeされるリソースですが、グローバル変数として書くとどうなるのでしょう？

```js
const g = () => ({
    [Symbol.dispose]() {
        console.log("dispose self");
    }
});

using disposable = g();
console.log("after using");
```

```output
after using
dispose self
```

NodeJSのプロセスが終了するときにリソースの解放が実行されているように見えます。
では、Ctrl + Cなどでプロセスをキルするとどうなるでしょう？

```js
const { setTimeout } = require("timers/promises");

const g = () => ({
    [Symbol.dispose]() {
        console.log("dispose self");
    }
});
(async () => {
  using disposable = g();
  console.log("after using");
  await setTimeout(10000);
  console.log("after 10000 ms");
})();
```

```output
after using
^C
```

流石に対応してないようです。まあ今までと同じように、NodeJSでは`process.on("SIGINT", () => { ... })`などを実装しないとプロセスのキルには対応できないということになります。

例外を投げるとどうでしょうか。

```js
const g = () => ({
    [Symbol.dispose]() {
        console.log("dispose self");
    }
});

using disposable = g();
console.log("after using");
throw Error("error");
```

```output
Successfully compiled 6 files with Babel (1192ms).
after using
dispose self
/Users/xxx/yyy/lib/global-error.js:8
function _dispose(stack, error, hasError) { function next() { if (0 !== stack.length) { var r = stack.pop(); if (r.a) return Promise.resolve(r.d.call(r.v)).then(next, err); try { r.d.call(r.v); } catch (e) { return err(e); } return next(); } if (hasError) throw error; } function err(e) { return error = hasError ? new dispose_SuppressedError(e, error) : e, hasError = !0, next(); } return next(); }
                                                                                                                                                                                                                                                                ^

Error: error
    at Object.<anonymous> (/Users/xxx/yyy/lib/global-error.js:22:9)
    at Module._compile (node:internal/modules/cjs/loader:1254:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1308:10)
    at Module.load (node:internal/modules/cjs/loader:1117:32)
    at Module._load (node:internal/modules/cjs/loader:958:12)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:81:12)
    at node:internal/main/run_main_module:23:47
```

リソースが解放されてからエラーになっていることがわかります。

まとめると、

- globalにusingを宣言してもプロセス終了時にリソースが解放される
- 例外でもリソースが解放される
- SIGINTなどのシグナルには反応しない

となります。

## disposableじゃないやつをusingする

disposableでないオブジェクト、より厳密には`Symbol.dispose`メソッドがないオブジェクトは`using`宣言のタイミングでエラーになります。`Symbol.asyncDispose`メソッドと`await using`宣言についても宣言時にエラーになります。

```js
const g = () => ({});

{
    console.log("disposable is not using");
    using disposable = g();
    console.log("disposable is using");
}
console.log("disposable is disposed");
```

```output
disposable is not using
/Users/xxx/yyy/lib/non-disposable.js:5
function _dispose(stack, error, hasError) { function next() { if (0 !== stack.length) { var r = stack.pop(); if (r.a) return Promise.resolve(r.d.call(r.v)).then(next, err); try { r.d.call(r.v); } catch (e) { return err(e); } return next(); } if (hasError) throw error; } function err(e) { return error = hasError ? new dispose_SuppressedError(e, error) : e, hasError = !0, next(); } return next(); }
                                                                                                                                                                                                                                                                ^

TypeError: Property [Symbol.dispose] is not a function.
    at _using (/Users/xxx/yyy/lib/non-disposable.js:6:424)
    at Object.<anonymous> (/Users/xxx/yyy/lib/non-disposable.js:13:20)
    at Module._compile (node:internal/modules/cjs/loader:1254:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1308:10)
    at Module.load (node:internal/modules/cjs/loader:1117:32)
    at Module._load (node:internal/modules/cjs/loader:958:12)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:81:12)
    at node:internal/main/run_main_module:23:47
```

## usingした変数を延命する

`using`宣言した変数は処理系がスコープを抜けると`Symbol.dispose`メソッドが呼ばれますが、何らかの手段で変数の延命をするとどうなるのでしょうか？
具体的にはusingした変数をreturnするとどうなるでしょうか。やってみましょう。

```js
const g = () => ({
    [Symbol.dispose]() {
        console.log("dispose g");
    }
});

const innerFunc = () => {
    using disposable = g();
    console.log("after using");
    return disposable;
};

using outerDisposable = innerFunc();
console.log("outerDisposable using");
```

```output
after using
dispose g
outerDisposable using
dispose g
```

なんと`Symbol.dispose`が2回呼ばれました。どうやらreturn文でもスコープを抜けた判定になるようです。
実際のコードでこれをやると、解放されたリソースをもう一回解放する不正なコードになるため、望ましくありません。

この問題を回避するためには、`innerFunc`で`using`を使うのをやめるか、1回しか実行されないような関数にするとよいと考えられます。（他にもあるかもしれません。）
内側だけ`using`を使うと内側で解放されるので、一番外側で`using`を使うというふうにしたほうが良さそうです。

```js
const g = () => ({
    [Symbol.dispose]() {
        console.log("dispose g");
    }
});

const innerFunc = () => {
    const disposable = g(); // constにした
    console.log("after using");
    return disposable;
};

using outerDisposable = innerFunc();
console.log("outerDisposable using");
```

```output
after using
outerDisposable using
dispose g
```

## disposable stackやusingを循環させてみる

DisposableStackが意図しない形で循環するとどうなるでしょうか？

```js
const g = (name) => ({
    [Symbol.dispose]() {
        console.log("dispose", name);
    }
});

const stack1 = new DisposableStack();
const stack2 = new DisposableStack();
stack1.use(g("stack1"));
stack2.use(stack1);
stack2.use(g("stack2"));
stack1.use(stack2);

stack1[Symbol.dispose]();
```

```output
dispose stack2
dispose stack1
```

無限ループはしませんでした。実装の詳細に依存する可能性はありますが、`DisposableStack`は依存するstackが既に解放されているかをマークし、解放されたものは再解放しないようになっているようです。

もっと意地悪してみます。

```js
const g = (name) => ({
    [Symbol.dispose]() {
        console.log("dispose", name);
        (this.dep || (() => {}))[Symbol.dispose]();
    }
});

using g1 = g("first");
using g2 = g("second");
g1.dep = g2;
g2.dep = g1;
```

```output
...（多量のログ）...
dispose first
dispose second
dispose first
dispose second
dispose first
dispose second
dispose first
dispose second
dispose first
dispose second
/Users/xxx/yyy/lib/mutual-dispose.js:8
function _dispose(stack, error, hasError) { function next() { if (0 !== stack.length) { var r = stack.pop(); if (r.a) return Promise.resolve(r.d.call(r.v)).then(next, err); try { r.d.call(r.v); } catch (e) { return err(e); } return next(); } if (hasError) throw error; } function err(e) { return error = hasError ? new dispose_SuppressedError(e, error) : e, hasError = !0, next(); } return next(); }
                                                                                                                                                                                                                                                                ^

SuppressedError
    at new SuppressedError (/Users/xxx/yyy/node_modules/suppressed-error/implementation.js:14:10)
    at new dispose_SuppressedError (/Users/xxx/yyy/lib/mutual-dispose.js:7:466)
    at err (/Users/xxx/yyy/lib/mutual-dispose.js:8:316)
    at next (/Users/xxx/yyy/lib/mutual-dispose.js:8:216)
    at err (/Users/xxx/yyy/lib/mutual-dispose.js:8:374)
    at next (/Users/xxx/yyy/lib/mutual-dispose.js:8:216)
    at _dispose (/Users/xxx/yyy/lib/mutual-dispose.js:8:391)
    at Object.<anonymous> (/Users/xxx/yyy/lib/mutual-dispose.js:27:3)
    at Module._compile (node:internal/modules/cjs/loader:1254:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1308:10)
```

予想通り無限ループになりました。リソースの解放に応じて他のリソースも解放したい場合、自前で書くのではなく、`DisposableStack`に任せるほうが良さそうです。

# Reactでの挙動

フロントエンドエンジニアとして気になるのは、Reactでの挙動です。
大体予想はつくのですが、Reactの関数コンポーネント内でusingを行った場合、どうなるのかを見てみましょう。

Babelを用いて、using構文をReact内で使用してみます。

（フルのコードは[こちら](https://github.com/nkowne63/ts52-using-react)です。）

```jsx
import React, { useState } from "react";

Symbol.dispose = Symbol.for("Symbol.dispose");

const rgen = (count) => ({
  [Symbol.dispose]() {
    console.log("disposed", count);
  }
});

function App() {
  const [count, setCount] = useState(0);
  using r = rgen(count);
  console.log("count", count);
  return (
    <div className="App">
      <header className="App-header">
        sample app
      </header>
      <button onClick={() => setCount(count => count + 1)}>count: {count}</button>
    </div>
  );
}

export default App;
```

これを実行してボタンを何回か押してみると以下のようなログが出力されます。

```log
App.js:14 count 0
App.js:7 disposed 0
App.js:14 count 1
App.js:7 disposed 1
App.js:14 count 2
App.js:7 disposed 2
App.js:14 count 3
App.js:7 disposed 3
```

レンダリングが走るごとにリソースの生成と解放処理が行われていることが観察できます。
なので、今まで`useEffect`フックで行われていたような、外部のリソースに対するアクセスとそのクリーンアップを`using`で代替するのは不適切と考えられます。むしろ普通の変数宣言として扱ってよいでしょう。

# まとめ

この記事では、Explicit Resource Management Proposalの動作の説明と挙動を一通り解説しました。実際に使う際の参考になれば幸いです。

なお、再度注意ですが、この記事でのusing宣言の動作はBabelのtransform及びes-shimsのpolyfill実装に依存しており、実際のV8エンジンやTypeScriptのトランスパイル出力の挙動とは異なる可能性があります。
以下の挙動がusing宣言に対応している処理系の実際の挙動と異なる場合はコメントをいただけると幸いです。