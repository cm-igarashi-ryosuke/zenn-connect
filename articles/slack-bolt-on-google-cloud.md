---
title: "Slack BoltをGoogle Cloudにデプロイするノウハウ"
emoji: "⚡️"
type: "tech"
topics: ["slack", "gcp", "cloudrun", "cloudfunctions"]
published: true
---

[Slack Bolt](https://github.com/SlackAPI/bolt-python)はSlack AppのAPIサーバーを作るフレームワーク（JavaScriptとPythonがある）です。APIサーバーをGoogle Cloudにデプロイする場合、Cloud FunctionsやCloud Runが候補として思いつくところですが、Slackの制約を理解していないとデプロイしたあとでハマることがあります。この記事では、Slack BoltをGoogle Cloudにデプロイする前に知っておいたほうが良いことをまとめます。

## 最初に結論

Slack BoltをGoogle Cloudにデプロイするなら、**Cloud Runで常時稼働CPUを設定する**のがおすすめです！

## Slackの制約

これは知っておいた方がいいという制約が2つあります。

- 3秒以内に応答がないとリトライされる
- アクションやコマンドは3秒以内に応答がないとエラー表示になる

### いわゆる「3秒ルール」

まずは起動時間がネックになります。Cloud FunctionsにしてもCloud Runにしても、コールドスタートからの3秒以内にレスポンスは厳しいです。かと言って常時インスタンスをスタンバイさせておくと、コストがかかります。（コストをかけてもいいと言う場合は、この話はここでおしまいです）

この問題点としては、リトライされたリクエストも処理してしまうことで、 **処理の重複が発生してしまう** という点です。例えば、何かしらのメッセージに反応してレスを返すようなAppの場合、リトライされたリクエストに対してもレスを返してしまうと、同じメッセージに対して2回レスが返ってしまいます。

コストをかけない回避策として最も簡単なのが、リトライされたリクエストを無視することです（多少、信頼性は下がりますが）。リトライリクエストの場合、リクエストヘッダーに `x-slack-retry-num` というキーが付与されているので、これをみてリトライかどうかを判定できます。

こちらの記事がとてもわかりやすく参考になりました。（もこさん、ありがとう）

https://dev.classmethod.jp/articles/slack-resend-matome/

### 非同期処理

もう一つネックになるのが、アクションやコマンドの処理では3秒以内にレスポンスを返さないとエラーになるという点です。Slack Boltでは、ハンドラー関数で `ack()` を実行することでレスポンスを返します。レスポンスを返した後に時間がかかる処理を継続したいところですが、Cloud FunctionsやCloud Runはレスポンスを返すとCPUを開放してしまい、処理を継続することができません。

Slack Bolt(Python)には[Lazy リスナー](https://slack.dev/bolt-python/ja-jp/concepts#lazy-listeners)という非同期処理を行うための機能が備わっていますが、これは[AWS Lambdaにしか対応していない](https://github.com/slackapi/bolt-python/issues/46#issuecomment-930048433)ことに注意してください。つまりCloud Functionsを使う場合、自前で非同期処理を外に逃がす仕組みを作る必要があります。（しかしGoogle Cloud上の構成が増えたり、ローカル開発がしにくくなるのであまりやりたくはありません。）

Cloud Runであれば、[常時稼働CPU](https://cloud.google.com/blog/ja/products/serverless/cloud-run-gets-always-on-cpu-allocation)を割り当てれば、インスタンスが起動中はリクエストを処理していない間もCPUを使うことができますので、この設定をするだけで良いです。

## おわりに

最初にも書いたんですが、Cloud Runにデプロイするのがおすすめです。

ちょっとネガティブな話になってしまいましたが、Slack Bolt自体はとてもよくできていて（生のWebhookを処理するのと比べて）実装工数を大幅に減らしてくれますし、Socket Modeの対応など開発者フレンドリーなところも好感です。

あと、なにか困ったらSlack Boltの開発者でもある[@seratch](https://qiita.com/seratch)さんが書かれている記事がとてもわかり易く参考になるのでおすすめです。（公式ドキュメントはすごく細かく書かれているものの、ちょっと分かりにくいですね。）
