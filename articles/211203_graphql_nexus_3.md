---
title: "Apollo Server + Nexus + PrismaでGraphQL開発: 基本的なGraphQL APIを作る"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "TypeScript", "Nexus", "Prisma"]
published: true
---
この記事は、[いかずちさんだー Advent Calendar](https://adventar.org/calendars/7111) 3日目の記事です。

# リクルーティング表記

株式会社toridoriでは、エンジニア採用を行っています。
興味がある方はぜひぼくのTwitterにDMください！　リファラルボーナスがお互いに発生します。

https://toridori.co.jp/

https://twitter.com/Ikazuchis_diary

# 趣旨

[前回の記事](https://zenn.dev/ikazuchi/articles/f013a49b0ac75d)では、Apollo Server + Nexus + PrismaでGraphQLの開発の準備を行うところまでを紹介しました。
今回の記事では、コードを発展させ、実際にGraphQLサーバとして実用的に動作するようになることを目指します。

# 前提

今回は、Relay的なGUIDやページングは考慮せず、とりあえずDBのデータをきれいに取得できる状態を目標とします。

## RDBの仮定

RDBのテーブル構造を、Prismaのスキーマを用いて以下のように仮定することにします。

```text:prisma/schema.prisma
model User {
  id        BigInt   @id @default(autoincrement())
  name      String
  email     String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
}

model Post {
  id        BigInt   @id @default(autoincrement())
  title     String
  body      String
  authorId  BigInt
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  author    User     @relation(fields: [authorId], references: [id])
}
```

なお、MySQLからテーブル構造取得したとき、多くの場合でフィールド名がsnake_caseになっているので、@mapを使ってcamelCaseに変換することになると思います。
このとき、`prisma.prisma`拡張機能の効果で、VSCodeのF2キーを用いた変数名変更を行うことで@mapが自動的に生成されます。
巨大な場合は適切なスクリプト等を作成して変換するのがよく、実際にGitHub等にいくつか公開されていますが、想定通りきれいに動作するものを見かけたことがありません…。

# 実装

サンプルコード、ほとんどCoplilotが書いたので間違ってたらごめん…。

## 型の設定

今回、Prisma上でBigInt型やDateTime型を利用しているので、それに対応するための設定を書きます。
まず、`npm i graphql-scalars`を実行して、`graphql-scalars`をインストールします。
これを利用して、schema.tsを編集しましょう。

```diff typescript:src/schema.ts
export const schema = makeSchema({
  ...(略),
  types: [
    AllTypes,
+   asNexusMethod(GraphQLBigInt, 'bigint', 'bigint'),
+   asNexusMethod(GraphQLDateTime, 'datetime', 'Date'),
  ],
})
```

これで、Nexusの型定義でBigInt型やDateTime型を扱えるようになります。

## objectの定義

### nexus-prismaを利用する場合

nexus-prismaの効果で、`prisma.schema`の型情報を利用することができます。
以下のように記述するだけで、relationも自動的に処理してくれます。

```typescript:src/schema/object/user.ts
export const user = objectType({
  name: 'User',
  definition(t) {
    t.field(User.id)
    t.field(User.name)
    t.field(User.email)
    t.field(User.createdAt)
    t.field(User.updatedAt)
    t.field(User.posts)
  },
})
```

```typescript:src/shema/object/post.ts
export const post = objectType({
  name: 'Post',
  definition(t) {
    t.field(Post.id)
    t.field(Post.title)
    t.field(Post.body)
    t.field(Post.createdAt)
    t.field(Post.updatedAt)
    t.field(Post.author)
  },
})
```

### nexus-prismaを利用しない場合

サボらずに自力で書く場合はこんな感じです。
リレーションを処理するresolverで、データベースからそのユーザの記事一覧を引く処理を書きます。

```typescript:src/schema/object/user.ts
export const user = objectType({
  name: 'User',
  definition(t) {
    t.nonNull.bigint('id')
    t.nonNull.string('name')
    t.nonNull.string('email')
    t.nonNull.dateTime('createdAt')
    t.nonNull.dateTime('updatedAt')
    t.list.field('posts', {
      type: 'Post',
      resolve(parent, _args, ctx) {
        return ctx.prisma.user.findUnique({where: { id: parent.id }}).posts()
      },
    })
  },
})
```

```typescript:src/schema/object/post.ts
export const post = objectType({
  name: 'Post',
  definition(t) {
    t.nonNull.bigint('id')
    t.nonNull.string('title')
    t.nonNull.string('body')
    t.nonNull.dateTime('createdAt')
    t.nonNull.dateTime('updatedAt')
    t.field('author', {
      type: 'User',
      resolve(parent, _args, ctx) {
        return ctx.prisma.post.findUnique({where: { id: parent.id }}).author()
      },
    })
  },
})
```

## queryの定義

次に、userを取得するためのクエリを定義します。

```typescript:src/schema/query/user.ts
export const user = queryField('user', {
  type: 'User',
  args: {
    userId: nonNull(arg({ type: 'bigint' })),
  },
  resolve(_parent, { id }, ctx) {
    return ctx.prisma.user.findUnique({where: { id }})
  },
})
```

postを検索するクエリもついでに定義してみます。

```typescript:src/schema/query/post.ts
export const searchPost = queryField('searchPost', {
  type: nonNull(list(nonNull('Post'))),
  args: {
    query: nonNull(arg({ type: 'string' })),
  },
  resolve(_parent, { query }, ctx) {
    return ctx.prisma.post.findMany({
      where: {
        OR: [
          { title: { contains: query }},
          { body: { contains: query }},
        ],
      },
    })
  },
})
```

こんなところでしょうか。

## mutationの定義

userの作成、更新、postの作成、削除くらい定義しておきましょう。

```typescript:src/schema/mutation/user.ts
export const createUser = mutationField('createUser', {
  type: 'User',
  args: {
    name: nonNull(arg({ type: 'string' })),
    email: nonNull(arg({ type: 'string' })),
  },
  resolve(_parent, { name, email }, ctx) {
    return ctx.prisma.user.create({ data: { name, email }})
  },
})

export const updateUser = mutationField('updateUser', {
  type: 'User',
  args: {
    id: nonNull(arg({ type: 'bigint' })),
    name: arg({ type: 'string' }),
    email: arg({ type: 'string' }),
  },
  resolve(_parent, { id, name, email }, ctx) {
    return ctx.prisma.user.update({
      where: { id: id },
      data: { name ?? undefined, email ?? undefined },
    })
  },
})
```

```typescript:src/schema/mutation/post.ts
export const createPost = mutationField('createPost', {
  type: 'Post',
  args: {
    title: nonNull(arg({ type: 'string' })),
    body: nonNull(arg({ type: 'string' })),
    authorId: nonNull(arg({ type: 'bigint' })),
  },
  resolve(_parent, { title, body, authorId }, ctx) {
    return ctx.prisma.post.create({ data: { title, body, authorId } })
  },
})

export const deletePost = mutationField('deletePost', {
  type: 'Post',
  args: {
    id: nonNull(arg({ type: 'bigint' })),
  },
  resolve(_parent, { id }, ctx) {
    return ctx.prisma.post.delete({ where: { id }})
  },
})
```

## schema定義の更新

フォルダ構成が変更されたので、それに合わせて`index.ts`を新しくします。

```typescript:src/schema/object/index.ts
export * from './user'
export * from './post'
```

```typescript:src/schema/query/index.ts
export * from './user'
export * from './post'
```

```typescript:src/schema/mutation/index.ts
export * from './user'
export * from './post'
```

```typescript:src/schema/index.ts
export * from './object'
export * from './query'
export * from './mutation'
```

# 実行してみる

これでF5などを押してテスト実行してみると、次のような`schema.graphql`が出力されます。

```graphql:src/generated/schema.graphql
type Query {
  user(userId: bigint!): User
  searchPost(query: String!): [Post!]
}

type Mutation {
  createUser(name: String!, email: String!): User
  updateUser(id: bigint!, name: String, email: String): User
  createPost(title: String!, body: String!, authorId: bigint!): Post
  deletePost(id: bigint!): Post
}

type User {
  id: bigint!
  name: String!
  email: String!
  createdAt: DateTime!
  updatedAt: DateTime!
  posts: [Post!]!
}

type Post {
  id: bigint!
  title: String!
  body: String!
  createdAt: DateTime!
  updatedAt: DateTime!
  author: User!
}
```

また、Apollo Studioが利用可能になります。

# おわりに

Nexusを用いた開発は、基本的にはこのようなコードを繰り返し書く作業になります。
ただし、現実的には新たなEnumを定義するとか、ページングの実装、バリデーションの実装などの必要もあるでしょう。
また、権限管理なども行いたくなると思います。
次回は、今回のような基本的な開発の枠を越えた要素について紹介していきたいと思います。