---
title: "PostgreSQLのTimeZoneを理解する"
emoji: "🪷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['postgresql']
published: true
publication_name: team_zenn
---

PostgreSQLにおけるTimeZoneの挙動がよく分かっていなかったので、詳しく調べてみました。

## 環境

- Postgresql 14

## TimeZoneはなんの設定か

いきなり核心ですが、**TimeZoneはセッションのパラメータ**です。セッション単位に、セッションが実行するクエリの結果に影響を与えます。

https://www.postgresql.org/docs/14/runtime-config-client.html#RUNTIME-CONFIG-CLIENT-FORMAT

公式ドキュメントから引用します。

> TimeZone (string)
> Sets the time zone for displaying and interpreting time stamps.

TimeZoneの設定があることで、データベースに保存されているTIME ZONE付きデータをセッション（クライアント）のタイムゾーンに合わせて表示したり更新したりできます。

公式ドキュメントのTimeZoneの説明としては、`8.5.3. Time Zones` が一番詳しいです。

https://www.postgresql.org/docs/current/datatype-datetime.html#DATATYPE-TIMEZONES

## TimeZoneはどのように決まるのか

### コネクション接続時の設定

以下の順で決定します。

- `ALTER [DATABASE | USER] SET TIMEZONE TO ~` により設定された TimeZone の値
- `postgresql.conf` に設定されている timezone の値
- 何もなければGMT

### コネクション接続後の設定

コネクション接続後は [SETコマンド](https://www.postgresql.org/docs/current/sql-set.html) で設定を変更することができます。

- `SET [session | local] timezone ~` または `SET [session | local] TIME ZONE ~` で設定

## DBクライアントがTimeZoneを指定する場合

DBクライアントでTimeZoneを指定できる場合、コネクション接続後に `SET SESSION ~` を実行していることが多いです。

例えばRailsのアプリケーションでActiveRecordに以下のようにTimeZoneを指定した場合、

```ruby:config/application.rb
config.active_record.default_timezone = :utc
```

Railsアプリケーションの実行時に、PostgreSQLのログには以下が記録されました。

```
LOG:  statement: SET SESSION timezone TO 'UTC'
```

ただし、JDBCを使用しているクライアントの場合は様子が異なります。ハッキリとしたことは分かっていないのですが、`user.timezone` というJVMのプロパティが影響するようです。コネクション接続時にSETコマンドを実行している様子がないので、どういう仕組みでセッションのTimeZoneを設定しているのかまでは分かりませんでした。

例えば [DBeaver](https://dbeaver.io/) というJDBCを利用するGUIのDBクライアントの場合は、デフォルトではローカルのTimeZoneになりますが、アプリケーションの起動オプションに `-Duser.timezone=<timezone>` を指定することでTimeZoneを指定できます。

## TimeZoneが実行結果に与える影響

それではセッションのTimeZoneが実行結果にどのように影響を与えるかを見てみます。

まずは、以下のようにTIME ZONEあり/なしのカラムを持つテーブルを作成します。

```sql
CREATE TABLE example (
  ts1 timestamp WITHOUT TIME ZONE,
  ts2 timestamp WITH TIME ZONE,
  time1 time WITHOUT TIME ZONE,
  time2 time WITH TIME ZONE
);
```

### 登録・参照の結果

セッションのtimezone=UTCでデータを登録します。

```sql
SET SESSION timezone TO 'UTC';
```

```sql
INSERT INTO example (ts1, ts2, time1, time2) VALUES (
  '2020-01-01 00:00:00',
  '2020-01-01 00:00:00',
  '00:00:00',
  '00:00:00'
);
```

結果を参照すると、登録したデータと同じであることが分かります。

```sql
SELECT * FROM example;
         ts1         |          ts2           |  time1   |    time2
---------------------+------------------------+----------+-------------
 2020-01-01 00:00:00 | 2020-01-01 00:00:00+00 | 00:00:00 | 00:00:00+00
```

次に、セッションのtimezone=Asia/Tokyoで参照します。

```sql
SET SESSION timezone TO 'Asia/Tokyo';
```

timestamp型のTIME ZONEありのカラムだけ、セッションのタイムゾーンの時間が表示されています。

```sql
SELECT * FROM example;
         ts1         |          ts2           |  time1   |    time2
---------------------+------------------------+----------+-------------
 2020-01-01 00:00:00 | 2020-01-01 09:00:00+09 | 00:00:00 | 00:00:00+00
```

逆も然りで、timezone=Asia/Tokyoで登録したデータは+09で登録されるため、timezone=UTCで参照すると+00された値が表示されます。

```sql
SET SESSION timezone TO 'Asia/Tokyo';

INSERT INTO example (ts1, ts2, time1, time2) VALUES (
  '2020-01-01 00:00:00',
  '2020-01-01 00:00:00',
  '00:00:00',
  '00:00:00'
);

SELECT * FROM example;
         ts1         |          ts2           |  time1   |    time2
---------------------+------------------------+----------+-------------
 2020-01-01 00:00:00 | 2020-01-01 00:00:00+09 | 00:00:00 | 00:00:00+09

SET SESSION timezone TO 'UTC';

SELECT * FROM example;
         ts1         |          ts2           |  time1   |    time2
---------------------+------------------------+----------+-------------
 2020-01-01 00:00:00 | 2019-12-31 15:00:00+00 | 00:00:00 | 00:00:00+09
```

### AT TIME ZONE

[AT TIME ZONE](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT)の結果は、timestamp型のTIME ZONEあり/なしで結果が変わります。

```sql
SET SESSION timezone TO 'UTC';
INSERT INTO example (ts1, ts2) VALUES (
  '2020-01-01 00:00:00',
  '2020-01-01 00:00:00'
);
SELECT * FROM example;
         ts1         |          ts2           |  time1   |    time2
---------------------+------------------------+----------+-------------
 2020-01-01 00:00:00 | 2020-01-01 00:00:00+00 | 00:00:00 | 00:00:00+00
 
SELECT ts1 AT TIME ZONE 'Asia/Tokyo', 
       ts1 AT TIME ZONE 'UTC' AT TIME ZONE 'Asia/Tokyo', 
       ts2 AT TIME ZONE 'Asia/Tokyo'
FROM example;
        timezone        |      timezone       |      timezone
------------------------+---------------------+---------------------
 2019-12-31 15:00:00+00 | 2020-01-01 09:00:00 | 2020-01-01 09:00:00
```

- TIME ZONEなしのカラムにAT TIME ZONEを指定した場合、そのカラムを指定したTIME ZONEとして値を表示します。
- TIME ZONEなしのカラムにAT TIME ZONEを2つ指定した場合、1つ目のAT TIME ZONEでカラムのTIME ZONEを指定し（つまりTIME ZONEありのカラムと同じ）、2つ目のAT TIME ZONEでカラムの値を指定したTIME ZONEにシフトして表示します。
- TIME ZONEありのカラムにAT TIME ZONEを指定した場合、そのカラムを指定したTIME ZONEにシフトして表示します。

だんだんわからなくなってきたのでこの辺にしておきます。

## まとめ

- アプリケーションやDBクライアントからDBに接続する時は、TimeZoneを明示しよう
- TimeZoneを変換する関数の動きは、セッションのtimezoneとカラムのTIME ZONEの有無によって変化するので、期待する動きになっているか確認しよう
