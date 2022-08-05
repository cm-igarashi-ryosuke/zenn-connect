---
title: Automatic Static Optimizationされたページでのnext/routerの注意事項について
emoji: "🫥"
type: "tech"
topics: [nextjs]
published: true
--- 

## 記事の概要

この記事は、[Automatic Static Optimization](https://nextjs.org/docs/advanced-features/automatic-static-optimization)されたページで、[next/router](https://nextjs.org/docs/api-reference/next/router) の `query` や `asPath` を使用する場合に注意が必要というお話です。内容は公式ドキュメントに書いてある通りなんですが、前提知識がない状態では理解が難しかったので、わかり易く解説しようと思いました。

Next.jsのバージョンは、現時点の最新である12.2です。

## 対象読者

Next.jsを使い始めたばかりで、公式のドキュメントを読んでも知識が足りなくて内容を理解できないという人に向けて書きます。

## キーワード

- Automatic Static Optimization
- Pre-rendering
- Hydrate

## Automatic Static Optimization とは

[Automatic Static Optimization](https://nextjs.org/docs/advanced-features/automatic-static-optimization)とは、ビルド時に、サーバー側の処理を伴わない（`getServerSideProps` や `getInitialProps` が存在しない）ページをhtmlファイルとして生成し、そうでないページをjsファイルとして生成する、という機能です。アプリケーション全体でビルドするのではなく、**ページ単位で最適なビルドにしてくれる**ということです。ページ単位で最適なビルドにすることで、サーバー側の処理を伴わないページは既にレンダリングが可能な状態のhtmlを返すことができるので、応答速度が上がります。ここで生成されるhtmlは、後述するPre-renderingと呼ばれる状態のhtmlになります。

## Pre-rendering とは

[Pre-rendering](https://nextjs.org/docs/basic-features/pages#pre-rendering)とは、クライアント側のjsでhtmlを全て生成するのではなく、予めレンダリングが可能なhtmlを返すことを指します。サーバー側の処理を伴わないページではビルド時に生成されたhtmlが、そうでないページではサーバー側が動的にPre-renderingしたhtmlを返します。

## Hydrate とは

[Hydrate](https://ja.reactjs.org/docs/react-dom.html#hydrate)とは、Next.jsで言うところのPre-renderingされたhtmlに、Reactのイベントリスナーをアタッチすることを指します。Pre-renderingされたhtmlの各要素には、Reactの仮想DOMとマッピングするためのidが振られています。

## next/routerの注意事項について

ここまでの前提知識があれば、あとはもう簡単です。

### 1. Static Optimizationで生成されたhtml（Pre-rendering）では `query` は `{}` になる

Automatic Static Optimizationはビルド時に行われる処理でした。ビルド時点では、QueryStringはありませんので `{}` になります。

`query` を参照するには、クライアント側で初回（Pre-rendering）のレンダリングが終わってから参照する必要があります。これは、`useEffect` で `router.isReady` の変化を検出して判定することができます。

### 2. Static Optimizationで生成されたhtml（Pre-rendering）に `asPath` を埋め込んではいけない

`asPath` はQueryStringを含むpathですので、 `query` と同様のことが言えます。

Automatic Static Optimizationされた時点ではQueryStringはありませんので、Pre-renderingの時点で参照してDOMに埋め込んでしまうと、実際にリクエストされた時のパスと異なる文字列になってしまいます。

回避方法も`query`と同様で、`useEffect` で `router.isReady` の変化を検出して、`true` になってから参照するようにします。

## 検証

`create-next-app`でまっさらなNext.jsプロジェクトを作成して検証してみます。

```shell
npx create-next-app@latest --ts
```

### 1. Automatic Static Optimization の挙動

まずは初期状態でビルドしてみます。少し分かりにくいですが、結果の表示に `/` の左に `◯` がついています。`/`ページはビルドでhtmlが生成されたよということです。

```shell
npm run build

(途中省略)

Page                                       Size     First Load JS
┌ ○ /                                      5.42 kB        83.1 kB
├   └ css/ae0e3e027412e072.css             707 B
├   /_app                                  0 B            77.7 kB
├ ○ /404                                   186 B          77.8 kB
└ λ /api/hello                             0 B            77.7 kB
+ First Load JS shared by all              77.7 kB
  ├ chunks/framework-db825bd0b4ae01ef.js   45.7 kB
  ├ chunks/main-f0e16f48d3775e5e.js        30.7 kB
  ├ chunks/pages/_app-deb173bd80cbaa92.js  499 B
  ├ chunks/webpack-7ee66019f7f6d30f.js     755 B
  └ css/ab44ce7add5c3d11.css               247 B

λ  (Server)  server-side renders at runtime (uses getInitialProps or getServerSideProps)
○  (Static)  automatically rendered as static HTML (uses no initial props)
```

次に `getInitialProps` を追加してビルドしてみます。

```typescript
// なにもしない getInitialProps
Home.getInitialProps = () => {
  return {};
}
```

今度は`/`の左に`λ`が表示されました。`/`ページはビルドでjsが生成されたよということです。

```shell
npm run build

(途中省略)

Page                                       Size     First Load JS
┌ λ /                                      5.43 kB        83.1 kB
├   └ css/ae0e3e027412e072.css             707 B
├   /_app                                  0 B            77.7 kB
├ ○ /404                                   186 B          77.8 kB
└ λ /api/hello                             0 B            77.7 kB
+ First Load JS shared by all              77.7 kB
  ├ chunks/framework-db825bd0b4ae01ef.js   45.7 kB
  ├ chunks/main-f0e16f48d3775e5e.js        30.7 kB
  ├ chunks/pages/_app-deb173bd80cbaa92.js  499 B
  ├ chunks/webpack-7ee66019f7f6d30f.js     755 B
  └ css/ab44ce7add5c3d11.css               247 B

λ  (Server)  server-side renders at runtime (uses getInitialProps or getServerSideProps)
○  (Static)  automatically rendered as static HTML (uses no initial props)
```

### 2. StaticでビルドされたhtmlにqueryやasPathを埋め込むとどうなるか

今度はこのように、`getInitialProps`なしで`query`や`asPath`の値を取り出してみます。

```typescript
import { useRouter } from 'next/router'

const Home: NextPage = () => {
  const { query, asPath } = useRouter()

  return (
    <div className={styles.container}>
      <p>q: {query.q}</p>
      <p>asPath: {asPath}</p>
    </div>
  )
}
```

ビルドされたhtmlにはこのようになります。（読みやすさのためフォーマットしてます）

```html
<body>
    <div id="__next">
        <div class="Home_container__bCOhY">
            <p>q: </p>
            <p>asPath:
                <!-- -->/
            </p>
        </div>
    </div>
</body>
```

この状態で、`http://localhost:3000/?q=hello` のようにQueryStringをつけてアクセスすると、`Error: Text content does not match server-rendered HTML.` といったエラーが発生します。

これを回避するためには、`useEffect`を使って、`isReady`が`true`になってから値を取り出すようにすれば良いです。

```typescript
const Home: NextPage = () => {
  const { isReady, query, asPath } = useRouter()
  const [q, setQ] = useState<string>('')
  const [currentPath, setCurrentPath] = useState<string>('')

  useEffect(() => {
    if (!isReady) return;
    setQ(query.q as string)
    setCurrentPath(asPath)
  }, [isReady])

  return (
    <div className={styles.container}>
      <p>q: {q}</p>
      <p>asPath: {currentPath}</p>
    </div>
  )
}
```

`console.log({ isReady, query, asPath })`で確認しても、`isReady`が`false`の間は、`query`は`{}`であることが確認できます。

### 3. ServerでビルドされたページにqueryやasPathを埋め込むとどうなるか

`getInitialProps`があるページでは、Pre-renderingの時点でQueryStringの情報は取得できますから、これで問題ありません。

```typescript
const Home: NextPage = () => {
  const { isReady, query, asPath } = useRouter()

  return (
    <div className={styles.container}>
      <p>q: {query.q}</p>
      <p>asPath: {asPath}</p>
    </div>
  )
}

// なにもしない getInitialProps
Home.getInitialProps = () => {
  return {};
}
```

`console.log({ isReady, query, asPath })`で確認しても、`isReady`がはじめから`true`であることが確認できます。

どちらの方法を選ぶかは、サーバー側の必要性で選択するのが良いかと思います。

## おわりに

対象読者は昨日の私です。内容に誤り等がありましたら教えて下さい。