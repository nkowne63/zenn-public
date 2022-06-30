---
title: "Language"
free: true
---

この章では、GraphQL の言語機能について簡単な解説をしていきます。

# Custom Directives

GraphQL の Custom Directive とは GraphQL の型やフィールド、引数にカスタムなロジックを加えるものです。
GraphQL には Directive と呼ばれる仕組みが存在していますが、それをプログラマーが定義できるものですね。
GraphQL サーバのスキーマ定義である schema.graphql で以下のように定義することができます。

```graphql
directive @deprecated(
  reason: String = "No longer supported"
) on FIELD_DEFINITION | ENUM_VALUE
```

このように、directive を定義する際に引数や適用できる場所を指定することができます。
適用できる場所に指定できる場所の一覧は以下のとおりです。

- SCALAR（Custom Scalar の定義）
- OBJECT（Object 型の定義）
- FIELD_DEFINITION（input type 以外のフィールド）
- ARGUMENT_DEFINITION（フィールドの引数）
- INTERFACE（Interface の定義）
- UNION（Union の定義）
- ENUM（Enum の定義）
- ENUM_VALUE（Enum の値の定義）
- INPUT_OBJECT（input type、mutation などの引数として現れたりする）
- INPUT_FIELD_DEFINITION（input type のフィールド）
- SCHEMA（トップレベルのスキーマ定義オブジェクト、GraphQL では query, mutation, subscription）

大体思いつける限り殆どの場所に Directive を指定できますね。

もちろん、schema.graphql に定義しただけで使えるようになるわけではなく、各フレームワーク、ライブラリごとのカスタムロジックの実装が必要になります。

## 参考

- [Qiita: GraphQL のディレクティブについて](https://qiita.com/koffee0522/items/bb623f974c418f5e15b0)
- [Apollo Docs: Creating schema directives](https://www.apollographql.com/docs/apollo-server/schema/creating-directives/)
- [Apollo Docs: GraphQL schema basics](https://www.apollographql.com/docs/apollo-server/schema/schema/)
- [Apollo Docs: Directives](https://www.apollographql.com/docs/apollo-server/schema/directives/)

# Custom Scalars

# Fragments

# Unions

# Interfaces
