---
title: "YAMLの型仕様およびjs-yamlにおけるtimestampの解析仕様について"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [javascript, yaml]
published: true
---

## まえがき

JavaScript製のYAMLパーサーライブラリである[js-yaml](https://github.com/nodeca/js-yaml)でYAMLファイルから日時を表す項目を読み込んだところ、値のフォーマットにより**文字列として読み込まれる場合**と**Dateとして読み込まれる場合**がある事がわかりました。

本記事では、YAMLパーサーがどのように型を区別しているのかと、文字列として読み取るワークアラウンドについて記載します。

## YAMLの仕様

今回調査するまで知らなかったのですが、YAMLの仕様には[スカラー型](https://yaml.org/type/)が定義されています。なぜかというと、YAMLはテキストファイルであると同時に、アプリケーションの**ネイティブデータ構造を表す**ためのデータでもあるからです。（※ただし型をどこまでサポートするかどうかはパーサーの実装次第です。詳しくは[追記](https://zenn.dev/bisque/articles/yaml-type-spec#(%E8%BF%BD%E8%A8%98)-yaml%E3%81%AE%E4%BB%95%E6%A7%98%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8Bschema%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)を読んでください。）

> YAML is both a text format and a method for presenting any native data structure in this format.

引用: [https://yaml.org/spec/1.2.2/#processes-and-models](https://yaml.org/spec/1.2.2/#processes-and-models)

YAMLパーサーは、YAMLの型仕様に則り、値をネイティブ型へと変換するのが本来の望ましい働きであるわけですね。

### 型の区別

JavaScript製のYAMLパーサーライブラリである[js-yaml](https://github.com/nodeca/js-yaml)が、値を[timestamp](https://yaml.org/type/timestamp.html)あるいは文字列として判定する条件を見ていきます。（実装はパーサーにより異なる可能性があります。）

#### 1. Tagに `!!timestamp` が指定されている

YAMLにはTagという仕様があり、値の前に `!!タイプ名` を記述することで、その値の型を**明示的に示す**ことができます。

```yaml
date: !!timestamp 2022-05-28
```

```
2022-05-28T00:00:00.000Z
```

Tagが指定されていて、その型としてパースできない場合、パーサーはYAMLを異常なデータとみなします。

```yaml
date: !!timestamp 2022/05/28
```

```
YAMLException: cannot resolve a node with !<tag:yaml.org,2002:timestamp> explicit tag at line 9, column 29:
    date: !!timestamp 2022/05/28
```

#### 2. 正規表現にマッチする

Tagがなくても、パターンマッチにより**暗黙的**に型を判断されます。

パターンにマッチする場合: Dateとして解析されます。

```yaml
date: 2022-05-28
```

```
2022-05-28T00:00:00.000Z
```

パターンにマッチしない場合: 文字列として解析されます。

```yaml
date: 2022/05/28
```

```
2022/05/28
```

#### 3. 文字列として記述されている

文字列として**明示的に示されている**場合は、文字列として解析されます。

クォートで囲っている場合:

```yaml
date: '2022-05-28'
```

```
2022-05-28
```

`|`でマルチライン指定している場合:

```yaml
date: | 
  2022-05-28
```

```
2022-05-28
```

## timestampを文字列として読み取る

アプリケーションの都合により、YAMLファイルのtimestamp型の値を文字列として解析したい場合があります。

ここからは[js-yaml](https://github.com/nodeca/js-yaml)のワークアラウンドになります。

js-yamlは[読み込みオプション](https://github.com/nodeca/js-yaml#load-string---options-)でターゲットとするschemaを指定することができます。デフォルトで用意されているschemaは以下の通りです。

- `DEFAULT_SCHEMA`: YAMLで定義されたすべての型をネイティブ型として解析する
- `CORE_SCHEMA(JSON_SCHEMA)`: JSONがサポートする型をネイティブ型として解析する
- `FAILSAFE_SCHEMA`: すべて文字列として解析する

この内、`CORE_SCHEMA(JSON_SCHEMA)`、つまりJSONは、timestampをサポートしていないので、YAMLファイルのtimestamp型は文字列として解析します。

ただしtimestamp以外にもJSONがサポートしていない型はすべて文字列として解析されてしまうことに注意してください。もしいずれのschemaでも要求にマッチしない場合、独自でschemaを作ることも[できるよう](https://github.com/nodeca/js-yaml/issues/161#issuecomment-72711349)です。

## (追記) YAMLの仕様におけるschemaについて

後になって気付いたのですが、これらのschemaはYAMLパーサーが実装すべき仕様として、[YAMLの仕様に定義](https://yaml.org/spec/1.2-old/spec.html#Schema)されていました。せっかくなので軽く確認しておきたいと思います。

### Failsafe Schema

> The failsafe schema is guaranteed to work with any YAML document. It is therefore the recommended schema for generic YAML tools. A YAML processor should therefore support this schema, at least as an option.

引用: [https://yaml.org/spec/1.2-old/spec.html#id2802346](https://yaml.org/spec/1.2-old/spec.html#id2802346)

Failsafe SchemaはどのようなYAMLでも機能することが保証されているschemaです。js-yamlではすべてをstringとして読み込むことでこのschemaをサポートしているのですね。

### JSON Schema

> The JSON schema is the lowest common denominator of most modern computer languages, and allows parsing JSON files. A YAML processor should therefore support this schema, at least as an option. It is also strongly recommended that other schemas should be based on it.

引用: [https://yaml.org/spec/1.2-old/spec.html#id2803231](https://yaml.org/spec/1.2-old/spec.html#id2803231)

JSON SchemaはJSONとの互換性があり、他のスキーマのベースとなることが推奨されています。

### Core Schema

> The Core schema is an extension of the JSON schema, allowing for more human-readable presentation of the same types. This is the recommended default schema that YAML processor should use unless instructed otherwise. It is also strongly recommended that other schemas should be based on it.

引用: [https://yaml.org/spec/1.2-old/spec.html#id2804923](https://yaml.org/spec/1.2-old/spec.html#id2804923)

Core SchemaはJSON Schemaの拡張であり、ヒューマン・リーダブルなプレゼンテーションを可能にするということです。

どういうことかというと、例えばJSON Schemaではnull型は `null` しか許容されませんが、Core Schemaでは `null`, `Null`, `NULL`, `~` がnull型として解析されます。

ちなみに、js-yamlではJSON Schemaの厳密さはサポートされておらず、Core Schemaと同じ基準で実装されているようです。

https://github.com/nodeca/js-yaml/blob/master/lib/schema/json.js#L1-L6

さらに、Core SchemaがYAMLプロセッサー（パーサー）が使用する推奨のデフォルトschemaであると記述されています。js-yamlは更に拡張されたschemaがデフォルトなので、このあたりが少し標準仕様と違いますね。

js-yamlのデフォルトschemaのコメントにも、このようにCore Schemaを拡張したものであると説明されています。

https://github.com/nodeca/js-yaml/blob/master/lib/schema/default.js#L1-L5

### Other Schemas

> It is strongly recommended that such schemas be based on the core schema defined above. In addition, it is strongly recommended that such schemas make as much use as possible of the the YAML tag repository at http://yaml.org/type/. This repository provides recommended global tags for increasing the portability of YAML documents between different applications.

引用: [https://yaml.org/spec/1.2-old/spec.html#id2805770](https://yaml.org/spec/1.2-old/spec.html#id2805770)

これらのschema以外にも、Core SchemaをベースにしつつTagを用いて言語特有のデータ構造を拡張したschemaにしても良いと書いてあります。js-yamlのデフォルトschemaは、まさにこの[Type](http://yaml.org/type/)の仕様をサポートしているわけですね。

## おわり

雰囲気で使ってきたYAMLの奥深さを少しだけ垣間見ることができました。