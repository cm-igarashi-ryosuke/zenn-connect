---
title: "Railsコマンドの実行環境をCloud Run Jobsに移行しました"
emoji: "🦑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "gcp", "appengine", "cloudrun", "cloudrunjobs"]
publication_name: team_zenn
published: true
---

## はじめに

昨年、ZennのバックエンドサーバーであるRailsアプリケーションの実行環境を、Google CloudのApp EngineからCloud Runに移行しました。

https://zenn.dev/team_zenn/articles/migrate-appengine-to-cloudrun

その際、DBマイグレーションやRakeタスクだけはCloud Runでは解決できず、Cloud BuildやApp Engineを実行環境としていました（詳細は👆の記事を参照してください）。今回、これらの実行環境をCloud Run Jobsに移行しましたので、その様子を紹介します。

## きっかけ

移行のきっかけとなったのが、Rubyの3.1から3.2へのバージョンアップでした。先に述べた通り、Rakeタスクの実行環境のため、RailsアプリケーションをApp Engineにデプロイしていました。

App Engineのラインタイムは、[Rubyランタイム](https://cloud.google.com/appengine/docs/flexible/ruby/runtime?hl=ja)を利用していました。Rubyランタイムは、Ruby3.1まではRubyのDockerイメージを利用していたため、App Engineのインスタンスを起動してSSH接続することで、任意のRailsコマンドを実行できていました。しかし、Ruby3.2からは[buildpacks](https://cloud.google.com/docs/buildpacks/overview?hl=ja)を使用してビルドされるようになり、Railsサーバーの起動以外のコマンドを実行することができなくなりました。

このままApp EngineでRakeタスクを実行する環境をを維持するには、[カスタムランタイム](https://cloud.google.com/appengine/docs/flexible/custom-runtimes/about-custom-runtimes?hl=ja)に変更する必要がありましたので、それならばと別の方法を探すことにしました。

## Cloud Run Jobsに移行する

Cloud Run Jobsは、昨年のCloud Run移行の後で登場したサービスです。Cloud Runと同じくコンテナを実行するサービスですが、HTTPリクエストを受け付けるWebサービスではなく、スケジューラーによってトリガーされ、コンテナを起動し処理を実行するサービスです。

Jobのコンテナのイメージは既にCloud Run Serviceで使用しているものを利用し、Jobで実行する際のコマンドと引数だけを変更するようにしました。

以下は、Cloud Buildの設定ファイルに追加したCloud Run Jobsの設定です。

```yaml
  # イメージをビルドしてArtifact Registryにプッシュ
  # ...

  # Jobをデプロイして、DBマイグレーション実行
  - id: apply-db-migrations
    name: gcr.io/google.com/cloudsdktool/cloud-sdk:alpine
    entrypoint: gcloud
    args:
      - run
      - jobs
      - deploy
      - rails-command # Jobの名前
      - --quiet
      - --project=$PROJECT_ID
      - --region=${_REGION}
      - --image=$_ARTIFACT_REPOSITORY_IMAGE_TAG # Cloud Run Serviceで使うのと同じコンテナイメージ
      - --service-account=$_CLOUD_RUN_SERVICE_ACCOUNT
      - --set-cloudsql-instances=$_CLOUDSQL_INSTANCE_FULL_NAME
      - --cpu=1
      - --task-timeout=60m
      - --max-retries=0
      - --parallelism=1
      - --set-env-vars=RAILS_ENV=production
      - --set-secrets=RAILS_MASTER_KEY=RAILS_MASTER_KEY:latest
      - --command=bundle,exec,rails # コンテナイメージのコマンドと引数を上書き
      - --args=db:migrate,db:migrate:status
      - --execute-now # すぐに実行（ステージング環境のみ）
      - --wait # 実行完了まで待機
  
  # Serviceをデプロイ
  # ...
```

このように、デプロイプロセスの中にCloud Run JobsのJobをデプロイするステップを追加することで、DBマイグレーションをデプロイプロセスの中で行うことができます（本番環境ではJobをデプロイするだけで実行は手動トリガーとしています）

また、Rakeコマンドを実行する際は、Google Cloudのコンソールから `args` 部分を書き換えて実行するか、

![](/images/articles/we-use-cloud-run-jobs/cloud-run-jobs-edit.png)

または、gcloudコマンドにより以下のように実行することもできます。

```shell
gcloud beta run jobs update rails-command --args=[ここに任意のコマンド] --execute-now --wait
```

## 思わぬ落とし穴

ここまでは検証環境では上手くいっていたのですが、本番環境にデプロイしたところ、DBとの接続が上手くいかないという問題が発生しました。

実際に発生したエラーです。（インスタンス名の部分だけマスクしてます。）

```
couldn't listen at "/tmp/cloudsql-proxy-tmp/zenn-dev-integration:us-central1:xxx/.s.PGSQL.5432": listen unix /tmp/cloudsql-proxy-tmp/zenn-dev-integration:us-central1:xxx/.s.PGSQL.5432: bind: invalid argument
rails aborted!
ActiveRecord::ConnectionNotEstablished: could not connect to server: File exists
Is the server running locally and accepting
connections on Unix domain socket "/cloudsql/zenn-dev-integration:us-central1:xxx/.s.PGSQL.5432"?
```

調査に当たり、すぐにこちらの記事が見つかり、参考になりました。感謝✨

https://qiita.com/tehu/items/d86ef3bb7625f699c763

Unixソケットのパスの長さに制限があるのですね。

Google Cloudの[ドキュメント](https://cloud.google.com/sql/docs/postgres/connect-run?hl=ja#connect_to)にもしっかりと記載がありました。

> 警告: Linux ベースのオペレーティング システムでは、ソケットパスの最大長は 108 文字です。パスの長さの合計がこの長さを超えると、Cloud Run からソケットに接続できなくなります。

既に運用しているCloud Run Serviceの方は第1世代のランタイムなので問題にはならずにいたのですが、Cloud Run Jobsは第2世代のランタイムでLinux互換ということで、こちらの制限にかかるようになってしまったようです。

ちなみに、Unixソケットのパスの長さがもっと長ければ、以下のようにわかりやすいエラーメッセージが出力されます。

```
ActiveRecord::ConnectionNotEstablished: Unix-domain socket path "/cloudsql/zenn-dev-integration:us-central1:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/.s.PGSQL.5432" is too long (maximum 107 bytes)
```

この問題を解決するためには、DBインスタンスの名前を短くするか、接続方法を変更する（Private IPを使って接続するなど）などの方法がありあすが、ZennチームではDBインスタンスの名前を短くすることにしました。

## おわりに

Cloud Run Jobsを利用することで、DBマイグレーションやRakeタスクを実行できるようになり、App Engineへのデプロイをやめることができました。App Engineでは可能だったSSH接続ができなくなったため、`rails console`などの対話的な操作は行えなくなりましたが、そもそも本番環境で`rails console`を利用することは危険なので、できなくなってよかったと考えています。

Cloud Run Jobsは、これまでのServiceではまかないきれないニーズを満たしてくれるサービスなので、選択肢の1つとして覚えておくと良いと思います。
