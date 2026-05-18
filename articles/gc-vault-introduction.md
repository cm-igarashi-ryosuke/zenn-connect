---
title: "長期 credentials をローカルに残さず Google Cloud を操作するための CLI ツール gc-vault を作りました"
emoji: "🔐"
type: "tech"
topics: ["gcp", "1password", "security", "gcloud", "claudecode"]
published: false
---

## はじめに

先日、[ローカル Terraform の Google Cloud 認証を ADC から 1Password + Service Account Impersonation に置き換える](https://dev.classmethod.jp/articles/terraform-gcp-auth-1password-impersonation/) という記事を書きました。Terraform 実行に関しては、`run.sh` の中で 1Password から起点 SA キーを取り出して短命の借用トークンを発行する方式に切り替えており、ローカルディスクに長期 credentials を残さない状態にできました。

ただ、ローカルから Google Cloud を触る場面は Terraform だけではありません。最近だとコーディングエージェント（Claude Code など）に「ログを調査して、コスト分析して」といった作業を頼むこともあります。

エージェントに CLI を使わせる場合、credentials の漏洩リスクに加えて、**意図しない変更を加えてしまうリスク** も気になります。調査を頼んだはずが書き込み系のコマンドを叩いてしまう、というのは普通に起こりえます。人間と同じ強い権限をそのまま渡すのは避けたく、利用主体ごとに **最小権限の原則** を適用したいところです。

`run.sh` のように毎回ラッパースクリプトを書くわけにもいかず、`gcloud auth login` に戻るのも避けたい。そこで汎用的な薄いラッパー CLI として **gc-vault** を作りました。

https://github.com/zenn-dev/gc-vault

この記事では、何ができるツールなのか、なぜ作ったのか、どう使うのかを紹介します。

:::message
gc-vault は現状 WIP（pre-alpha）です。仕様は今後変わる可能性があります。
:::

## 何ができるツールか

`gc-vault exec` の後ろに任意のコマンドを書くと、裏側で 1Password から短命の借用トークンを発行し、それを渡した状態でコマンドが実行されます。

```bash
gc-vault exec my-app-dev -- gcloud projects describe my-app-dev
gc-vault exec my-app-dev -- bq query 'SELECT count(*) FROM ...'
gc-vault exec my-app-dev -- terraform plan
gc-vault exec my-app-dev -- bin/rails console
```

`gcloud auth login` のような恒久的な認証情報は **どこにも書かない**のがポイントです。借用トークンはメモリ上にしか存在せず、プロセスが終われば消えます。`CLOUDSDK_AUTH_ACCESS_TOKEN` か ADC を見るツールであれば一通り動くので、`gcloud` / `bq` / Terraform / 各種クライアントライブラリを使うアプリケーションがそのまま対象になります。

## なぜ作ったか

`gcloud auth login` / `gcloud auth application-default login` が残すリフレッシュトークンは、寿命が事実上無期限で、`~/.config/gcloud/` が漏れると本人が気づかないうちに本番含む全プロジェクトを長期間操作される可能性があります。

Google Cloud には `--impersonate-service-account` などの素材は揃っていますが、**「1Password に長期キーを保管し、必要なときだけ短命トークンを発行する」というフローを統一する薄いラッパー**は手元になかったので、自前で作ることにしました。

## アーキテクチャ

設計目標は「**ローカルディスクに長期 credentials を一切残さない**」の 1 点に絞っています。

```
1Password
  └── bootstrap-{user}@{project} の SA キー JSON
        │ op read (1Password 認証)
        ▼
gc-vault exec <profile> -- <cmd>
  1. 1Password から bootstrap SA キーを取得し一時ファイルへ
  2. CLI 用:  iamcredentials.generateAccessToken で短命トークン (1h) を取得
             → CLOUDSDK_AUTH_ACCESS_TOKEN にセット
  3. SDK 用:  impersonated_service_account 形式の ADC JSON を生成
             → GOOGLE_APPLICATION_CREDENTIALS にセット
             (SDK 側がトークンを自動更新)
  4. cmd を exec
  5. 終了時に一時ファイル削除
```

CLI 用とライブラリ用で 2 通りの環境変数を用意しているのは、`gcloud` / `bq` のような CLI は `CLOUDSDK_AUTH_ACCESS_TOKEN`（静的なトークン文字列）を読み、Google Cloud Client Libraries や Terraform Provider は ADC（`GOOGLE_APPLICATION_CREDENTIALS`）を読むためです。後者は `impersonated_service_account` 形式の ADC JSON を渡すと、SDK 側が必要に応じてトークンを自動更新してくれるので、長時間プロセスでもトークン期限切れを意識せずに済みます。

### 想定する SA 構成

各 Google Cloud プロジェクトに、ユーザーごとに `bootstrap` SA と 1 つ以上の `target` SA を作ります。

| SA | 役割 | 権限 |
|---|---|---|
| `bootstrap-{user}@{project}` | 借用元（1Password に鍵を保管） | 借用先の target SA に対する `roles/iam.serviceAccountTokenCreator` のみ |
| `target-{role}-{user}@{project}` | 借用先（実際にリソースを操作） | ロールに応じた最小権限 |

`bootstrap` は target を借用する以外の権限を持たないので、もし 1Password 経由でキーが漏洩しても、それ単独では何もできません。被害は借用先 target の権限範囲に限定されます。

target SA は **操作する主体（ロール）ごとに複数作れます**。例えば手元の操作用には `target-user-{me}@{project}`、Claude Code 用には readonly 権限だけを付与した `target-claude-{me}@{project}` を作り、プロファイル単位で切り替える、といった運用ができます。命名規約は運用に合わせて自由に調整できます。

## 使い方

### 設定ファイル

`~/.config/gc-vault/config.toml` にプロファイルを定義します。プロファイルごとに「どの 1Password アイテムを起点に、どの target SA を借用するか」を指定するイメージです。

詳細は [README](https://github.com/zenn-dev/gc-vault) と [`examples/config.toml`](https://github.com/zenn-dev/gc-vault/blob/main/examples/config.toml) を参照してください。

### `gc-vault exec`

借用クレデンシャルでコマンドを 1 回実行します。

```bash
$ gc-vault exec my-app-dev -- gcloud projects describe my-app-dev
$ gc-vault exec my-app-dev -- bq query 'SELECT ...'
$ gc-vault exec my-app-dev -- terraform plan
```

### `gc-vault shell`

連続して操作する場合は、借用クレデンシャルがセットされた subshell を起動できます。`exit` で抜けると環境変数も消えます。

```bash
$ gc-vault shell my-app-dev
gc-vault: starting subshell with profile "my-app-dev" (exit to leave)
$ gcloud run services list
$ gcloud sql instances list
$ exit
```

`shell` 中は `GCP_VAULT_ACTIVE_PROFILE` 環境変数がセットされるので、プロンプトに profile 名を出して「今どの環境にいるか」を可視化するのもおすすめです（rc ファイルの末尾、テーマ読み込みより後ろに置く必要があります）。

```zsh:~/.zshrc (末尾)
if [[ -n "$GCP_VAULT_ACTIVE_PROFILE" ]]; then
  PROMPT="(gcp:$GCP_VAULT_ACTIVE_PROFILE) $PROMPT"
fi
```

### `gc-vault doctor`

ローカル環境の前提条件を診断するコマンドです。`gcloud` / `op` の有無、1Password にサインインしているか、設定ファイルが読めるか、`~/.config/gcloud/` に古い credentials が残っていないか、といった項目をまとめてチェックします。

```bash
$ gc-vault doctor
OK    gcloud CLI found
OK    1Password CLI signed in: my.1password.com
OK    config: /Users/alice/.config/gc-vault/config.toml (3 profile(s))
        - my-app-dev
        - my-app-stg
        - my-app-prod
OK    no bare gcloud credentials
```

`gc-vault` 自身は gcloud の state には手を出さない設計なので、移行が済んだら以下のような純正の手順で古い ADC を消しておきます。

```bash
gcloud auth revoke --all
gcloud auth application-default revoke
```

その後、ファイルが残っていれば `gc-vault doctor` が WARN で警告してくれます。

## Claude Code との併用

Claude Code の Bash ツールはデフォルトでサンドボックスが有効で、これが macOS の Unix ドメインソケット等を制限するため、`op` CLI から 1Password デスクトップアプリへの通信経路が遮断され、`gc-vault exec` をそのまま呼んでも失敗します。

サンドボックス設定で必要な経路を恒常的に許可してしまう案もありますが、`op` と 1Password アプリ間の通信は複数の IPC 経路にまたがっており、一度許可するとプロンプトインジェクション時に 1Password 全体への到達が **常時** 開通している状態になるため、採用しませんでした。

代わりに採用しているのは、**呼び出しごとに `dangerouslyDisableSandbox: true` を付ける**運用です。サンドボックスは外れますが Bash 承認プロンプトが出るので、ユーザーの承認なしには進みません。加えて Claude Code の Bash ツールは各呼び出しを独立プロセスとして spawn するため、1Password が各実行を別セッション扱いし、呼び出しごとに 1Password 認証も要求されます。サンドボックス解除と 1Password 認証で **二段階防御** になる、というのが現状の整理です。

リポジトリには Claude Code 用の [Skill 定義](https://github.com/zenn-dev/gc-vault/tree/main/.claude/skills/gc-vault) を同梱していて、これを `~/.claude/skills/` に置いておくと、Claude Code が必要なタイミングで自動的に上記のフロー（`dangerouslyDisableSandbox: true` 付きの `gc-vault exec`）を組み立てるようになります。

```bash
mkdir -p ~/.claude/skills
ln -s "$(pwd)/.claude/skills/gc-vault" ~/.claude/skills/gc-vault
```

### 認証プロンプトが煩わしい場合の代替策

毎回 1Password の認証を承認するのが煩わしい場合は、Claude Code を起動する **前** に `gc-vault shell` でサブシェルに入っておく方法もあります。

```bash
gc-vault shell my-app-dev   # ここで 1Password 認証
claude                       # 借用クレデンシャルを継承して起動
```

この場合、Claude Code 内では環境変数を継承しているので `gc-vault exec` を挟まずに `gcloud` を直接呼べますし、1Password の認証も最初の 1 回で済みます。

ただしトレードオフとして、セッション中は `CLOUDSDK_AUTH_ACCESS_TOKEN` や ADC ファイルがエージェントから読み出し可能な状態にあります。プロンプトインジェクションでこれらが取り出されると、target SA の権限と bootstrap キーの残存期間が攻撃者に渡ります。target SA を readonly に絞ってあれば被害範囲は限定できますが、書き込み権限を持つプロファイルではこの運用は避けたほうが無難だと考えています。

## おわりに

Terraform 記事で書いた「長期な秘密はディスクに置かず、短命トークンで実操作する」原則を、Google Cloud エコシステム全般に広げたい、というのが gc-vault を作った動機でした。

まだ pre-alpha で、いまはチーム内でドッグフーディングしている段階です。反響があれば Homebrew での配布や macOS 以外のマルチプラットフォーム対応も進めていきたいと考えています。

同じような悩みを持っている方の参考になれば嬉しいですし、ご意見があれば Issue やコメントで教えていただけると助かります。
