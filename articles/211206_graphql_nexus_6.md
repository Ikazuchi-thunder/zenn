---
title: "Apollo Server + Nexus + PrismaでGraphQL開発: InputObjectとバリデーション"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "TypeScript", "Nexus", "Prisma"]
published: true
---
この記事は、[いかずちさんだー Advent Calendar](https://adventar.org/calendars/7111) 6日目の記事です。

# リクルーティング表記

株式会社toridoriでは、エンジニア採用を行っています。
興味がある方はぜひぼくのTwitterにDMください！　リファラルボーナスがお互いに発生します。

https://toridori.co.jp/

https://twitter.com/Ikazuchis_diary

# 趣旨

[前回の記事](https://zenn.dev/ikazuchi/articles/211204_graphql_nexus_4)では、NexusでRelayのページングを実装するにはどうしたらいいかを解説しました。
今回の記事では、InputObjectとバリデーションを実装するための方法を説明します。
とはいえ、バリデーションについてはこれがベストなのかはかなり疑問があります…。

# InputObject

GraphQLのMutationでは、引数をInputObjectとしてひとつのオブジェクトにまとめてしまうのが一般的です。
どういう理由なのか明確に把握しているわけではないのですが、引数が多くなりすぎるのを防ぐためと考えればよいのかなと思っています。
昔のRelayの仕様ではInputObjectのフィールドとして`clientMutationId`というのを持たせて、サーバ側はそれをそのまま返すというのをやっていたみたいですが、現代的には不要そうです。

## 実装

以下のように、引数を外出ししてやります。
そんなに難しいことはないですが、フィールド単位でデフォルト値を設定できなくなることには留意しておきましょう。
まあ、対策方法とかはないですが…。

```typescript diff:src/mutation/user.ts
export const createUser = mutationField('createUser', {
  type: 'User',
  args: {
-   name: nonNull(arg({ type: 'string' })),
-   email: nonNull(arg({ type: 'string' })),
+   input: nonNull(arg({ type: 'CreateUserInput' })),
  },
- resolve(_parent, { name, email }, ctx) {
+ resolve(_parent, { input }, ctx) {
-   return ctx.prisma.user.create({ data: { name, email }})
+   return ctx.prisma.user.create({ data: { ...input }})
  },
})

export const createUserInput = inputObjectType({
  name: 'CreateUserInput',
  definition(t) {
    t.nonNull.string('name')
    t.nonNUll.string('email')
  },
})
```

# nexus-validate

Nexusでバリデーションを実装する方法はいろいろありますが、今回は、[nexus-validate](https://www.npmjs.com/package/nexus-validate)を使っていこうと思います。
これを用いることで、Mutationの定義の際にvalidate関数を実装できるようになります。

## 導入

`npm i nexus-validate yup`として導入したのちに、`src/schema.ts`に設定を追加します。

```typescript:src/schema.ts
const schema = makeSchema({
  ...
  plugins: [
    ...
    validatePlugin()
  ],
})
```

## 実装

以前に定義した`createUser` mutationにバリデーションを追加しましょう。

```typescript diff:src/mutation/user.ts
export const createUser = mutationField('createUser', {
  type: 'User',
  args: {
    input: nonNull(arg({ type: 'CreateUserInput' })),
  },
+ validate: ({ string }, { input }, ctx) => {
+   string().required('name is required').min(3, 'name is too short').validate(input.name)
+   string().required('email is required').email('email is invalid').validate(input.email)
+ },
  resolve(_parent, { name, email }, ctx) {
    return ctx.prisma.user.create({ data: { name, email }})
  },
})
```

これで、名前は3文字以上、メールアドレスは有効なメールアドレスであるという条件のバリデーションを追加できました。
今回はDBを参照していないのでcontextを使っていませんが、必要に応じてvalidate関数内からcontextを利用することができます。
また、今回はnexus-validateの機能でしかバリデーション処理をしていませんが、

```typescript diff:src/mutation/user.ts
    if(input.name.length < 3) throw new UserInputError('name is too short', { invalidArgs: ['name'] })
```

などと、より柔軟にバリデーションを記述することもできます。

# おわりに

今回は、InputObjectとバリデーションの追加によって、Mutationの肉付けを行いました。
次回は、認証・認可周りについて説明していこうと思います。