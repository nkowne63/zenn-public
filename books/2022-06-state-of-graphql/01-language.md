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
適用できる場所に指定できる場所は大まかに 2 種類に分かれており、GraphQL の spec では ExecutableDirectiveLocation と TypeSystemDirectiveLocation に分けられています。Apollo Docs では後者のみの directive を schema directive、前者を含むものを query directive と呼び分けています。

まず、TypeSystemDirectiveLocation (schema directive)は以下のものを指します。

- SCHEMA（query, mutation, subscription のうちトップレベルのスキーマ定義オブジェクトであるもの）
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

また、ExecutableDirectiveLocation (query directive) は以下のものを指します。

- QUERY（GraphQL クエリ）
- MUTATION（GraphQL ミューテーション）
- SUBSCRIPTION（GraphQL サブスクリプション）
- FIELD（GraphQL のクエリなどのフィールド）
- FRAGMENT_DEFINITION（フラグメント定義）
- FRAGMENT_SPREAD（フラグメント内のスプレッド）
- INLINE_FRAGMENT（インラインフラグメント）
- VARIABLE_DEFINITION（変数定義）

大体思いつける限り殆どの場所に Directive を指定できますね。
Apollo Docs では query directive は実行時に query document を変形するために実装を制限されているようです。

さらに、repeatable キーワードを用いることによって、以下のように同じフィールドに二回同じ directive を書けるようになります。

```graphql
directive @delegateField(name: String!) repeatable on OBJECT | INTERFACE

type Book @delegateField(name: "pageCount") @delegateField(name: "author") {
  id: ID!
}

extend type Book @delegateField(name: "index")
```

もちろん、schema.graphql に定義しただけで使えるようになるわけではなく、各フレームワーク、ライブラリごとのカスタムロジックの実装が必要になります。

## 参考

