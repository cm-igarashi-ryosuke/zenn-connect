---
title: "ZennのシンタックスハイライターをPrism.jsからShikiに移行しています"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zenn", "prismjs", "shiki"]
published: false
publication_name: team_zenn
---

:::message
移行完了していないものの、検証はほぼ終わっており、年内に切りの良いところまでアウトプットしたかったので記事を書いています。
:::

タイトルの通り、Zennのシンタックスハイライターを [Prism.js](https://prismjs.com/) から [Shiki](https://shiki.matsu.io/) に移行しています。

シンタックスハイライターとは、Zennの記事などにコードブロックを記述したときに、そのコードブロックの言語にあわせて文法や変数などに色を付けるためのライブラリです。例えば以下のようなものです。

```javascript
console.log("Hello world");
```

Shiki への移行については、以前からご提案いただいていましたが、移行にかかる作業時間的なコストと得られるメリットを天秤にかけ、様子見をしておりました。また、Prism.jsは [v2を開発中](https://github.com/orgs/PrismJS/discussions/3531) ということで、そちらも様子を見ていました。

https://github.com/zenn-dev/zenn-community/issues/593

しかし先日、Xで @mizchi さんから「Moonbit言語のシンタックス定義が欲しい」というご要望をいただき、またそれに連なるように、他の言語についても対応を求める声を複数いただいたいたことから、優先度を上げてShikiへの移行検討をはじめました。

https://x.com/mizchi/status/2000544806716793084

## Prism.js と Shiki の比較
まず、両者の特徴やアーキテクチャの違いについて確認しました。

### Prism.js

- 軽量・高速を謳う
- パッケージサイズが小さい（コア部分が2kb、言語定義が1つあたり300~500byte）
- 独自の[言語定義](https://prismjs.com/extending.html)。JavaScriptで書かれ、正規表現でトークンを解析する。
- [プラグイン](https://prismjs.com/#plugins)も豊富
- CSRを主要用途とする（SSRでも利用可能）
- CSS Classを使う（CSSによるテーマ切り替えが可能）
- 文法解析は[エッジケースで非対応パターン](https://prismjs.com/known-failures)あり

### Shiki

- 言語定義にTextMate Grammarを使用
- Oniguruma（正規表現ライブラリ）による、厳密な文法解析
- HASTベースで解析
- SSRを主要用途とする（CSRでも利用可能）
- ESModulesであり、どのような環境でも利用可能
- CSS Classを使わない（style属性にcolorを指定する）

自分の理解では、Prism.jsはCSRを想定しており、厳密さより軽量・高速を重視している。ShikiはSSRを想定して厳密さを重視している。厳密さを実現するために、TextMate Grammar（TextMateというエディタが発祥の言語定義フォーマット）を使っている。VSCodeもこのフォーマットを使っている。TextMate Grammarは、Onigurumaという正規表現ライブラリの正規表現を前提としている。Onigurumaの豊富な表現により、より厳密な言語解析が可能となっている。

ところで、Onigurumaは開発が終了したとなっているのですが今後どうなるのでしょうか。

### ベンチマーク

ローカル環境で、[zenn-markdown-html](https://github.com/zenn-dev/zenn-editor/tree/canary/packages/zenn-markdown-html) を使って簡易的なベンチマークテストを実行しました。

| テストケース | Prism (0.2.11) | Shiki (LOCAL) | 差分 |
|---|---:|---:|---:|
| コードブロックなし | 0.56 ms | 0.57 ms | ≈同じ |
| コードブロック1個 (JavaScript) | 0.46 ms | 1.98 ms | 4.3x 遅い |
| コードブロック5個 (異なる言語) | 0.68 ms | 1.67 ms | 2.5x 遅い |
| diff モード | 0.48 ms | 0.72 ms | 1.5x 遅い |
| 大きなコードブロック (50行) | 0.87 ms | 4.53 ms | 5.2x 遅い |

速度については Prism.js に優位性があるものの、ミリ秒レベルなのでShikiも十分に速いです。

:::details ベンチマークのスクリプト
```javascript
/**
 * zenn-markdown-html ベンチマーク
 *
 * 使い方:
 *   # リリース版をテスト（package.json の version）
 *   pnpm bench
 *
 *   # ローカル版をテスト
 *   pnpm bench:local
 *
 * バージョン変更:
 *   package.json の zenn-markdown-html のバージョンを変更して pnpm install
 */

import { createRequire } from 'module';
const require = createRequire(import.meta.url);

const isLocal = process.argv.includes('--local');

// ローカル版またはインストール版を動的にインポート
// どちらも CommonJS なので require を使用
const modulePath = isLocal ? '../../lib/index.js' : 'zenn-markdown-html';
const mod = require(modulePath);
const markdownToHtml = mod.default || mod;

console.log(`\n📦 Testing: ${isLocal ? 'LOCAL' : 'npm'} version\n`);
console.log('='.repeat(60));

// ------------------------------------------------------------
// テストケース
// ------------------------------------------------------------

const testCases = {
  'コードブロックなし': `
# タイトル

これは段落です。**太字**と*イタリック*があります。

- リスト1
- リスト2
- リスト3

> 引用文
`.trim(),

  'コードブロック1個 (JavaScript)': `
# タイトル

\`\`\`javascript
console.log('hello');
const x = 1;
const y = 2;
\`\`\`
`.trim(),

  'コードブロック5個 (異なる言語)': `
# 記事タイトル

## JavaScript

\`\`\`javascript
console.log('hello');
\`\`\`

## TypeScript

\`\`\`typescript
const x: number = 1;
interface User { name: string; }
\`\`\`

## Python

\`\`\`python
def hello():
    print("hello")
\`\`\`

## HTML

\`\`\`html
<div class="container">
  <p>Hello</p>
</div>
\`\`\`

## CSS

\`\`\`css
.container {
  display: flex;
  justify-content: center;
}
\`\`\`
`.trim(),

  'diff モード': `
# 変更点

\`\`\`javascript diff
-const old = 1;
+const new = 2;
 const unchanged = 3;
-removed();
+added();
\`\`\`
`.trim(),

  '大きなコードブロック (50行)': `
# Large Code

\`\`\`javascript
${Array.from({ length: 50 }, (_, i) => `const line${i} = ${i};`).join('\n')}
\`\`\`
`.trim(),
};

// ------------------------------------------------------------
// ベンチマーク実行
// ------------------------------------------------------------

async function runBenchmark(name, markdown, iterations = 10) {
  // ウォームアップ
  await markdownToHtml(markdown);

  const times = [];

  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    await markdownToHtml(markdown);
    const end = performance.now();
    times.push(end - start);
  }

  const avg = times.reduce((a, b) => a + b, 0) / times.length;
  const min = Math.min(...times);
  const max = Math.max(...times);

  console.log(`\n📝 ${name}`);
  console.log(`   平均: ${avg.toFixed(2)} ms`);
  console.log(`   最小: ${min.toFixed(2)} ms`);
  console.log(`   最大: ${max.toFixed(2)} ms`);

  return { name, avg, min, max };
}

// メイン実行
const results = [];

for (const [name, markdown] of Object.entries(testCases)) {
  const result = await runBenchmark(name, markdown);
  results.push(result);
}

// サマリー
console.log('\n' + '='.repeat(60));
console.log('📊 サマリー (平均時間)');
console.log('='.repeat(60));

for (const r of results) {
  const bar = '█'.repeat(Math.ceil(r.avg / 5));
  console.log(`${r.name.padEnd(35)} ${r.avg.toFixed(2).padStart(8)} ms ${bar}`);
}

console.log('\n');
```
:::

## 実装の移行

こちらがそのPull Requestです。

https://github.com/zenn-dev/zenn-editor/pull/611

もともと `diff` のハイライトを独自実装しており、それの移植による変更量が多いのですが、本筋のPrismからShikiへの移行自体はそれほど大変な作業ではありませんでした。

シンタックスハイライトは、MarkdownからHTMLに変換する処理の一部として行います。変換には [markdown-it](https://github.com/markdown-it/markdown-it) というライブラリを使っており、これのプラグインとして Shiki を差し込むようなイメージで実装されています。

実装でいくつか工夫した点を解説します。

### 遅延読み込み

Shikiのインスタンスはシングルトンパターンで生成し、はじめは言語を何も読み込まない状態で初期化することで、ロード時間を最小にします。

```typescript
  // 最初は空の言語セットで初期化（高速）
  highlighterInstance = await createHighlighter({
    themes: [SHIKI_THEME],
    langs: [],
  });
```

ハイライト処理を行う直前で、対象の言語がロードされているかをチェックして、未ロードの場合はロードします。

```typescript
/**
 * 言語がサポートされているかチェックし、必要に応じてロードする
 */
async function ensureLanguageLoaded(
  highlighter: Highlighter,
  langName: string
): Promise<boolean> {
  // 既にロード済みかチェック
  const loadedLangs = highlighter.getLoadedLanguages();
  if (loadedLangs.includes(langName)) {
    return true;
  }

  // bundledLanguages に含まれているかチェック
  if (langName in bundledLanguages) {
    await highlighter.loadLanguage(langName as BundledLanguage);
    return true;
  }

  return false;
}
```

### 非同期処理

Shikiのハイライト処理には非同期関数があります。Markdownの中に複数のコードブロックがある場合、同期処理よりも非同期処理のほうがパフォーマンスが良いはずなので、シンタックスハイライトは非同期処理でまとめて行っています。

具体的には、以下の処理を行います。

1. markdown-it がコードブロックを検出すると、コードブロック情報を一時情報として配列に保存し、HTMLにはプレースホルダーとして `<!--SHIKI_CODE_BLOCK_xxxxxxxx-->` を挿入する
2. 一時保存した全てのコードブロックを Promise.all でハイライト処理
3. プレースホルダーをハイライト済み HTML に置換

### Transformer

[Transformers](https://shiki.matsu.io/guide/transformers) という機能が便利でした。

ShikiはハイライトされたHTMLを生成する過程で、HAST（HTML用のAST）を使用します。HASTを操作することで、生成されるHTMLをカスタマイズすることができます。Prismaの時はHTMLをカスタマイズするために、生成されたHTMLに対して文字列一致をしてHTMLタグや属性を加えるなどしていたので、Transformerを使うことでより確実な実装にすることができました。

## マイグレーション計画

ZennではMarkdownから変換したHTMLをDBに保存しています。HTMLはPrismでハイライト済の状態なので、Shikiで再変換する必要がありますが、大量の記事がありますので変換に時間がかかり、この間、PrismとShikiの両方が共存する状態となります。

幸いにもPrismが生成するHTMLのCSSと、Shikiのstyleで競合する部分が無かったため、共存は問題ありませんでした。ゆっくり時間をかけて既存の記事の再変換を行う予定です。

## パフォーマンス検証

現在、MarkdownからHTMLへの変換処理（[zenn-markdown-html](https://github.com/zenn-dev/zenn-editor/tree/canary/packages/zenn-markdown-html)）は、Cloud Run Functions（第1世代）のHTTPサービスとして稼働しています。これをShikiバージョンにしてパフォーマンス検証を行いました。

**Prism版**
![](https://storage.googleapis.com/zenn-user-upload-integration/4d68fbffdf80-20251225.png)

**Shiki版**
![](https://storage.googleapis.com/zenn-user-upload-integration/ca0aec88bcc1-20251225.png)

どちらも20リクエスト/秒で1万件処理したときのメトリクスです。

- Prism版は、使用メモリが120MBほどで、20リクエスト/秒をさばけています。
- Shiki版は、使用メモリが512MB（設定値の最大）近くにまで達しており、3リクエスト/秒程度しかさばけていません。

実装上の問題の可能性もありますが、使用メモリは線形に上がってないのでメモリリークではなさそうな気がします。AIが言うにはOnigurumaがメモリをたくさん使うらしいですが、こんなに使うんでしょうか。

とりあえず深追いはせずに、メモリを調整したところ、2Giあれば十分そうということが分かりました。

もう一つ比較検証として、Cloud Run Functionsの第1世代と第2世代で比較を行いました。第1世代は、1つのインスタンスで同時に1リクエストを処理します。並列度を上げるには、たくさんのインスタンスを起動します。第2世代は、1つのインスタンスで複数のリクエストを処理します。シングルトンパターンで遅延ロードにしたShiki版は、第2世代の方がパフォーマンスメリットがありそうと考えました。

**Shiki版（第1世代）**
![](https://storage.googleapis.com/zenn-user-upload-integration/16dade20ee9c-20251225.png)

**Shiki版（第2世代）**

同時接続数: 20

![](https://storage.googleapis.com/zenn-user-upload-integration/d23d386b5d3b-20251225.png)
![](https://storage.googleapis.com/zenn-user-upload-integration/4706de466811-20251225.png)

第2世代については、同時接続数を変えて何度か試しましたが、いずれも第1世代の方がリクエストあたりの実行時間（レイテンシ）が小さい結果になりました。これは想定と違う結果でした。もう少し時間をかけて検証したいと思います。

## さらなる高速化

現在、オンラインエディタのプレビューも含め、すべてCloud Run Functionsで変換を行っています。今後、オンラインエディタのプレビューについては、ブラウザ環境で変換を行うようにすることで、レイテンシを改善したいと考えています。

それにしても[60fpsでの変換](https://zenn.dev/mizchi/articles/markdown-incremental-preview)は無理ですが 😂 （これは実際に触ってみて衝撃的な速さでした・・・！）

それでは皆さま良いお年を！
また来年もZennをよろしくお願いいたします。
