---
title: "JavaScriptでインラインコメント機能を実装する"
emoji: "🦔"
type: "tech"
topics: ["javascript"]
published: false
---

技術検証として、簡易的なインラインコメント機能を実装してみました。インラインコメントとは、Notion や Google Docs にあるような、テキストの一部にコメントを付ける機能のことです。

検証の課題としては、Webページ上で**どうやってユーザーの選択範囲を取得するのか**ということでした。結論としては、Web APIの [Selectionオブジェクト](https://developer.mozilla.org/ja/docs/Web/API/Selection) を使うことで、選択範囲にあるNodeを取得でき、これを使って選択範囲の位置を特定できました。思ったよりも早く結論が出たので、コメントのハイライトや複数のコメントの重なりの表現なども試してみました。

## できたもの

Resultのタブで動作が確認できます。

@[jsfiddle](https://jsfiddle.net/bisque/a51fud6m/777/)

## 解説

実装はすべてPure JavaScriptです。（思ったよりDOMの操作が多く、途中でReactを使わなかったことを後悔しましたが、ブラウザのWeb APIをゴリゴリ書くのもいい経験だったと思います。）

### 選択範囲の取得

先に述べたとおり、Selectionオブジェクトを使うことで取得できます。

まずはシンプルな例です。

@[jsfiddle](https://jsfiddle.net/bisque/46j0prne/17/)

ブラウザ上でテキストを範囲選択したイベントを取るには、`mouseup` イベントをリッスンします。選択範囲が閉じている（範囲選択でない）かを判断するには、`selection.isCollapsed` を使うことができます。

`selection.anchorOffset` が選択範囲の先頭、 `selection.focusOffset` が選択範囲の末尾になります。注意点として、先頭・末尾はテキストの向きとは関係なく、選択を開始した始点が先頭、選択を終了した終点が末尾になることです。

次に、少し複雑な例を見てみましょう。この例では、テキストに装飾が施されています。ちょうど、この次に示すテキストがハイライトされているような状態です。

@[jsfiddle](https://jsfiddle.net/bisque/k1ws4ucm/12/)

このようにテキストが単純なTextNodeだけのHTMLでない場合、最初の例で説明した`selection.anchorOffset` と `selection.focusOffset` だけでは選択範囲を特定できない場合があります。これは、`selection.anchorOffset` は`selection.anchorNode`の中の開始位置、`selection.focusOffset`は`selection.focusNode`の中の終了位置を表すからです。

具体的に言うと、選択範囲が `テキストの<span>選択範囲</span>を取得` のように複数のNodeにまたがっている場合、`selection.anchorNode` は `装飾されたテキストの` というTextNode、`selection.focusNode` は `を取得する` というTextNodeになり、`selection.anchorOffset` と `selection.focusOffset` はそれぞれのNode内のオフセットになります。

したがってテキスト全体の正確な選択範囲を取得するには、

1. テキスト全体の `Node` から `anchorNode` と `focusNode` の位置関係を調べる。位置関係というのは、`anchorNode` と `focusNode`のどちらが先頭なのかや、`anchorNode` と `focusNode`がテキスト全体の `Node` の中に収まっているか、などです。
2. 位置関係の確認ができたら、テキスト全体の `Node` の子要素を先頭から捜査し、テキスト全体における開始位置と終了位置のオフセットを求めます。

### テキストのハイライト

テキストのハイライトは、選択範囲が変わったときやコメントが追加・削除されたときに、テキストに装飾を行います。仕組みとしては、状態（例ではグローバル変数）として「プレーンなテキスト」「選択範囲のオフセット」「コメントのオフセット」を持ち、それらを元にテキストに装飾を行い再レンダリングします。

例えば、テキストが `テキストの選択範囲を取得する` で、コメントのオフセットが `(5, 9)` だったとします。このとき、テキストを装飾すると、 `テキストの<span class="depth1">選択範囲</span>を取得する` となります。

また、複数のコメントでオフセットの範囲が重なる場合は、色の濃淡で重なりを表現します。例えば、コメントのオフセットが `(5, 9)` と `(7, 12)` だったとします。このとき、テキストを装飾すると、 `テキストの<span class="depth1">選択</span><span class="depth2"範囲</span><span class="depth1">を取得</span>する` となります。

今回の実装ではこれを以下のように行いました。

1. テキスト1行の1文字ごとに `{mask: number, depth: number}[]` という構造の配列に変換する（実装内の`function getMask`）
2. この配列をNode単位にまとめて `{mask: number, depth: number, text: string}[]` という構造の配列にする（実装内の`function getTextBlocks`）
3. これをHTMLに変換する（実装内の`function renderText`）

このあたりの実装は、NotionをWebブラウザ上で操作して、コメントを追加したときにHTMLがどう変わったかを観察し、その挙動を真似しました。

## おわり


