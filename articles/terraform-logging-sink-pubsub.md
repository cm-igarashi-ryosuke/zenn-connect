---
title: "Cloud LoggingシンクからPub/Subにイベントを送信するTerraform構成"
emoji: "📝"
type: "tech"
topics: ["terraform", "gcp", "cloudlogging", "pubsub"]
published: false
---

## はじめに

Cloud Loggingのログをフィルタリングして、Pub/Subトピックに送信する仕組みをTerraformで構築していたときのことです。

「これ、循環参照になるんじゃないか？」

そう思い込んで、IAM設定だけは手動で行っていました。Terraformで管理すると依存関係がぐるぐる回ってしまうと勝手に判断していたんです。

ところが、その結果どうなったかというと、手動で設定した権限がTerraformの実行時に巻き戻ってしまい、ログの転送が失敗する事態に。「あれ、さっき設定したのに...」と何度も同じ作業を繰り返すハメになりました。

この記事では、実際には循環参照にならないこと、そして全てTerraformで管理できることを説明します。同じような思い込みをしている方の参考になれば幸いです。

## 構成するリソース

今回構築するのは以下の3つのリソースです：

1. **Cloud Logging Sink** - ログをフィルタリングしてPub/Subに転送
2. **Pub/Sub Topic** - ログイベントを受け取るトピック
3. **IAM Binding** - SinkのサービスアカウントにPublisher権限を付与

## 私が陥った誤解

最初、私はこう考えていました。

「Cloud Logging SinkはPub/Sub Topicを参照する。そして、Pub/Sub TopicのIAM設定でSinkのwriter_identityを使う。これって循環参照では？」

頭の中では、こんな図を描いていました：

```
Cloud Logging Sink → Pub/Sub Topic を参照（destination指定）
        ↑                    ↓
        └────────────────────┘
    Pub/Sub Topic → Sink を参照（IAM設定でwriter_identityを使用）
```

だから、IAMだけは手動で設定しないとダメだと思っていたんです。

ところが、ある日Terraformのドキュメントをよく読んでいて、「あれ？」と気づきました。

### 実際には循環参照にならない

Terraformにおける循環参照というのは、リソースAの作成にリソースBが必要で、かつリソースBの作成にリソースAが必要という状態のことです。

でも、今回のケースは違いました。

まず、Sinkの`destination`は文字列で指定します。トピックのリソース参照ではないので、トピックが存在しなくてもSinkは作成できます。

次に、IAMバインディング。これはSinkの`writer_identity`という属性を読み取るだけです。この属性はSink作成時に自動生成されるので、Sinkが先に作られていれば問題ありません。

つまり、実行順序は以下のようになります：

```
1. Cloud Logging Sink作成 → writer_identityが生成される
2. Pub/Sub Topic作成
3. IAMバインディング作成 → 既存のwriter_identityを読み取って設定
```

依存関係は一方向なので、循環参照は発生しないんですね。単純なことに、ドキュメントをちゃんと読んでいなかっただけでした。

## Terraformコード例

### 単一ファイルでの実装

まず、全てを1つのファイルに書く場合の例です：

```hcl
# Cloud Logging Sink
resource "google_logging_project_sink" "notify-sink" {
  name                   = "notify-sink"
  description            = "アプリケーションの通知対象ログをフィルタしてPub/Subに送信"
  destination            = "pubsub.googleapis.com/projects/${var.project_id}/topics/cloud-logging-notify-sink"
  filter                 = "jsonPayload.notify = true"
  unique_writer_identity = true
}

# Pub/Sub Topic
resource "google_pubsub_topic" "log-notification-topic" {
  name = "cloud-logging-notify-sink"
}

# IAM Binding: SinkのサービスアカウントにPublisher権限を付与
resource "google_pubsub_topic_iam_member" "publisher" {
  topic  = google_pubsub_topic.log-notification-topic.name
  role   = "roles/pubsub.publisher"
  member = google_logging_project_sink.notify-sink.writer_identity
}
```

このコードで問題なく動作します。`terraform plan`を実行してみると、ちゃんと依存関係を解決してくれていることが確認できました。

### モジュール分割での実装

実際のプロジェクトでは、再利用性と保守性のためにモジュール化することが多いと思います。その場合、`output`ブロックを使ってモジュール間でデータを受け渡します。

#### modules/cloud-logging/main.tf

