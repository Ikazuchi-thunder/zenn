---
title: "Apollo Server + Nexus + PrismaでGraphQL開発: Relayに従う1"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "TypeScript", "Nexus", "Prisma"]
published: true
---
この記事は、[いかずちさんだー Advent Calendar](https://adventar.org/calendars/7111) 4日目の記事です。

# リクルーティング表記

株式会社toridoriでは、エンジニア採用を行っています。
興味がある方はぜひぼくのTwitterにDMください！　リファラルボーナスがお互いに発生します。

https://toridori.co.jp/

https://twitter.com/Ikazuchis_diary

# 趣旨

[前回の記事](https://zenn.dev/ikazuchi/articles/f013a49b0ac75d)では、Apollo Server + Nexus + Prismaで基本的なGraphQL APIが動作するところまでを解説しました。
今回と次回は、Relayの要求するパターンに基づいたGraphQLの作り方を解説します。

# Relay GraphQL Server Specification

RelayはMeta（Facebook）製のGraphQLクライアントフレームワークです。
このフレームワークには、サーバサイドに要求する規格があります。
その規格は、クライアントにRelayを用いない場合でも有用なので、どんどん従っていくのがよいと思います。

# IDとNode

Relayでは、オブジェクトはGraphQL全体で唯一のIDを持つのが良いとされています。
そのIDを持つオブジェクトを、Nodeインターフェースに従っているといいます。

idというフィールド名はこの統一ID（以下、GUIDと呼びます）に使うので、schema.prismaを多少変更しておきましょう。

```diff text:prisma/schema.prisma
- id         BigInt   @id @default(autoincrement())
+ databaseId BigInt   @id @default(autoincrement()) @map(id)
```

## Nodeインターフェースの定義

Nexusを用いて、Nodeインターフェースを定義します。
id型のidという名前のフィールドを持っていればOKです。

```typescript:src/schema/interface/node.ts
export interface node = interfaceType({
  name: 'Node',
  definition(t) {
    t.id('id')
  },
})
```

インターフェースの定義では、`resolveType`というメソッドの定義も要求されます。
これは、Nodeの実装であるオブジェクトの実態がどの型なのかを判断するためのメソッドです。
たとえば、以下のように実装します。

```typescript:src/schema/interface/node.ts
resolveType(data) {
  return 'name' in data ? 'User' : 'Post'
}
```

この判断のための方法は今行った`resolveType`メソッドの実装以外にもあります。
Nodeのために行うのでは、この方法ではなく、クエリの定義のときに`__typename`を指定する方法のほうが便利だと思います。
そのために、`src/schema.ts`を少し変更します。

```typescript diff:src/schema.ts
makeSchema({
  ...
+ features: {
+   abstractTypeStrategies: {
+     resolveType: true,
+     __typename: false,
    },
  },
  ...
})
```

https://nexusjs.org/docs/guides/abstract-types#picking-your-strategy-or-strategies

## ObjectをNodeインターフェースに準拠させる

Nexusであるオブジェクトがinterfaceの実装であることを示す際に、`t.implements()`を使います。

```typescript diff:src/schema/object/user.ts
export const User = objectType({
  name: 'User',
  definition(t) {
+   t.implements('Node')
-   t.field(User.id)
+   t.field(User.databaseId)
    t.field(User.name)
    t.field(User.email)
    t.field(User.createdAt)
    t.field(User.updatedAt)
    t.field(User.posts)

+   t.nonNull.id('id', {
+     resolve: (parent, _args, _ctx) => Buffer.from('User:' + parent.id).toString('base64'),
+   })
  },
})
```

ここではNodeインターフェースを`'Node'`と文字列で指定しています。
Nodeインターフェース実装後に一度実行していれば、Nexusが型生成を行ってくれているので、文字列によるインターフェースの指定でも型推論ができます。
また、もちろん前節で定義したnode変数を引数にすることもできます。

また、GUIDをどのような形式にするのかは議論の余地があります。
今回は、オブジェクト名とデータベースのIDを連結し、base64変換したものとしています。
公式ドキュメントにも、慣習的にbase64化されていると書かれているのでそこは決まりでいいと思うのですが、もとの文字列をなににするかというところです。

## Nodeクエリの実装

次に、IDからその実態を返すNodeクエリを実装します。
Nodeインターフェースの実装の際に、`__typename`をクエリの際に返す設定にしていることとします。

```typescript:src/schema/query/node.ts
export const node = queryField('node', {
  type: 'Node',
  args: {
    id: nonNull(stringArg()),
  },
  resolve: (_parent, { id }, ctx) => {
    const idStr = Buffer.from(id, 'base64').toString()
    const [type, databaseId] = idStr.split(':')
    if (type === 'User') {
      res = await ctx.prisma.user.findUnique({ where: { id: databaseId } })
      return res ? { ...res, __typename: 'User' as const } : null
    } 
    if (type === 'Post') {
      res = await ctx.prisma.post.findUnique({ where: { id: databaseId } })
      return res ? { ...res, __typename: 'Post' as const } : null
    }
    return null
  },
})
```

ここで気をつけるべきなのは、`__typename`はconst型で返すところです。
これを行わないと型チェックが通らなくて苦しむことになります。

# Nodeのメリット

多くのオブジェクトをNodeインターフェースに準拠させることで、Nodeクエリから様々なオブジェクトを取得できるようになります。
サーバ開発をするにあたっては、開発中のテスト等に便利に使うことになると思います。
クライアント的には出力されたものを中身によってUIを出し分けることもできます。

# おわりに

今回は、RelayのGUIDとNodeへの準拠について説明しました。
次回は、ページングについて解説しようと思います。