---
title: Cloud FunctionsをGitHub PackagesのContainer Registoryで配布してみる
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudfunctions", "github", "githubactions", "githubpackages"]
published: true
---

長いタイトルの通り、色々と試してみたかった実験的な記録です。

## 背景

とあるサービスから呼び出している Cloud Functions（HTTP でトリガーされる関数）があり、ローカル実行やテストの時も GCP 上の Cloud Functions を呼び出していたので、Cloud Functions をコンテナ化してローカルで起動してアクセスできるようにできないかなと思ったのがきっかけです。

## 環境

- macOS Catalina (10.15.7)
- Docker Engine (20.10.5)
- Nodejs (14.x)

## Cloud Functions の関数をローカル起動用にビルドする

まずは Cloud Functions の関数を HTTP でリクエストできるようにしなくてはなりません。最初は Express.js などでラップして Web サーバにしようかと思ったんですが、実は Cloud Functions では[Function Frameworks](https://cloud.google.com/functions/docs/functions-framework?hl=ja)という OSS ライブラリが使われているということが[公式ドキュメント](https://cloud.google.com/functions/docs/running/function-frameworks?hl=ja)に書かれていました。

さらに関数をローカルで実行可能なコンテナにビルドする[pack](https://cloud.google.com/functions/docs/building/pack?hl=ja)というツールまで提供されています。これを使うことで、関数を Function Frameworks でラップしたコンテナを自動的に作ってくれます。なんて便利なんでしょう。

ちなみにこれも調べているうちに知ったんですが、 Cloud Functions の関数のコンテナは GCP の Container Registory に配置されており、ユーザからもアクセス可能になっています。これをローカルに pull して実行することもできました。

### 1. 関数を作成

今回作成するのは HTTP でトリガーされる関数です。関数の実装は公式ドキュメントにある[サンプル](https://cloud.google.com/functions/docs/writing/http?hl=ja#sample_usage)を利用します。

まずは npm を初期化しモジュールをインストールします。

```bash
npm init
npm install escape-html --save
```

サンプル実装を配置します。

```javascript:index.js
const escapeHtml = require('escape-html');

/**
 * HTTP Cloud Function.
 *
 * @param {Object} req Cloud Function request context.
 *                     More info: https://expressjs.com/en/api.html#req
 * @param {Object} res Cloud Function response context.
 *                     More info: https://expressjs.com/en/api.html#res
 */
exports.helloHttp = (req, res) => {
  res.send(`Hello ${escapeHtml(req.query.name || req.body.name || 'World')}!`);
};
```

### 2. pack でコンテナをビルド

pack の[インストール手順](https://buildpacks.io/docs/tools/pack/)を参照し、brew でインストールします。本当はコンテナで実行したかったのですが、docker.sock のパーミッションエラー実行できませんでした。（sudo をつけてもダメで、いったん諦めました。）

```bash
brew install buildpacks/tap/pack
```

公式ドキュメントの[手順](https://cloud.google.com/functions/docs/building/pack?hl=ja)に従い、pack でビルドします。

```bash
pack build \
  --builder gcr.io/buildpacks/builder:v1 \
  --env GOOGLE_RUNTIME=nodejs \
  --env GOOGLE_FUNCTION_SIGNATURE_TYPE=http \
  --env GOOGLE_FUNCTION_TARGET=helloHttp \
  cloud-functions-sample
```

### 3. コンテナを起動

ビルドしたコンテナを起動します。ポート 8080 をコンテナのポートとバインドします。

```bash
docker run --rm -p 8080:8080 cloud-functions-sample
```

localhost:8080 にアクセスすることで、関数からの応答が返ることがわかります！

```bash
$ curl http://localhost:8080
Hello World!
```

ここまでで作成したサンプル関数を GitHub に push しておきます。

## GitHub Actions でビルドして GitHub Packages の Container Registory に登録する

次に GitHub Actions の定義を作成します。

```yaml:.github/workflows/build.yml
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [master]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: cloud-functions-sample

    strategy:
      matrix:
        node-version: [14.x]

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node-version }}

      - name: Run npm install
        run: npm install

      - name: Install pack-cli
        run: |
          sudo add-apt-repository ppa:cncf-buildpacks/pack-cli \
            && sudo apt-get update \
            && sudo apt-get install pack-cli

      - name: Run pack build
        run: |
          pack build \
            --builder gcr.io/buildpacks/builder:v1 \
            --env GOOGLE_RUNTIME=nodejs \
            --env GOOGLE_FUNCTION_SIGNATURE_TYPE=http \
            --env GOOGLE_FUNCTION_TARGET=helloHttp \
            $IMAGE_NAME

      - name: Log in to GitHub Docker Registry
        uses: docker/login-action@v1
        with:
          registry: docker.pkg.github.com
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push a container image to GitHub Container Registory
        run: |
          docker tag $IMAGE_NAME docker.pkg.github.com/$GITHUB_REPOSITORY/$IMAGE_NAME:latest \
            && docker push docker.pkg.github.com/$GITHUB_REPOSITORY/$IMAGE_NAME:latest

```

pack の[インストール手順](https://buildpacks.io/docs/tools/pack/)に Ubuntu へのインストール方法もあったので、そのままコピペでインストールすることができました。

Container Registry への push 手順は他の Registry と同じような感じです。認証には `GITHUB_TOKEN` か `Personal Access Token` を利用するのですが、GitHub Actions の場合は自動で `GITHUB_TOKEN` が発行されているのでクレデンシャルの管理が不要なのがとても便利です！

これで Container Registry へのコンテナの登録も完了しました。

## GitHub Packages の Container Registory から pull して実行する

最後に登録したコンテナをローカルに pull して実行してみます。

```bash
$ docker pull docker.pkg.github.com/bisque33/github-container-integration/cloud-functions-sample:latest
Error response from daemon: Head https://docker.pkg.github.com/v2/bisque33/github-container-integration/cloud-functions-sample/manifests/latest: no basic auth credentials
```

公開リポジトリだからといって認証が不要という訳にはいかないようです。

認証のため、個人設定の Developer settings から[Personal access tokens](https://github.com/settings/tokens)を生成します。権限は `read:packages` のみで OK です。

発行されたトークンを `~/.github/pat-only-read-packages.txt` に配置しました。それを使って docker login を行い、もう一度 docker pull をしてみます。

```bash
$ cat ~/.github/pat-only-read-packages.txt | docker login https://docker.pkg.github.com -u bisque33 --password-stdin
Login Succeeded

$ docker pull docker.pkg.github.com/bisque33/github-container-integration/cloud-functions-sample:latest
WARNING: ⚠️ Failed to pull manifest by the resolved digest. This registry does not
	appear to conform to the distribution registry specification; falling back to
	pull by tag.  This fallback is DEPRECATED, and will be removed in a future
	release.  Please contact admins of https://docker.pkg.github.com. ⚠️

latest: Pulling from bisque33/github-container-integration/cloud-functions-sample
c81a311ac26a: Already exists
05c49db37ee9: Already exists
3c2cba919283: Already exists
91af061cbbcd: Already exists
45d7126bebce: Already exists
cf6b7f7a08d9: Pull complete
09b4558c7e27: Pull complete
8f0d61e10f8d: Pull complete
785ed11a713f: Pull complete
476a9b260057: Pull complete
769b7fb5f768: Pull complete
9e1b79ed05fb: Pull complete
Digest: sha256:1e6d4e80cd55be14300f5cf97955d6749712c2d0feae9284917c27b68ae95981
Status: Downloaded newer image for docker.pkg.github.com/bisque33/github-container-integration/cloud-functions-sample:latest
docker.pkg.github.com/bisque33/github-container-integration/cloud-functions-sample:latest

$ docker images | grep cloud-functions-sample
docker.pkg.github.com/bisque33/github-container-integration/cloud-functions-sample          latest                    dea60fbcdd4b   41 years ago    224MB
```

WARNING が表示されましたがいったん置いておいて、起動すると正常に動作することが確認できました！

```bash
docker run --rm -p 8080:8080 docker.pkg.github.com/bisque33/github-container-integration/cloud-functions-sample:latest
```

```bash
$ curl http://localhost:8080
Hello World!
```

## おわりに

作成したサンプルのリポジトリは[こちら](https://github.com/bisque33/github-container-integration)に公開しています。

GitHub Actions と Github Packages の Container Registory を連携することで簡単にイメージをレジストリに登録でき、プライベートなイメージの共有がやりやすくなったのではないでしょうか！
