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

# @skip

# @include

# @deprecated

# @specifiedBy

# @defer

# @stream
