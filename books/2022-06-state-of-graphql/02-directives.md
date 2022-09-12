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

@specifiedBy は custom scalar（1 章）の仕様がどこに書いてあるかを示すことができます。

たとえば以下のとおりです。

```gql
scalar UUID @specifiedBy(url: "https://tools.ietf.org/html/rfc4122")
```

注意してほしいのは、これはあくまでも「人間が読むための仕様書」であり、これをもとに実装されているかどうかは実装次第であるということです。過信しすぎないようにしましょう。

## 参考

- [GraphQL spec: 3.13.4 @specifiedBy](https://spec.graphql.org/October2021/#sec--specifiedBy)

# @defer / @stream について

@defer / @stream は GraphQL の 2021 年現在の仕様には定められていません。そのためこのセクションは GitHub の Pull Request をもとに書くことになります。

この仕様は公式の仕様には取り込まれていないため、実装ごとに挙動が異なったりこれから変更される可能性があります。ご注意ください。

## モチベーション

公式ブログによると、@defer / @stream は GraphQL のモデルの欠点を改善するためのものです。

その欠点とは、「リクエストのデータが全部取得されるまでレスポンスが返ってこない」というものです。

以下のように、非常にデータ取得に時間がかかる部分を含むクエリを考えてみましょう。

```gql
query ProfilePageQuery {
  person(id: "some-id") {
    ...personalProfile
    ...friendsList # ここの取得に5秒くらいかかる
  }
}
```

たとえば SNS のマイページに自分のプロファイルとフレンドのリストを表示したいとして、リストがとても長いとしましょう。すると自分のプロファイルの取得が一瞬で終わるのにも関わらずフレンドリストの取得が終わるまでクライアントでは何も表示されません。これはユーザービリティーを著しく下げてしまいます。

もちろんクエリを分割することもできますが、その場合は GraphQL の利点である、「1 回のクエリで必要なデータがすべて手に入る」という性質が殺されてしまいます。

そこで、GraphQL のクエリを分けずに、かつ、重い部分を分ける方法として@defer / @stream が考案されました。

## 挙動

この挙動は[GitHub: Spec edits for @defer/@stream](https://github.com/graphql/graphql-spec/pull/742)を参照に書いています。

以下のクエリを見てみましょう。

```gql
query {
  person(id: "cGVvcGxlOjE=") {
    ...HomeWorldFragment @defer(label: "homeWorldDefer")
    name
    films @stream(initialCount: 1, label: "filmsStream") {
      title
    }
  }
}
fragment HomeWorldFragment on Person {
  homeWorld {
    name
  }
}
```

@defer / @stream ではともに label が必要であり、if で実際に効果を発動させるかフラグを与えることができます。また、@stream では initialCount を与えることができます。

@defer はフラグメントスプレッドとインラインフラグメントに、@stream はフィールドに指定することができます。

さて、これを実行すると最初は以下のレスポンスが返ってきます。

```json
{
  "data": {
    "person": {
      "name": "Luke Skywalker",
      "films": [{ "title": "A New Hope" }]
    }
  },
  "hasNext": true
}
```

ディレクティブが指定されていない name フィールドと、@stream で initialCount: 1 を指定した films で 1 つだけオブジェクトが返ってきています。

一見普通の GraphQL レスポンスですが、「hasNext」というフィールドが true になっています。

そして、次のレスポンスが以下のようになっています。

```json
{
  "incremental": [
    {
      "label": "homeWorldDefer",
      "path": ["person"],
      "data": { "homeWorld": { "name": "Tatooine" } }
    },
    {
      "label": "filmsStream",
      "path": ["person", "films", 1],
      "items": [{ "title": "The Empire Strikes Back" }]
    }
  ],
  "hasNext": true
}
```

data フィールドではなく incremental フィールドにデータが入っています。label がもとのクエリで指定した値であり、path にはデータがもとのクエリのどの部分にあたるのかが格納されています。

3 つ目のレスポンスは以下の通りになっています。

```json
{
  "incremental": [
    {
      "label": "filmsStream",
      "path": ["person", "films", 2],
      "items": [{ "title": "Return of the Jedi" }]
    }
  ],
  "hasNext": true
}
```

@defer に由来する部分が消えていますね。
そして、最後のレスポンスが以下のようになります。

```json
{
  "hasNext": false
}
```

この「hasNext: false」をもってクライアントでは終了を判定することができます。

## 実装

さて、仕様で議論されていても実際に使えなければ意味がありません。しかも厄介なことに、GraphQL 仕様草稿では「実装しなくても良い（実装する場合は従ってね）」とされており、実装されていない場合は単に無視されてしいます。

ここでは、GraphQL Helix と Apollo Client の 2 つの実装を見てみます。

### GraphQL Helix

### Apollo Client

## 参考

- [GitHub: Spec edits for @defer/@stream](https://github.com/graphql/graphql-spec/pull/742)
- [GraphQL: Improving Latency with @defer and @stream Directives](https://graphql.org/blog/2020-12-08-improving-latency-with-defer-and-stream-directives/)
- [Zenn: GraphQL の @defer, @stream ディレクティブを試してみる](https://zenn.dev/adwd/articles/53c991e373f84d)
- [GraphQL Helix: Using the @defer and @stream directives](https://www.graphql-helix.com/docs/recipes/using-the-defer-and-stream-directives)
- [GitHub: Support for defer/stream](https://github.com/apollographql/apollo-client/issues/7691)
- [Apollo: Using the @defer directive in Apollo Client](https://www.apollographql.com/docs/react/data/defer/)