- [GraphQL: spec](https://spec.graphql.org/October2021/#sec-Type-System.Directives.Custom-Directives)
- [Qiita: GraphQL のディレクティブについて](https://qiita.com/koffee0522/items/bb623f974c418f5e15b0)
- [Apollo Docs: Creating schema directives](https://www.apollographql.com/docs/apollo-server/schema/creating-directives/)
- [Apollo Docs: GraphQL schema basics](https://www.apollographql.com/docs/apollo-server/schema/schema/)
- [Apollo Docs: Directives](https://www.apollographql.com/docs/apollo-server/schema/directives/)

# Custom Scalars

GraphQL では Int や Float, String や Boolean などの、普通のプログラミング言語にありがちなプリミティブ型が一通り揃っています。また、type を使ってそれらの直積型を定義することもできます。

一方で、実際にアプリケーションを作る上ではこれらのプリミティブ型に制約をつけたようなプリミティブ型が欲しくなることがしばしばあります。代表例としては Date がそれにあたるでしょう。Date を文字列で表現するにせよ、数値で表現するにせよ、GraphQL の SDL で Int や String とは違う扱いをしたくなることは想像できると思います。

GraphQL では、そのようなカスタムな Scalar を以下のように定義することができます。

```graphql
scalar MyCustomScalar
```

これだけです。これによって、GraphQL の Scalar が使える場所ではどこでも MyCustomScalar が使えるようになります。

一方で、もちろんこれだけでは実際に parse するときに問題が生じるため、これまた実際にフレームワークやライブラリに応じた実装を行う必要があります。

## 参考

- [Apollo Docs: Unions and interfaces](https://www.apollographql.com/docs/apollo-server/schema/unions-interfaces/)
- [Apollo Docs: Custom scalars](https://www.apollographql.com/docs/apollo-server/schema/custom-scalars/)

# Fragments

GraphQL ではクエリを書くことでクライアントからサーバーに対してリクエストを送るわけですが、その中で同じフィールドを繰り返し送ることになる場合がよくあります。

例えば以下のクエリでは、id, name, profilePic のフィールドが繰り返しになっています。

```graphql
query noFragments {
  user(id: 4) {
    friends(first: 10) {
      id
      name
      profilePic(size: 50)
    }
    mutualFriends(first: 10) {
      id
      name
      profilePic(size: 50)
    }
  }
}
```

こうしたフィールドの組は Fragment と呼ばれるものでくくりだすことができ、例えば以下のようになります。

```graphql
query withFragments {
  user(id: 4) {
    friends(first: 10) {
      ...friendFields
    }
    mutualFriends(first: 10) {
      ...friendFields
    }
  }
}

fragment friendFields on User {
  id
  name
  profilePic(size: 50)
}
```

JavaScript を書く人だとオブジェクトスプレッド構文をご存知だと思いますが、それと似たようなノリで展開してくれます。

また、Fragment は入れ子にすることができます。

```graphql
fragment friendFields on User {
  id
  name
  ...standardProfilePic
}
```

## Fragment と変数

Fragment の中で変数を使用する場合、その値は親のクエリから注入しなければなりません。

```graphql
query HeroComparison($first: Int = 3) {
  leftComparison: hero(episode: EMPIRE) {
    ...comparisonFields
  }
  rightComparison: hero(episode: JEDI) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  friendsConnection(first: $first) {
    totalCount
    edges {
      node {
        name
      }
    }
  }
}
```

GraphQL 本体の spec には Fragment 単体で変数を持てる機能はありません。この状況を解決するには今の所 Relay を用いて@arguments / @argumentsDefinitions といった独自ディレクティブを使用することが必要そうです。
また、参考のところにあるように Quramy さんが Apollo-Link でこのディレクティブを使えるようにするものを作成してくださっています。

## Inline Fragment

Fragment は名前付きのものだけでなく、インラインでフィールドを区別するために使用することもできます。

後に述べますが、例えば union によってどちらの型が返ってくるかわからないようなクエリでは、ありうる場合を以下のようにインラインで列挙する必要があります。

```graphql
query inlineFragmentTyping {
  profiles(handles: ["zuck", "coca-cola"]) {
    handle
    ... on User {
      friends {
        count
      }
    }
    ... on Page {
      likers {
        count
      }
    }
  }
}
```

また、例えば以下のように条件に応じてプロパティーを含めるかどうかといったディレクティブなどをグループに適用することができるようになります。

```graphql
query inlineFragmentNoType($expandedInfo: Boolean) {
  user(handle: "zuck") {
    id
    name
    ... @include(if: $expandedInfo) {
      firstName
      lastName
      birthday
    }
  }
}
```

Fragment の中でさらにこの構文を使うこともでき、その場合は Fragment Spread と呼ばれたりもします。

## 参考

- [Qiita: GraphQL の Fragment colocation と variables でモヤついている](https://qiita.com/Quramy/items/1f9431b42d95ebdc59a8)
- [GraphQL の Fragment についての話](https://zenn.dev/sjbworks/articles/0b34ce8aca6b72)
- [GraphQL: spec](https://spec.graphql.org/October2021/#sec-Language.Fragments)
- [GraphQL: Fragments](https://graphql.org/learn/queries/#fragments)
- [Apollo Docs: Fragments](https://www.apollographql.com/docs/react/data/fragments/)

# Unions and Interfaces

この 2 つは Survey では別々のものとして扱われていますが、一緒に説明されることが多いため、こちらでも一緒に説明することにします。また、先程の fragments を同時に使うことが多いです。

基本的なモチベーションとしては、どちらも一つの型で複数の型を代表したいようなときにこれらを使うことができます。

## Interfaces

名前の通り、型（type）が実装するべきプロパティーと値を宣言したものです。とはいえ、GraphQL では、普通の言語の Interfaces のようにメソッドに相当するものはなく、データのプロパティーだけです。
また、注意点として、interface で宣言したものは、それを implement する type でも必ず宣言する必要があります。

```graphql
interface Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
}

type Human implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
  starships: [Starship]
  totalCredits: Int
}

type Droid implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
  primaryFunction: String
}
```

実際にクエリするには、以下のように inline fragment を使うのが便利です。逆に言うと、特定の type にしかないプロパティーを inline fragment なしで直接記述しようとするとエラーになります。

```graphql
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    __typename
    name
    ... on Droid {
      primaryFunction
    }
  }
}
```

## Unions

Unions も一つの型で複数の型を代表できるという点では interfaces と非常に似ていますが、違う点もあります。
それは、必ずしも interface のように、複数の型同士で同じフィルドを持つ必要がない点です。
また、interface と違って、宣言時に union の型に含まれる型のリストをすべて列挙することも特徴です。

```graphql
union SearchResult = Book | Author

type Book {
  title: String!
}

type Author {
  name: String!
}
```

クエリでは、interface と同じように inline fragments を用いるのが便利です。

```graphql
query GetSearchResults {
  search(contains: "Shakespeare") {
    __typename
    ... on Book {
      title
    }
    ... on Author {
      name
    }
  }
}
```

## 参考

- [AWS AppSync: GraphQL の Interface と Union](https://docs.aws.amazon.com/ja_jp/appsync/latest/devguide/interfaces-and-unions.html)
- [Apollo Docs: Unions and interfaces](https://www.apollographql.com/docs/apollo-server/schema/unions-interfaces/)
- [GraphQL: Schema](https://graphql.org/learn/schema/)
