---
title: "AWS SESでメール認証（DKIM / SPF / DMARC）をセットアップして仕組みを理解する"
emoji: "📤️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ses"]
published: false
---

2024年に、Gmailが「[メール送信者のガイドライン](https://support.google.com/mail/answer/81126)」で送信元ドメインのメール認証を必須化したのは記憶に新しいと思います。現在、事業者としてメールを送信するには、メール認証（DKIM / SPF / DMARC）が事実上必須になっています。今回は、新たにAWS SESをセットアップする機会がありましたので、メール認証も含めセットアップしながら仕組みを理解していきたいと思います。

## ドメインを登録する

まずはじめに、SESから送信するメールの**送信元のドメイン**を登録します。この作業では、SESが利用者に対してドメインの所有権を検証します。利用者は検証済みのドメインのFromアドレスでメールを送信できるようになります。

### セットアップ

SESコンソールの ID > IDの作成 から IDタイプ: `ドメイン` を選択し、ドメイン名を入力します。

![](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1788150725/2026/08/31/osyzogqr4osfypzalk02.png)

DKIMの詳細設定 > IDタイプ は「Easy DKIM」を選択します。

今回、DNSはRoute53ではない外部のDNSを使いましたので、Route53への発行のチェックは外して、DNSレコードの登録は手動で行います。

![](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1788150797/2026/08/31/spilmvp5wh5yfypisppt.png)

「Easy DKIM」は後述するDKIMの署名に使う鍵の管理や配布をしてくれるサービスです。

SESはドメインの検証（所有権の確認）を、DKIMのDNSレコードが登録されていることをもって確認しますので、続けてDKIMのセットアップに進みます。

## DKIM（DomainKeys Identified Mail）

DKIMは、送信するメールに署名を付けて、**メールが改ざんされていないことを検証する**仕組みです。

### セットアップ

今回は、DKIMに必要な秘密鍵の管理や公開鍵の配布を自動でおこなう「Easy DKIM」を利用します。（それ以外には、自分で鍵を管理する方法もあります。）

Easy DKIMを選択すると、Easy DKIMが配布する公開鍵をDNSで引けるように、DNSレコードが生成されます。これを送信元ドメインのDNSに登録します。

![](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1788151104/2026/08/31/lmieppkl9qkga7gv6ct3.png)

レコードの形式は `CNAME <token>._domainkey.<送信ドメイン> <token>.dkim.amazonses.com` となります。

※ `token`は鍵の識別子です。

### 検証の仕組み

DKIMの署名は以下のように付与されます。

1. メールの主要ヘッダー（From / Subject / Date / To など）と本文をそれぞれハッシュ化する
2. そのハッシュを秘密鍵で署名 する
3. 結果を`DKIM-Signature`ヘッダーとしてメールに付ける

実際のヘッダーはこんな形です（Gmailの「メッセージのソースを表示」で見えます）

```
DKIM-Signature: v=1; 
                a=rsa-sha256; 
                d=<送信ドメイン>; 
                s=<token>; 
                h=From:To:Subject:Date:...; 
                bh=<ハッシュ>; 
                b=<署名>
```

受信したメールサーバーは、`DKIM-Signature`ヘッダーの情報から公開鍵を参照します。

1. `d=<送信ドメイン>`と`s=<token>`からCNAMEレコードに登録した`<token>._domainkey.<送信ドメイン>`を組み立てる。
2. CNAMEから`<token>.dkim.amazonses.com`を取得する。
3. `<token>.dkim.amazonses.com`のTXTレコードを取得して、公開鍵を取得する。

最後に公開鍵で署名を検証します。結果が一致すれば「DKIM=PASS」となります。

## SPF（Sender Policy Framework）

SPFは送信元メールサーバーのIPが、MAIL FROM（エンベロープFromともいう。Fromアドレスのドメインではないことに注意）のドメインのDNSに登録されているIPアドレスと一致することを検証します。

SESの標準では、このエンベロープFromには`amazonses.com`が設定されます。これだけでもSPFの要件は満たしているのですが、DMARCの要件のうちの一部（エンベロープFromとFromアドレスのドメインが一致すること）が満たされません。DMARCの合格基準は、DKIMかSPFのうち1つ以上を満たすことなのでこのままでもいいのですが、「カスタム MAIL FROM ドメイン」を設定することで、エンベロープFromをカスタムドメインにすることができます。以下にその手順を示します。

### カスタム MAIL FROM ドメインのセットアップ

登録したドメインのページで、カスタム MAIL FROM ドメインを設定します。エンベロープFromはバウンスを受け取れるアドレスでなければならない決まりがあるため、ドメインは慣例として`bounce.<送信元ドメイン>`サブドメインが使われます。登録すると、SPFの検証に必要なDNSレコードが発行されますので、これをDNSに登録します。

![](https://devio2024-media.developers.io/image/upload/f_auto/q_auto/v1788151206/2026/08/31/ufprywlrgd7vgi2wy7we.png)

これを登録することで設定が完了し、エンベロープFrom（具体的にはReturn-Pathヘッダーの値など）が登録したドメインのものになります。

Before:

```
<一意なID>@<リージョン>.amazonses.com
```

After:

```
<一意なID>@bounce.<送信元ドメイン>
```

### 検証の仕組み

1. エンベロープFromにあるドメインに対してTXTレコードを引く

   `dig bounce.<送信元ドメイン> TXT`
   → "v=spf1 include:amazonses.com ~all"

2. includeのドメインに対して、さらにTXTレコードを引く

   `dig amazonses.com TXT`
   → "v=spf1 ip4:199.255.192.0/22 ip4:54.240.0.0/18 … -all"

この中の ip4: を順に見て、送信元のIPアドレスがリストに含まれるかを検証します。

## DMARC（Domain-based Message Authentication, Reporting, and Conformance）

DKIMで「メールが署名されたドメインのもので改ざんされていないこと」、SPFで「メールサーバーのドメインとIPが正しいこと」がそれぞれ検証されました。しかし、これら2つが攻撃者によって正しく設定されたものであり、Fromのメールアドレスだけが偽装されたものである可能性が残ります。DMARCはそれを塞ぐために**Fromアドレスの検証**を行います。

### セットアップ

DMARCについては、SESは関係しません。以下のレコードをDNSサーバーに登録します。

```
TXT _dmarc.<Fromのドメイン> <内容>
```

TXTレコードの内容は以下のとおりです。

| Key  | Value                                                        |
| ---- | ------------------------------------------------------------ |
| v=   | `DMARC1`（固定）                                             |
| p=   | 不合格の場合のアクション。`reject`: 受信拒否, `quarantine`: 迷惑メール, `none`: 何もしない |
| rua= | 集計レポートの送り先。受信サーバーが 1 日 1 回、「どの IP から何通・SPF/DKIM の結果はこうだった」という XML を送ってくる。 |
| ruf= | 失敗レポートの送り先。不合格メール 1 通ごとの詳細（ヘッダ等）を送る。 |
| fo=  | 失敗レポートを出す条件。`1`: SPFかDKIMのどちらか一方でも不合格なら送る（既定の`0`は両方不合格のときだけ） |

見ての通り、レポートの送り先が必要になります。私のプロジェクトでは、[EASYDMARC](https://easydmarc.com/)というサービスを利用しています。

### 検証の仕組み

1. DKIMの検証結果がPASSであり、検証されたドメインとFromアドレスのドメインが一致することを検証する
2. SPFの検証結果がPASSであり、検証されたドメインとFromアドレスのドメインが一致することを検証する
3. 1,2のいずれかがOKであれば、DMARCはPASSとなる。不合格の場合は、DMARCのポリシー（セットアップで登録したもの）に従って処分される

## おわり

メール認証について自分で調べて理解を深めることができました。みなさまの理解のお役に立てれば幸いです。