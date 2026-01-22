---
title: "Zennのプレビュー変換をブラウザ上で実行するようにして高速化しました"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zenn", "shiki"]
published: false
publication_name: team_zenn
---

現在、「[新しいエディタ](https://info.zenn.dev/2025-12-08-release-new-editor)」としてご提供している記事編集ページのエディタでは、2カラムレイアウトでプレビューがリアルタイムに確認できるようになっています。しかし、プレビューの生成がサーバー側で行われているため、編集内容が反映されるまでに一定のラグがありました。

そこで、プレビューの生成（MarkdownからHTMLへの変換）を行っている [zenn-markdown-html](https://www.npmjs.com/package/zenn-markdown-html) をブラウザ上で実行するように変更し、プレビュー反映の高速化を実現しました。

ちょっと前に、[60fpsでの変換](https://zenn.dev/mizchi/articles/markdown-incremental-preview)を見て、「Zennのエディタももっと速くしたい」と、感化されたというのもあります。

## CSP (Content Security Policy)によるwasmの実行制限

zenn-markdown-html はシンタックスハイライターとして [Shiki](https://shiki.style/) を使用していますが、Shikiは標準で、Onigurumaという正規表現エンジンのwasm(WebAssembly)を利用します。

しかし、Zennはユーザー投稿を扱うサービスであるため、セキュリティリスクを軽減するためにCSPを導入しています。これにより、Onigurumaのwasmの実行も制限されており、ブラウザ上でShikiを動作させることができませんでした。

https://zenn.dev/team_zenn/articles/introduced-csp-to-zenn

そこで、ブラウザ上で実行する際は、Shikiの方で用意されている[JavaScript版の正規表現エンジン](https://shiki.style/guide/regex-engines#javascript-regexp-engine)に切り替えて実行することで、CSPの制限を回避しました。ドキュメントによると、現時点では[すべての言語で完全な互換がある](https://shiki.style/references/engine-js-compat)そうです。

https://github.com/zenn-dev/zenn-editor/pull/628

## ベンチマーク

ブラウザ上での変換速度を確認するために、簡単なベンチマークを行いました。ブラウザ上での実行なので、パフォーマンスはマシンスペックに依存します。

スペック: Apple M4 / 32 GB

ベンチマークとして、3パターンのMarkdownを用意して、それぞれを100回変換を行い平均値を取りました。

| ケース | 説明 | 文字数 | 平均処理時間 | 出力HTML長 |
|--------|------|--------|-------------|-----------|
| 小規模 | 見出し + 段落 + リスト | 91 | 5.42 ms | 1,118 |
| 中規模 | コードブロック複数含む一般的な記事 | 1,578 | 6.25 ms | 12,328 |
| 大規模 | 複数言語のコードブロック + 長文 | 5,110 | 14.39 ms | 37,267 |

比較的大きな記事でも、15ms程度で変換できていることがわかります。

また、実際の[私の大きめな記事](https://zenn.dev/team_zenn/articles/zenn-markdown-editor-by-cm6)（コードブロックが15個ある）でも試してみましたが、30~40msで変換できました。

| ファイル名 | 文字数 | 平均処理時間 | 出力HTML長 |
|-----------|--------|-------------|-----------|
| zenn-markdown-editor-by-cm6.md | 16,858 | 34.14 ms | 76,360 |

## 改善効果

これまではサーバーの負荷を抑えるために300msのデバウンス時間を設けており、プレビューの反映には少しもっさりとした印象がありました。しかし、処理をブラウザ側で実行する仕組みへと変更したことで負荷の制約がなくなり、デバウンスを50msまで小さくしました。

その結果、60~70msでプレビューが反映されるようになり、以前よりもスムーズで快適な執筆体験が得られるようになりました。

それではまた！
