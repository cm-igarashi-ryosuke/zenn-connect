---
title: "ADKマルチエージェント構成で遭遇する「Multiple tools」エラーと、その解決策"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googleadk", "gemini", "vertexai", "agent", "python"]
published: false
---

## はじめに

Google ADK（Agent Development Kit）を使ってエージェントアプリケーションを構築していたところ、以下のエラーに遭遇しました。

```
400 INVALID_ARGUMENT: Multiple tools are supported only when they are all search tools.
```

エラーメッセージだけではすぐにピンとこなかったので、原因の調査から解決に至るまでの経緯を整理してみます。

検証環境: ADK v1.28.0、Vertex AI 経由、Python 3.13

## 単一エージェントでのツール混在

最初に作ろうとしていたのは、カスタム関数（Function Calling）と Google Search を1つのエージェントにまとめた構成でした。

```python:agent.py
from google.adk.agents import Agent
from google.adk.tools import google_search

agent = Agent(
    model="gemini-2.5-flash",
    name="assistant",
    instruction="...",
    tools=[
        search_knowledge_base,  # カスタム Python 関数
        google_search,          # Gemini 組み込みツール
    ],
)
```

しかし、これを実行してみると以下のエラーが返ってきました。

```
400 INVALID_ARGUMENT. {'error': {'code': 400, 'message': 'Multiple tools are supported only when they are all search tools.', 'status': 'INVALID_ARGUMENT'}}
```

### 原因

調べてみたところ、Gemini 2.x 系のモデルでは、1回の API リクエストに含められるツールは**1種類のみ**という制約があるようです。ADK が提供するツールには、大きく分けて以下の種別があります。

| 種別 | ADK でのインターフェイス |
|---|---|
| カスタム関数（Function Calling） | Python 関数、`FunctionTool`、`BaseTool` のサブクラス |
| Google Search | `google_search` |
| Vertex AI Search | `VertexAiSearchTool` |
| Code Execution | `code_execution` |
| URL Context | `url_context` |
| Google Maps | `google_maps_grounding` |

カスタム関数以外は Gemini の組み込みツールとして扱われます。**カスタム関数と組み込みツールを1つの API リクエストに含めることができない**、という制約です。ちなみに、これは ADK の制約ではなく、Gemini API（モデル）側の制約になります。

## sub_agents 構成での回避を試みる

ツール種別を分離すれば回避できるのではないかと考え、各サブエージェントが1種類のツールのみを持つ `sub_agents` 構成に変更してみました。

```python:agent.py
search_agent = Agent(
    model="gemini-2.5-flash",
    name="search_agent",
    description="組織固有のナレッジベースを検索するエージェント",
    instruction="...",
    tools=[search_knowledge_base],  # FunctionDeclaration のみ
)

web_agent = Agent(
    model="gemini-2.5-flash",
    name="web_agent",
    description="一般的な情報を Google 検索で取得するエージェント",
    instruction="...",
    tools=[google_search],          # 組み込みツールのみ
)

root_agent = Agent(
    model="gemini-2.5-flash",
    name="root_agent",
    instruction="ユーザーの質問に応じて適切なサブエージェントに委譲する",
    sub_agents=[search_agent, web_agent],
)
```

これなら大丈夫だろうと思っていたのですが、この構成でも同じ `400 INVALID_ARGUMENT` エラーが発生しました。

### GitHub での議論

この問題は GitHub Issue として報告・議論されています。

- [google/adk-python#899](https://github.com/google/adk-python/issues/899) — sub_agents で異なるツール種別を使うと失敗する
- [google/adk-python#969](https://github.com/google/adk-python/issues/969) — 組み込みツールと FunctionDeclaration の共存不可（Master Issue）

Issue 内では `AgentTool` でラップする方法や、組み込みツールをカスタム関数に置き換える方法などの回避策が提案されています。ただ、ADK のバージョンや構成によって効果が異なるとの報告もあり、自分の環境では完全な解決には至りませんでした。

## Gemini 3 で解決

そんな中、Gemini 3 モデルで**ツールの組み合わせ（Tool Combination）** 機能がサポートされたことがわかりました（プレビュー）。

https://ai.google.dev/gemini-api/docs/tool-combination

これにより、組み込みツールとカスタム関数を**同一リクエスト内で併用**できるようになりました。

### 検証結果

実際に動くのか気になったので、モデルを `gemini-3-flash-preview` に変更して4つの構成を検証してみました。

| # | 構成 | モデル | 結果 |
|---|---|---|---|
| 1 | 同一エージェントにツール混在 | gemini-2.5-flash | **エラー** |
| 2 | sub_agents 構成（ツール分離） | gemini-2.5-flash | **エラー** |
| 3 | 同一エージェントにツール混在 | gemini-3-flash-preview | **成功** |
| 4 | sub_agents 構成（ツール分離） | gemini-3-flash-preview | **成功** |

Gemini 3 にすることで、単一エージェントでもサブエージェント構成でもツール種別の制約を気にする必要がなくなったようです。

## まとめ

Gemini 2.x 系では、組み込みツールとカスタム関数の混在による `400 INVALID_ARGUMENT` エラーが発生します。sub_agents でツールを分離しても回避できず、なかなか悩ましい制約でした。

Gemini 3 では Tool Combination 機能（プレビュー）により、この制約が解消されています。単一エージェントでもサブエージェント構成でも正常に動作することを確認できたので、同じエラーで困っている方の参考になれば幸いです。

内容に誤り等がありましたら、コメントで教えていただけると助かります。
