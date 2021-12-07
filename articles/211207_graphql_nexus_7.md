---
title: "Apollo Server + Nexus + PrismaでGraphQL開発: 認証と認可"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GraphQL", "TypeScript", "Nexus", "Prisma"]
published: true
---
この記事は、[いかずちさんだー Advent Calendar](https://adventar.org/calendars/7111) 7日目の記事です。

# リクルーティング表記

株式会社toridoriでは、エンジニア採用を行っています。
興味がある方はぜひぼくのTwitterにDMください！　リファラルボーナスがお互いに発生します。

https://toridori.co.jp/

https://twitter.com/Ikazuchis_diary

# 趣旨

[前回の記事](https://zenn.dev/ikazuchi/articles/211204_graphql_nexus_4)では、NexusでのInputObjectとバリデーションについて解説しました。
今回は、認証・認可をどのように実装していくかを考えていきます。

# 認証

認証は、GraphQL外に、RESTの口を作って行うのが定跡のようです。
別にGraphQL上にそのための口を作ってもいいのですが、Apolloのcontextで認可を行う際に、「認証用の口にアクセスするときだけトークンが不要だよ」というコードを書くのが面倒なのです。
今回は、firebase adminを使って認証を行います。

## /auth APIの作成

もはやRESTの書き方をここで解説するのはどうかとも思いますが、まずはAPIを生やしましょう。

```typescript:src/main.ts
  app.post('/auth', auth)
```

`src/auth.ts` に、認証の処理を実装しましょう。

```typescript:src/auth.ts
export const auth = async (req: Request, res: Response) => {
  if (req.body.idToken === undefined) {
    res.status(401).send('idToken is required')
    return
  }
  const ticket = await admin.auth().verifyIdToken(req.body.idToken)
  if(!ticket || !ticket.email) {
    res.status(401).send('invalid idToken')
    return
  }
  const user = ... // emailからDB引いてユーザ情報取得（べつにemailだけ返してもいい説もある）
  return res.json({
    token: sign(user, jwtSecret, { expiresIn: '3d' }),
  })
}
```

みたいな感じでよいと思います。
必要に応じて新規ユーザの生成などの処理も書くことになるかもしれません。
ただ、このコードのようにjwtを用いる場合は、tokenが漏れたときのことを考えてあんまり重要な情報は書かないようにしましょう。
jwtは偽造はできませんが解凍はできるので。

# 認可

認可は、認証の過程で作られたtokenが正しい物かどうかの確認と、それに紐つく権限情報の取得がポイントになります。
手順は、

1. tokenを解凍
2. そいつの持っている権限を取得
3. contextに格納
4. 各Object, Query, MutationのResolverに認証処理を書く

みたいな感じになります。

## tokenを解凍

```typescript:src/context.ts
const decodedToken = verify(token, jwtSecret) as decodedToken
```

みたいな感じでよいでしょう。
失敗時には`AuthenticationError`をthrowします。
この段階ではまだ認証なので403ではなく401です。

## 権限の取得・contextへの格納

`src/context.ts`内でDBを叩いてそのユーザの権限を取得します。
RESTの段階でemailを横流ししただけみたいな処理を書いている場合は、ここで新規ユーザとかの処理も書きましょう。
権限をどのように管理するかはプロジェクトに依存するかと思います。

また、ついでにユーザ情報も格納して、実はRelayで定められているviewerクエリを作るのもいいでしょう。
たとえば、次のようにviewerオブジェクトを定義します。

```typescript:src/object/viewer.ts
export const viewer = objectType({
  name: 'viewer',
  definition(t) {
    t.nonNull.string('name')
    t.nonNull.string('email')
    t.nonNull.list.nonNull.field('permissions', {
      type: 'Permission', // PermissionというEnumが定義されているとする
    })
  },
})
```

で、一度実行するとviewerオブジェクトの型情報が生成されるので、Contextを以下のように定義して返すようにします。

```typescript:src/context.ts
type Context = {
  prisma: PrismaClient
  viewer: NexusGenObjects['Viewer']
}

export const context = async ({req}: ExpressContext) => {
  ...
  // 雰囲気こんな感じ
  return {
    prisma,
    viewer: {
      name: user.name,
      email: user.email,
      permissions: user.permissions,
    },
  }
}
```

viewerクエリはシンプルに`ctx.viewer`を返すだけです。

```typescript:src/query/viewer.ts
export const viewer = queryField('viewer', {
  type: 'Viewer',
  resolve: (ctx) => ctx.viewer,
})
```

## 認証処理

認証処理は、`fieldAuthorizePlugin`を使って実装するのがよさそうです。

https://nexusjs.org/docs/plugins/field-authorize

`src/schema.ts`にその設定を書きます。

```typescript:src/schema.ts
const schema = makeSchema({
  ...
  plugins: [
    ...
    fieldAuthorizePlugin()
  ],
})
```

すると、（前回の記事でvalidationが生えたように、）フィールド定義の際にauthorizeを指定することができるようになります。

```typescript:src/query/post.ts
export const user = queryField('user', {
  type: 'User',
  args: {
    userId: nonNull(arg({ type: 'bigint' })),
  },
  authorize: (_parent, args, ctx) => {
    return permissionCheck(ctx.viewer.permissions, 'read:user' as NexusGenEnums['Permission'])
  },
  resolve(_parent, { id }, ctx) {
    return ctx.prisma.user.findUnique({where: { id }})
  },
})
```

なお、authorizeでは、

- true or Promise<true>を返すと認証成功
- false or Promise<false>を返すとNot Authorizedエラー
- その他のErrorを返したりthrowしたりすると、それがエラー

という挙動を示します。

# おわりに

今回は、認証・認可周りについて説明しました。
次回は、…ついにネタが切れてきたので、ないかもしれません。
またなにか思いついたら書きますね。