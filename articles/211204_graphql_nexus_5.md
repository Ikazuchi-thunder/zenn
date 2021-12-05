---
title: "Apollo Server + Nexus + PrismaでGraphQL開発: Relayに従う2"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "TypeScript", "Nexus", "Prisma"]
published: true
---
この記事は、[いかずちさんだー Advent Calendar](https://adventar.org/calendars/7111) 5日目の記事です。

# リクルーティング表記

株式会社toridoriでは、エンジニア採用を行っています。
興味がある方はぜひぼくのTwitterにDMください！　リファラルボーナスがお互いに発生します。

https://toridori.co.jp/

https://twitter.com/Ikazuchis_diary

# 趣旨

[前回の記事](https://zenn.dev/ikazuchi/articles/211204_graphql_nexus_4)では、NexusでRelayのGUID/Nodeに従うためにどうしたらいいかの説明をしました。
今回の記事では、Relayのページングについて解説したいと思います。
お酒飲みすぎてつらいのできょうは手短に…。

# Relayのページング

Relayのページングシステムでは、以下の引数を持ちます。

- `first`
- `after`
- `last`
- `before`

このうち、`first`と`after`は、「`after`で指定したIDのノードの次のノードから、`first`で指定した数のノードを返す」という意味です。
また、`last`と`before`は、「`before`で指定したIDのノードの前のノードから、`last`で指定した数のノードを（手前方向に）返す」という意味です。
実用的に`last`と`before`が何に使われているのかはよく知らないのですが（手前方向にページングしたくなること、ある？）、慣例的に実装されているようなので、実装します。

## prisma-relay-cursor-connection

世の中にはPrismaでRelayのページングを実装した、prisma-relay-cursor-connectionというライブラリがあるので、それを利用しましょう。
`npm i @devoxa/prisma-relay-cursor-connection`として、インストールを行います。
また、relayなページングを実装するために、`src/schema.ts`を以下のように編集します。

```typescript:src/schema.ts
plugins: {
  connectionPlugin({
    extenedConnection: {
      totalCount: { type: 'Int', requireResolver: false },
    }
  })
}
```

ここでは、ついでに、`totalCount`というカラムを追加しています。
これは、ページングを考慮せずにすべてのノードの数を返すためのカラムです。

## Queryにprisma-relay-cursor-connectionを適用する

2つ前の記事で紹介した記事のPostのQueryに、prisma-relay-cursor-connectionを適用します。

```typescript:src/query/post.ts
export const searchPosts = extendType({
  type: 'Query',
  definition(t) {
    t.connectionField('posts', {
      type: 'Post',
      additionalArgs: {
        search: stringArg({ nullable: true }),
      },
      resolve(_root, args, ctx){
        const baseArgs = {
          where: {
            OR: [
              { title_contains: args.search },
              { content_contains: args.search },
            ],
          },
        };
        return findManyCursorConnection<Campaign, {databaseId: bigint}>(
          (_args) => ctx.prisma.post.findMany({..._args, ...baseArgs}),
          () => ctx.prisma.post.count(baseArgs),
          { ...args },
          {
            getCursor: (record) => ({ databaseId: record.databaseId }),
            encodeCursor: (cursor) => Buffer.from('Post:' + cursor.databaseId).toString('base64'),
            decodeCursor: (cursor) => ({databaseId: Buffer.from(cursor, 'base64').toString().split(':')[1]}),
          }
        )
      }
    })
  }
})
```

だいぶ複雑ですが、以下のような感じです。

- additionalArgs: first, last等以外の引数の指定です。
- baseArgs: additionalArgsで`where`や`orderBy`を実現します。
- getCursor: ページングのためのIDを指定します。databaseIdでいいと思います。
- encodeCursor: ページングのためのカーソルを指定します。我々にはGUIDという武器があるので、それでいいでしょう。
- decodeCursor: ページングのためのカーソルのデコード方法を指定します。GUIDからdatabaseIdを取得すればよいでしょう。

### 注意点

以上のようにすることでページングはうまく実装できるのですが、実際に発行されるSQLは確認した方がいいでしょう。
特に`orderBy`をindexではないカラムに指定した場合、速度面に不安があります。
2つ前の記事で指定した、以下の指定によって標準出力にSQLが出力されます。

```typescript:src/context.ts
prisma.$on('query', (e) => {
  console.log(e)
})
```

# 何がうれしいのか

正直サーバ側ではよくわからないのですが、これに従うことでApollo ClientやRelayがクライアントサイドでうまいことしてくれみたいです。
クエリはもちろん、オブジェクトのResolverでも大量のデータが返ってくるリスクがある場合はページングを作成しましょう。

# おわりに

きょうは大変短くて申し訳なかったですが、ページングについて解説しました。
次回はバリデーションなどのその他の要素について説明します。