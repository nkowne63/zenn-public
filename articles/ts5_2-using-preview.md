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

この記事ではTypeScript 5.2のiteration planに先回りして、実際にusingがどのような挙動なのかをいじってみます。

# using宣言の例と動作

# DisposableStack

# using宣言の効用

# 意地悪してみる

# Reactでの挙動

<!-- 
https://github.com/tc39/proposal-explicit-resource-management

https://babeljs.io/docs/babel-plugin-proposal-explicit-resource-management
 -->