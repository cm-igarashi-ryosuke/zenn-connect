---
title: ZennのMarkdownエディタにテキスト検索機能を実装しました
emoji: 🔍
type: tech
topics: [codemirror, react]
published: false
publication_name: team_zenn
---

ZennのMarkdownエディタにテキスト検索機能を実装しました。

本記事では、[CodeMirror](https://codemirror.net/)の[@codemirror/search](https://www.npmjs.com/package/@codemirror/search)をベースに、ミニマムな検索機能の実装方法を解説します。

## なぜ検索機能を実装したのか

ZennのMarkdownエディタに使われているCodeMirrorは、パフォーマンスの最適化のため**仮想スクロール**と呼ばれるテクニックを使いDOMを描画しています。仮想スクロールは可視範囲以外はDOMが表示されていないので、ブラウザの検索機能ではエディタ内のテキスト全体を検索することができない問題がありました。

:::details 仮想スクロールとは
仮想スクロールとは、大量のデータを描画する際にパフォーマンスを落とすことなく描画とスクロールを実現するテクニックです。仕組みとしては、データ全体を描画するためのエリアを用意しつつ、その中でブラウザの画面に見えている部分だけを描画するというものです。

仕組みを理解するには以下の記事が分かりやすいです。

https://qiita.com/neer_chan/items/5ff1a82ed2fe121026d5

https://zenn.dev/so_nishimura/articles/6a934ad066bedf
:::

これを解決するには、以下の手段が考えられました。

1. CodeMirrorの仮想スクロールを無効化するか描画範囲を広げる
2. CodeMirrorの検索機能を使う

しかし、CodeMirror v6の仮想スクロールは無効化したり描画範囲を広げたりというオプションは提供されていなかったため、1は選択できませんでした。

実装を確認したところ、ブラウザの可視範囲に加えて、上下に合計1000pxをスクロール方向などを加味しつつ、余分に描画していることがわかりました。

https://github.com/codemirror/view/blob/6.26.3/src/viewstate.ts#L46-L47

CodeMirrorの作者であるmarijn氏もフォーラムで以下のようにコメントしています。

https://discuss.codemirror.net/t/viewport-issues-with-cm-6/3586/2

> Viewporting is a rather fundamental aspect of the way the library is designed (much of the complexity analysis in the implementation depends on it) and not something that can be turned off. (I keep getting CM5 support requests where people set viewportMargin: Infinity and then are surprised when the editor gets slow. I want to avoid that failure case this time.)
>
> The way this breaks browser search is very annoying, but a trade-off that seems unavoidable with this design.

というわけで、2の「CodeMirrorの検索機能を使う」ことにしました。

## @codemirror/search をカスタマイズする

@codemirror/searchそのまま利用すると、以下のような検索機能が表示されます。本来、CodeMirrorはソースコードのためのエディタであり、Markdownエディタに対する検索機能としてはいささか機能が過剰でした。

![](/images/articles/cm6-simple-search-extension/default-panel.png)
*デフォルトの検索パネル*

幸い、@codemirror/searchのExtensionには、独自の検索パネルを差し込むオプションが用意されていましたので、そちらを使って見た目と機能を調整しました。

![](/images/articles/cm6-simple-search-extension/custom-panel.png)
*カスタマイズした検索パネル*

検索パネルのインターフェイスは以下のように定義されています。

![](/images/articles/cm6-simple-search-extension/create-panel.png)
*https://codemirror.net/docs/ref/#search.search^config.createPanel*

が、具体的な実装例を探しても見つからなかったので、@codemirror/searchの[SearchPanel](https://github.com/codemirror/search/blob/affb772655bab706e08f99bd50a0717bfae795f5/src/search.ts#L560)の実装をベースに不要な機能を削ぎ落とす形で実装しました。

CodeMirrorの作者であるmarijn氏もフォーラムで以下のようにコメントしています。

> The [implementation](https://github.com/codemirror/search/blob/affb772655bab706e08f99bd50a0717bfae795f5/src/search.ts#L560) of the built-in search panel is probably a good place to start.

*https://discuss.codemirror.net/t/replace-default-search-panel-with-my-own-component/5885/2*

### SearchPanelのカスタム実装

以下が、カスタム検索パネルの実装例です。長いので区切りが分かりやすいように分割しますが、全て1つのファイルのソースコードです。

```typescript
import {
  SearchQuery,
  closeSearchPanel,
  findNext,
  findPrevious,
  getSearchQuery,
  search,
  searchKeymap,
  setSearchQuery,
} from '@codemirror/search';
import {
  EditorView,
  Panel,
  ViewUpdate,
  keymap,
  runScopeHandlers,
} from '@codemirror/view';

class SearchPanel implements Panel {
  searchField: HTMLInputElement;
  dom: HTMLElement;
  query: SearchQuery;

  constructor(readonly view: EditorView) {
    const query = (this.query = getSearchQuery(view.state));
    this.commit = this.commit.bind(this);

    // searchFieldの初期化
    this.searchField = document.createElement('input');
    this.searchField.value = query.search;
    this.searchField.placeholder = 'Search';
    this.searchField.setAttribute('aria-label', 'Search');
    this.searchField.className = 'cm-textfield';
    this.searchField.name = 'search';
    this.searchField.setAttribute('form', '');
    this.searchField.setAttribute('main-field', 'true');
    this.searchField.onchange = this.commit;
    this.searchField.onkeyup = this.commit;

    function button(name: string, onclick: () => void) {
      const button = document.createElement('button');
      button.className = 'cm-button';
      button.name = name;
      button.onclick = onclick;
      button.type = 'button';
      button.setAttribute('aria-label', name);

      return button;
    }

    // domの初期化
    this.dom = document.createElement('div');
    this.dom.className = 'cm-search-custom'; // デフォルトのthemeが適用されるのを回避するため、独自のクラスを指定
    this.dom.onkeydown = (e) => this.keydown(e);

    this.dom.appendChild(this.searchField);

    const nextButton = button('next', () => findNext(view));
    this.dom.appendChild(nextButton);

    const prevButton = button('prev', () => findPrevious(view));
    this.dom.appendChild(prevButton);

    const closeButton = button('close', () => closeSearchPanel(view));
    this.dom.appendChild(closeButton);
  }
```

`SearchPanel`は@codemirror/searchの[SearchPanel](https://github.com/codemirror/search/blob/6.5.6/src/search.ts#L586)から必要な機能だけを残した独自の検索パネルです。@codemirror/searchはJavaScriptのライブラリなのでDOMもWeb APIを使って生成します。

`getSearchQuery`は@codemirror/searchで定義された`StateField<SearchState>`の`query`フィールドを取得します。`query`フィールド（`SearchQuery`）はデフォルトの検索パネルの状態管理にも使われるもので、たくさんのフィールドを持ちますが、カスタム検索パネルではその内の`search`フィールドのみを利用します。`search`フィールドはstring型で検索文字列を管理するフィールドです。

```typescript
  commit() {
    const query = new SearchQuery({
      search: this.searchField.value,
    });
    if (!query.eq(this.query)) {
      this.query = query;
      this.view.dispatch({ effects: setSearchQuery.of(query) });
    }
  }

  keydown(e: KeyboardEvent) {
    if (runScopeHandlers(this.view, e, 'search-panel')) {
      // scope内でsearchKeymapに対するキーバインドが実行されたとき、preventDefaultを行う
      e.preventDefault();
    } else if (e.code == 'Enter' && e.target == this.searchField) {
      e.preventDefault();
      (e.shiftKey ? findPrevious : findNext)(this.view);
    }
  }

  update(update: ViewUpdate) {
    for (const tr of update.transactions)
      for (const effect of tr.effects) {
        if (effect.is(setSearchQuery) && !effect.value.eq(this.query))
          this.setQuery(effect.value);
      }
  }

  setQuery(query: SearchQuery) {
    this.query = query;
    this.searchField.value = query.search;
  }

  mount() {
    this.searchField.select();
  }

  get top() {
    return true; // .cm-panels-top の位置に表示する
  }
}
```

`commit()`はinputの状態を監視し、変更があれば`this.view.dispatch`で状態（`SearchQuery`）を更新します。

`keydown()`はキー入力を監視し、Enterキーが押された時に検索を実行するCommand（`findNext`または`findPrevious`）を実行します。

`update()`は外部から`SearchQuery`の状態が変更された場合に`setQuery()`を呼び出します。このカスタム検索パネルでは必要ないかもしれませんが念のため最低限は残しています。

その他、お作法的に必要なものを残します。

```typescript
const baseTheme = EditorView.baseTheme({
  // デフォルトのスタイルを上書き
  '.cm-panels.cm-panels-top': {
    // Zenn独自のスタイルなので省略します
  },
  // カスタムのスタイル
  '.cm-panel.cm-search-custom': {
    // Zenn独自のスタイルなので省略します
  },
});
```

`baseTheme`は独自の検索パネルのスタイル定義です。詳細はZenn独自のスタイルなので省略しますが、工夫としては、クラス名を独自の`cm-search-custom`にすることで、デフォルトのスタイルが適用されるのを防いでいます。

```typescript
export const searchPanel = [
  // search Extensionに独自のSearchPanelを差し込む
  search({ createPanel: (view) => new SearchPanel(view) }),
  // 独自のSearchPanelのスタイルを適用する
  baseTheme,
  // デフォルトのSearchPanelのキーマップを適用する
  keymap.of(searchKeymap),
];
```

最後にこれらをまとめてExtensionとして取り込める形でexportします。

これで、カスタム検索パネルが実装できました。

## おわりに

相変わらずCodeMirrorのドキュメントは初見で理解するのが難しいですが、実装とドキュメントを照らし合わせながら何度も読み返すことでようやく理解できました。また、はじめてExtensionの内部実装を読み、状態管理の仕組みもなんとなく分かってきたので、次は状態管理についてより詳しく学びたいと思います。
