---
title: "Railsのマスターキーでcredentialファイル以外のファイルを暗号化・復号化する"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails"]
published: true
---

Railsには、APIキーなどのシークレット情報を安全に管理するための機能があります。

https://railsguides.jp/security.html#%E7%8B%AC%E8%87%AA%E3%81%AEcredential

マスターキーを使って事前にシークレット情報を暗号化されたcredentialファイル（`config/credentials.yml.enc`）に書き込み、アプリケーションの起動時にもマスターキーを使ってcredentialファイルを復号化し、アプリケーション内で参照可能になるという仕組みです。

マスターキーさえ安全に管理していればcredentialファイルはリポジトリで管理することができます。また、実行環境に与える環境変数はマスターキーだけで良いので、シークレット情報が漏洩するリスクを減らすことができます。

しかし中には「RSAの秘密鍵」などcredentialファイルで管理するには少々大きいシークレット情報もあり、**credentialファイル以外のファイルをファイル単位で暗号化・復号化できる仕組み**を考えてみました。

## ファイルを暗号化・復号化するタスクを用意する

事前にファイルをマスターキーで暗号化するrakeタスクを用意しました。

@[gist](https://gist.github.com/bisque33/029339742eed3bbd7a8788b618f077f0)

例えば、`staging`環境用の`private-key.staging.pem`というファイルを暗号化したい場合、`staging`環境用のマスターキー（`./config/credentials/staging.key`）を用意し、以下のようにrakeタスクを実行します。

```
rails credential:encrypt[private-key.staging.pem,staging]
# => ./config/credentials/private-key.staging.pem.enc が生成されます
```

正しく暗号化されているか（復号化できるか）を確認するには、以下のようにrakeタスクを実行します。

```
rails credential:decrypt[private-key.staging.pem.enc,staging]
# => ./config/credentials/private-key.staging.pem が生成されます（ファイルがある場合は上書きされるので注意してください）
```

## アプリケーションの起動時に暗号化されたファイルを復号化する

暗号化されたファイルをアプリケーションの起動時に復号化するように設定します。場合によっては、起動時ではなく必要になる直前にしても良いかもしれません。

```ruby:./config/application.rb
module MyApp
  class Application < Rails::Application
  
    # 暗号化された秘密鍵の復号化
    content_path = Rails.root.join("config", "credentials", "private-key.#{ENV["ENV_NAME"]}.pem")
    unless File.exist?(content_path)
      encrypted_path = Rails.root.join("config", "credentials", "private-key.#{ENV["ENV_NAME"]}.pem.enc")
      # ENV["RAILS_MASTER_KEY"]により復号化されます。
      contents = encrypted(encrypted_path).read 
      IO.binwrite(content_path, contents)
    end
    
  end
end
```

これで実行環境に復号化されたファイル（`./config/credentials/private-key.staging.pem`）が生成されます。

## その他の戦略

実行環境によっては、今回紹介した以外の方法でもシークレットなファイルを安全に実行環境に与える方法があるかもしれません。

例えば、Google Cloud Runであれば、[Secret Managerをボリュームとしてマウント](https://cloud.google.com/run/docs/configuring/secrets)して参照する方法があります。Secret Managerで管理できるファイルのサイズは64kbまでなので、もっと大きなファイルの場合は[Cloud Storage FUSE](https://cloud.google.com/storage/docs/gcs-fuse)を使ってCloud Storageバケットをファイルシステムとしてマウントするのもありかもしれません。（試してないですけど！）

また、たくさんの（あるいは巨大な）ファイルをアプリケーションの起動時に復号化すると、アプリケーションの開始が遅くなってしまうデメリットも考えられます。アプリケーションの特性や実行環境などに応じて戦略を検討してください。

## まとめ

Railsのcredentialの仕組みを利用して、任意のファイルを暗号化・復号化することができました。実行環境におけるシークレット情報の管理がマスターキー１つで済むので便利です。他にも良い方法があれば、ぜひコメント欄で教えて下さい。