```hcl
variable "project_id" {
  description = "GCPのproject_id"
  type        = string
}

resource "google_logging_project_sink" "notify-sink" {
  name                   = "notify-sink"
  description            = "アプリケーションの通知対象ログをフィルタしてPub/Subに送信"
  destination            = "pubsub.googleapis.com/projects/${var.project_id}/topics/cloud-logging-notify-sink"
  filter                 = "jsonPayload.notify = true"
  unique_writer_identity = true
}
```

#### modules/cloud-logging/outputs.tf

`output`ブロックでモジュール外部から`writer_identity`を参照できるようにします。

```hcl
output "notify_sink_writer_identity" {
  description = "notify-sinkのwriter identity"
  value       = google_logging_project_sink.notify-sink.writer_identity
}
```

#### modules/cloud-pubsub/main.tf

```hcl
variable "notify_sink_writer_identity" {
  description = "notify-sinkのwriter identity"
  type        = string
}

resource "google_pubsub_topic" "log-notification-topic" {
  name = "cloud-logging-notify-sink"
}

# Cloud Logging SinkのサービスアカウントにPublisher権限を付与
resource "google_pubsub_topic_iam_member" "publisher" {
  topic  = google_pubsub_topic.log-notification-topic.name
  role   = "roles/pubsub.publisher"
  member = var.notify_sink_writer_identity
}
```

#### environments/prd/main.tf

モジュール間の依存関係を定義します。

```hcl
# 先にcloud-loggingモジュールを定義
module "cloud-logging" {
  source     = "../../modules/cloud-logging"
  project_id = var.project_id
}

# cloud-loggingのoutputを使ってcloud-pubsubに値を渡す
module "cloud-pubsub" {
  source                      = "../../modules/cloud-pubsub"
  notify_sink_writer_identity = module.cloud-logging.notify_sink_writer_identity
}
```

## outputの役割について

ちなみに、`output`ブロックは循環参照を解決するためのものではありません。モジュール間でデータを受け渡すための仕組みです。

Terraformのモジュールは独立したスコープを持つため、あるモジュール内のリソース属性を別のモジュールから直接参照することはできません。

例えば、以下のような記述はできません：

```hcl
# これはできない
module "cloud-pubsub" {
  # 別モジュールのリソース属性を直接参照することはできない
  writer_identity = module.cloud-logging.google_logging_project_sink.notify-sink.writer_identity
}
```

`output`ブロックで値を公開することで、モジュール外部から参照可能になります：

```hcl
# これは可能
module "cloud-pubsub" {
  # outputで公開された値は参照できる
  writer_identity = module.cloud-logging.notify_sink_writer_identity
}
```

最初、私はこのあたりも混同していて、「outputを使えば循環参照が解決できる」と勘違いしていました。実際には、そもそも循環参照ではなかったわけですが。

## unique_writer_identityについて

```hcl
resource "google_logging_project_sink" "notify-sink" {
  unique_writer_identity = true  # または false
}
```

この`unique_writer_identity`という設定について、少し補足します。

### unique_writer_identity = true（推奨）

`true`にすると、専用のサービスアカウントが作成されます。

- 形式：`serviceAccount:p{PROJECT_NUMBER}-{UNIQUE_ID}@gcp-sa-logging.iam.gserviceaccount.com`
- メリット：
  - Sink毎に独立した権限管理が可能
  - セキュリティ上より安全
  - 権限の最小化が容易

### unique_writer_identity = false

`false`にすると、プロジェクト共通のCloud Logsサービスアカウントを使用します。

- 形式：`serviceAccount:cloud-logs@system.gserviceaccount.com`
- デメリット：
  - 全てのSinkで同じサービスアカウントを共有
  - 細かい権限制御が困難

特別な理由がない限り、`unique_writer_identity = true`を使うのが良いと思います。

## おわりに

Cloud Logging SinkからPub/Subへの送信は、実は循環参照にはなりません。Sinkの`destination`は文字列指定で、IAMバインディングは属性値の読み取りなので、依存関係は一方向です。

`output`ブロックは循環参照の解決ではなく、モジュール間のデータ受け渡しに使います。

IAMバインディングを含めて全てTerraformで管理することで、手動設定の巻き戻りという問題も起きなくなりました。最初からこうしておけばよかったと反省しています。

この構成により、Cloud Loggingのログを安全かつ効率的にPub/Subに転送し、さらなる処理（Slackへの通知、BigQueryへの保存など）に繋げることができます。

内容に誤り等がありましたら教えて下さい。

## 参考リンク

- [Cloud Logging: Configuring exports](https://cloud.google.com/logging/docs/export/configure_export_v2)
- [Terraform: google_logging_project_sink](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/logging_project_sink)
- [Terraform: google_pubsub_topic_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/pubsub_topic_iam)
