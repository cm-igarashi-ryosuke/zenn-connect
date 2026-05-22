---
title: "CloudflareでProxyしているサイトにレートリミットを導入する方法を調べました"
emoji: "🚦"
type: "tech"
topics: ["cloudflare", "cloudflareworkers", "durableobject", "ratelimit"]
published: false
---

## はじめに

特定の API エンドポイント（1 リクエストが高コストな外部サービスを呼ぶもの）に対して、IP 単位のレート制限が必要になりました。実装場所として、まずはエッジ（Cloudflare）で止めるのがよいだろうと考え、そこを出発点に検討を始めました。

最初に検討したのは Cloudflare の [WAF Rate Limiting Rules](https://developers.cloudflare.com/waf/rate-limiting-rules/) です。しかし、利用中の Pro プラン（$20/月）では Rate Limiting Rules の枠が 2 つまでに制限されており、既存のルールで埋まっていました。枠を増やすには Business プラン（$200/月）へのアップグレードが必要となりますが、今回追加したい 1 ルールのためにプランを 10 倍のコストに引き上げるのは現実的ではありません。

そこで、コストを抑えてレート制限を実現するために別の手段を検討することにしました。

対象のドメインはすでに Cloudflare で Proxy されているため、[Workers Routes](https://developers.cloudflare.com/workers/configuration/routing/routes/) を使えば `example.com/api/example*` のような route パターン単位で Worker をアタッチでき、対象エンドポイントだけを Worker 経由にできます。サイト全体を Worker に流す必要がなく、影響範囲を絞れるのは都合が良さそうでした。

Cloudflare には Workers から利用できるレート制限手段が複数あります。

1. Workers Rate Limiting Binding（公式ビルトイン）
2. Durable Object（DO）
   - 2a. in-memory state でカウント
   - 2b. SQLite-backed storage で永続化

それぞれを順に試しましたが、いずれにも注意点がありました。最終的には backend 側（アプリケーション + RDB）に実装し直しましたが、その過程で得た知見を整理します。

## 要件

- 特定のエンドポイント `/api/example?mode=heavy` を対象とする
- IP 単位で 60 秒間に最大 10 リクエストまで許可する
- 超過した場合は HTTP 429 を返す

## パターン1. Workers Rate Limiting Binding

Cloudflare 公式のビルトイン機能で、`wrangler.toml` に設定を書くだけで利用できます。

### 実装

```toml
# wrangler.toml
[[ratelimits]]
name = "MY_LIMITER"
namespace_id = "1001"

  [ratelimits.simple]
  limit = 10
  period = 60   # 10 or 60 のみ指定可能
```

```typescript
// src/index.ts
export default {
  async fetch(req, env) {
    const key = req.headers.get("cf-connecting-ip") ?? "unknown";
    const { success } = await env.MY_LIMITER.limit({ key });
    if (!success) {
      return new Response("Too Many Requests", { status: 429 });
    }
    return fetch(req);
  },
};
```

実装は数行で済むため非常にシンプルです。

### 検証結果

検証環境にデプロイし、単一の IP から `100 req / 10 秒`（設定値の数十倍）を送信したところ、期待値と実測値に大きな乖離が確認されました。

- 期待値: 10 件成功、90 件 `success: false`
- 実測値: 100 件中 `success: false` は数件程度のみ

設定値 `10 req / 60s` に対し、実効値は数十〜100 req/min 程度まで通る挙動でした。

### 原因

公式ドキュメント（[Workers Rate Limiting - Accuracy](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/#accuracy)）に以下の記述があります。

> The Rate Limiting API is **permissive, eventually consistent, and intentionally designed to not be used as an accurate accounting system**.
> ...the isolate that serves each request will check against its locally cached value of the rate limit. Very quickly, but not immediately...
>
> 出典: <https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/#accuracy>

要約すると以下のとおりです。

- 各 isolate がローカルキャッシュでカウンタを保持している
- isolate 間の同期は数秒程度の遅延がある
- 同一 PoP 内に複数の isolate が存在する場合、リクエストが分散し、各 isolate のローカルカウントが分散する
- Cloudflare 自身が「accurate accounting に使うべきではない」と明言している

このため、設定値どおりの精度が必要なケースにはマッチしないことが分かりました。

### コスト

Rate Limiting Binding 自体には追加課金はありません。Workers のリクエスト数と CPU 時間にのみ課金されます。

| プラン | リクエスト | CPU 時間 |
|---|---|---|
| Workers Free | 100,000 / 日 | 10ms / 呼び出し |
| Workers Paid（$5/月） | 10M / 月含まれる、超過 $0.30 / 100万 req | 30秒 / 呼び出し、追加課金あり |

### 考察

- Workers Rate Limiting Binding は厳密な abuse 防御には向きません
- 緩めのフィルタとしては十分機能しますが、設定値どおりの制御が必要なユースケースでは別手段が必要です

## パターン2. Durable Object（in-memory）

Durable Object（DO）は「論理的に単一インスタンス」として動作するため、複数 PoP からのリクエストを集約してカウントできます。本パターンでは in-memory state でカウンタを保持します。

なお、Durable Object を Workers Free プランで利用するには、DO クラスを `new_sqlite_classes` として宣言する必要があります（旧来の Key-Value-backed は Workers Paid プラン必須）。本パターンでは DO クラスは SQLite-backed として宣言しますが、コード側では `ctx.storage` を呼ばず in-memory state のみで動作させます。

### 実装

wrangler.toml では SQLite-backed として宣言します。

```toml
[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["RateLimiter"]   # Free プラン対応に必要
```

IP ごとに DO を作成する設計とします（`idFromName(ip)`）。カウンタはクラスのインスタンス変数として in-memory に保持します。

```typescript
// rate_limiter.ts
export class RateLimiter extends DurableObject {
  private timestamps: number[] = [];

  async checkLimit() {
    const now = Date.now();
    this.timestamps = this.timestamps.filter(t => now - t < 60_000);
    if (this.timestamps.length >= 10) return { success: false };
    this.timestamps.push(now);
    return { success: true };
  }
}
```

```typescript
// index.ts (Worker 本体)
const id = env.RATE_LIMITER.idFromName(ip);
const stub = env.RATE_LIMITER.get(id);
const { success } = await stub.checkLimit();
```

storage API を呼ばないため、SQLite の rows read/written 課金は発生しません。

### 検証 1. バースト動作

11 件を連続で送信したところ、期待どおり 11 件目から 429 が返りました。

```
1〜10: HTTP 200
11〜15: HTTP 429
```

### 検証 2. idle を挟んだ場合

`5 件 → 15 秒 idle → 6 件` という条件で送信しました。通常の sliding window が機能していれば、Phase 1 の 5 件のタイムスタンプは 60 秒以内であるためウィンドウに残っており、Phase 2 の 6 件目（累計 11 件目）で 429 が返るはずです。

実測結果は次のとおりです。

```
Phase 1: 5 件すべて 200
15 秒 idle
Phase 2: 6 件すべて 200
```

Phase 1 のカウントが失われていることが分かります。すなわち、idle を挟むとレート制限を回避できる状態でした。

### 原因

DO のライフサイクルが影響しています。公式の DO Lifecycle ドキュメントに以下の記述があります。

> after 10 seconds of inactivity while in this state.
> **When hibernated, the in-memory state is discarded**
>
> 出典: <https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/>

DO は一定条件下で 10 秒の idle で hibernate され、in-memory state が破棄されます。再アクセス時には `constructor()` から再起動するため、カウンタは初期状態に戻ります。

このため、攻撃者の視点では以下のすり抜けが成立します。

- 10〜15 秒程度の間隔で 9 件ずつバーストを送る
- カウンタが毎回リセットされるため、1 分間に 36〜54 件程度が通過する（設定値の 4〜5 倍）

このすり抜けが成立する以上、abuse 防御として実用に耐えません。

### 考察

- DO の in-memory state は揮発する前提で設計する必要があります
- 短時間バーストは防げますが、間隔を空けた持続的アクセスには対応できません

## パターン3. Durable Object（SQLite-backed persistent）

timestamps を DO の Storage API で永続化する方針に切り替えます。SQLite-backed として作成済みのため、KV API (`ctx.storage.get/put`) を使えば内部的に SQLite に書き込まれます。

### 実装

```typescript
export class RateLimiter extends DurableObject {
  async checkLimit() {
    const now = Date.now();
    const stored = (await this.ctx.storage.get<number[]>("ts")) ?? [];
    const timestamps = stored.filter(t => now - t < 60_000);
    if (timestamps.length >= 10) {
      return { success: false };   // ブロック時は write しない（write 課金を抑制）
    }
    timestamps.push(now);
    await this.ctx.storage.put("ts", timestamps);
    return { success: true };
  }
}
```

DO は single-threaded（input gates）で動作するため、get → 計算 → put の間に他リクエストが割り込む可能性はなく、アトミック性が確保されます。

### 検証結果

検証パターンはすべて期待どおりの結果となりました。

| 検証 | 結果 |
|---|---|
| バースト 11 件目から 429 | OK |
| 15 秒 idle 後の hibernate を跨いでもカウンタが維持される | OK |
| ウィンドウ経過後はカウンタが除外される | OK |

### 追加課題

DO storage には TTL がありません。再訪しない IP の DO storage は明示的に削除しない限り残り続けます。Cloudflare DO Storage には組み込みの TTL 機能がないため、`storage.put` で書いたデータは自動的には消えません。

### 対策

Alarm API を使って TTL 相当の挙動を実装します。DO Alarms は指定時刻にメソッドを呼ぶ機能で、これを利用して TTL 相当の挙動を実装できます。

```typescript
async checkLimit() {
  // ... 既存ロジック ...
  await this.ctx.storage.put("ts", timestamps);
  // 最後のリクエストから 65 秒経過後にクリーンアップ。
  // setAlarm は呼ぶたびに最新時刻に上書きされるため、
  // 連続アクセス中は発火せず、idle 後に発火する。
  await this.ctx.storage.setAlarm(now + 60_000 + 5_000);
}

async alarm() {
  await this.ctx.storage.deleteAll();
}
```

これにより、最後のアクセスから 65 秒経過した IP の DO storage は自動的にクリーンアップされます。

### コスト

Durable Object は Workers Free / Paid プランの両方で利用できますが、Workers Free で利用するには SQLite-backed (`new_sqlite_classes`) として作成する必要があります（旧来の Key-Value-backed は Paid 必須）。

| 課金対象 | Workers Free | Workers Paid |
|---|---|---|
| DO リクエスト | 100,000 / 日 | 1M / 月含まれる、超過 $0.15 / 100万 req |
| Duration（実行時間 × 128MB） | 13,000 GB-s / 日 | 400,000 GB-s / 月含まれる、超過 $12.50 / 100万 GB-s |
| SQLite rows read | 5M / 日 | 25B / 月含まれる、超過 $0.001 / 100万 rows |
| SQLite rows written | 100,000 / 日 | 50M / 月含まれる、超過 $1.00 / 100万 rows |
| SQLite 容量 | 5GB（ストレージ容量上限） | 5GB-月含まれる、超過 $0.20 / GB-月 |

本実装の 1 リクエストあたりの消費は以下です（`setAlarm` は内部的に SQLite への write を伴うため、rows written としてカウントされます）。

| 操作 | rows read | rows written |
|---|---|---|
| 成功時 | 1 | 2（`put` + `setAlarm`） |
| ブロック時 | 1 | 0 |

writes が枠的に最もタイトで、Workers Free プランで運用する場合は `50,000 success/日` 程度が上限となります（`100K writes ÷ 2 writes/req`）。ブロック時には write を発生させない設計としたため、abuse の規模が大きくなっても課金が比例して増えない点はメリットです。

加えて、Worker 自体のリクエスト数（100K/日 Free）も DO とは別に消費されるため、判定対象のエンドポイント全体のトラフィックがこの枠を超えるかどうかも確認が必要です。Workers Free プランの枠内で運用できるかどうかは、対象エンドポイントのトラフィック規模と上記の枠を突き合わせて判断する必要があります。

## Worker で実装する場合の追加考慮事項

パスの正規化に関する話です。最終的には backend 側に実装し直したため本構成は採用していませんが、Worker でレート制限を実装する場合に押さえておきたい注意点として、検証中に判明したパス判定の落とし穴を残しておきます。

判定対象が `/api/example?mode=heavy` のみだと考えて実装すると、bypass される可能性があります。

backend のルーティング（例えば Rails の `resources` 系）では、慣習的に以下のすべてが同じアクションにルーティングされます。

- `/api/example`
- `/api/example.json`（format 自動許容）
- `/api/example/`（trailing slash 許容）
- `/api/example%2ejson`（percent-encoded `.` も decode して `.json` として解釈）

Worker 側の判定が `url.pathname === "/api/example"` の単純比較だと、`%2ejson` などの percent-encoded variant が pass-through され、backend 側でレート制限対象のアクションが実行される状態となります。検証環境で実際に bypass が成立することを確認しました。

### 解決策

URLPattern と `decodeURIComponent` を組み合わせて、変則的なパスもすべて判定対象に含めます。

```typescript
const TARGET_PATTERN = new URLPattern({
  pathname: "/api/example{.:format}?{/}?",
});

const isTarget = (url: URL): boolean => {
  let pathname = url.pathname;
  try {
    pathname = decodeURIComponent(url.pathname);
  } catch {
    // 不正な % エンコーディングは元の pathname のまま判定
  }
  return TARGET_PATTERN.test({ pathname }) &&
    url.searchParams.getAll("mode").includes("heavy");
};
```

- `URLPattern` で `.json` や trailing slash などのバリアントを宣言的に表現
- `decodeURIComponent` で encoded path を事前に decode
- `getAll("mode").includes(...)` により `?mode=normal&mode=heavy` の重複 query で bypass されるのを防止（Rack は最後の値を採用し、`URLSearchParams.get` は最初の値を返すという解釈差を考慮）

検証で判明した点として、Workers ランタイムの URLPattern は percent-encoded path を decode せず raw 比較する挙動でした。そのため、事前に `decodeURIComponent` で正規化する処理は必須です。

## 最終構成

最終的には backend 側（アプリケーション + RDB）にレート制限を実装し直しました。既存の DB をそのまま使えば追加のインフラやコストはほとんど発生しません。処理としても `INSERT ... ON CONFLICT DO UPDATE` 相当の DB クエリ 1 回で済むため、リクエストあたりのオーバーヘッドも DB リクエスト 1 回分のみです。

トレードオフとしては、エッジで止める構成と違い、abuse トラフィックが backend まで到達する点があります。そのため、DoS/DDoS のような大規模リクエストは Cloudflare のルールでレート制限が必要です。

## まとめ

本検証を通じて、目的と規模に応じて以下の使い分けが適切と判断しました。

- **機能単位の細かいレート制限**: backend のアプリケーション層に実装する。既存スタック内で完結し、追加インフラ・課金なしで実現できる
- **大規模アクセス / DoS 対策**: Cloudflare の Rate Limiting Rules（WAF 層）で止める。Workers / DO / backend のいずれも起動しないため、コスト面でも最も有利
- **Workers / Durable Object での実装**: backend に手を入れにくい場合などの選択肢。厳密な制御が必要なら DO + SQLite + Alarm の構成が必要で、それ以外では backend か WAF の方が有利

本ケースでは、機能単位の細かいレート制限が目的であったため、最終的に backend 側での実装を採用しました。
