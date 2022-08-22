---
title: "Directives"
free: true
---

この章では GraphQL のディレクティブ（Directives）について簡素な解説をしていきます。

# そもそもディレクティブ（Directives）とは

ディレクティブ（Directive）は GraphQL のスキーマやオペレーションを修飾するものです。

後で述べますが意味、構文ともにデコレーターにかなり似ています。

ディレクティブは様々なものもあり、デフォルトで用意されているものもあれば、カスタムで定義することもできます。

カスタムディレクティブについては[Language](./01-language.md)の章で説明しているためそちらを参照ください。

この章で扱うディレクティブは前者の、デフォルト（GraphQL の spec）で用意されているもの、もしくはデフォルト入りが議論されているものが中心です。

まず、2021 年度の GraphQL Spec で定義されているものが以下の 4 つです。

- @skip
- @include
- @deprecated
- @specifiedBy

さらに、State of GraphQL Survey 2022 ではデフォルト入りが議論されているものとして以下の 2 つが存在します。

- @defer
- @stream

## 参考

- [GraphQL spec](https://spec.graphql.org/October2021/#sec-Type-System.Directives)
- [Apollo Docs: Directives](https://www.apollographql.com/docs/apollo-server/schema/directives/)
- [Qiita: GraphQL のディレクティブについて](https://qiita.com/koffee0522/items/bb623f974c418f5e15b0)
- [GraphQL: Improving Latency with @defer and @stream Directives](https://graphql.org/blog/2020-12-08-improving-latency-with-defer-and-stream-directives/)
- [GitHub: Spec edits for @defer/@stream](https://github.com/graphql/graphql-spec/pull/742)

# @skip / @include

@skip と@include ディレクティブは両方とも Boolean である if を引数に取るディレクティブで、フィールドやフラグメントスプレッド、インラインフラグメントに使うことができます。

@skip は引数の if が true のときに、@include は if が false のときに GraphQL がフィールドを返さなくなるという効果があります。

```graphql
query myQuery($someTest: Boolean!) {
  # $someTestがtrueのときはexperimentalFieldのときは飛ばされる。
  experimentalField @skip(if: $someTest)
}
```

## 両方指定したら？

両方はある意味で真逆の存在ですので、意地悪をすることができます。

```graphql
query myQuery($someTest: Boolean!) {
  experimentalField @include(if: $includeThis) @skip(if: $skipThis)
}
```

この場合何が起こるのかというと、skip が false かつ@include が true のとき「のみ」フィールドが帰ってくるべきである、と仕様に定められています。

両方に true を指定したり、両方に false を指定したりするとフィールドは返ってきません。

指定順によって変動が起こることもありません。

## 参考

- [GraphQL spec: 3.13.1 @skip](https://spec.graphql.org/October2021/#sec--skip)
- [GraphQL spec: 3.13.2 @include](https://spec.graphql.org/October2021/#sec--include)

# @deprecated

@deprecated はフィールドの定義と enum の値に指定して「このフィールドや値はもうサポートされてないよ/廃止されたよ」ということを通知するために使用するものです。

reason という引数に使用をやめるべき理由を書くこともでき、何も書かない場合は「No longer supported」であることも定められています。

GraphQL の仕様の例にもある通り、新しいフィールドへの誘導を書くこともできるようです。

```graphql
type ExampleType {
  newField: String
  oldField: String @deprecated(reason: "Use `newField`.")
}
```

## 参考

- [GraphQL spec: 3.13.3 @deprecated](https://spec.graphql.org/October2021/#sec--deprecated)

# @specifiedBy

# @defer

# @stream
