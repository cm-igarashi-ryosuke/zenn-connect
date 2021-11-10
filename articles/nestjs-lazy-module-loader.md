---
title: "NestJSのLazyModuleLoaderを使って遅延読み込みしてみる"
emoji: "🦥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

NestJS の v8 で追加された `LazyModuleLoader` を使うと、Module の読み込み（プロバイダーのインスタンス作成）を必要になるまで遅延することができます。これによりアプリケーションの起動時間が早くなり、Cloud Run のようなサーバレスなコンテナ環境でも使いやすくなります。

`LazyModuleLoader` について、公式ドキュメントに分かりやすく書かれているのですが、読んだだけではいまいち実際の動きがイメージできなかったので、実際に試してみました。
https://docs.nestjs.com/fundamentals/lazy-loading-modules

## 1. プロジェクトの作成

NestJS のプロジェクトを新規作成します。

```bash
npm i -g @nestjs/cli
nest new sample-project

cd sample-project
```

作成が完了したら、アプリケーションが起動することを確認します。

```bash
$ npm run start:dev

[Nest] 96119  - 2021/11/10 15:39:08     LOG [NestFactory] Starting Nest application...
[Nest] 96119  - 2021/11/10 15:39:08     LOG [InstanceLoader] AppModule dependencies initialized +52ms
[Nest] 96119  - 2021/11/10 15:39:08     LOG [RoutesResolver] AppController {/}: +12ms
[Nest] 96119  - 2021/11/10 15:39:08     LOG [RouterExplorer] Mapped {/, GET} route +4ms
[Nest] 96119  - 2021/11/10 15:39:08     LOG [NestApplication] Nest application successfully started +3ms
```

次に、遅延読み込みにするモジュールを追加していきます。（この時点ではまだ通常のモジュールです。）

```bash
$ nest generate module cat
CREATE src/cat/cat.module.ts (81 bytes)
UPDATE src/app.module.ts (365 bytes)

$ nest generate controller cat
CREATE src/cat/cat.controller.spec.ts (478 bytes)
CREATE src/cat/cat.controller.ts (97 bytes)
UPDATE src/cat/cat.module.ts (166 bytes)

$ nest generate service cat
CREATE src/cat/cat.service.spec.ts (446 bytes)
CREATE src/cat/cat.service.ts (88 bytes)
UPDATE src/cat/cat.module.ts (240 bytes)
```

```typescript:src/cat/cat.module.ts
@Module({
  providers: [CatService],
  controllers: [CatController],
})
export class CatModule {}
```

`CatService` のインスタンスが作成されたタイミングが分かるように、コンストラクターにログを追加します。

```typescript:src/cat/cat.service.ts
@Injectable()
export class CatService {
  constructor() {
    console.log('init CatService');
  }

  run() {
    return 'nyan';
  }
}
```

`CatController` から `CatService` を呼び出すようにします。

```typescript:src/cat/cat.controller.ts
@Controller('cat')
export class CatController {
  constructor(private catService: CatService) {}

  @Get()
  index() {
    return this.catService.run();
  }
}
```

ここまでできたらアプリケーションを起動してみます。

```bash
$ npm run start:dev

[Nest] 8005  - 2021/11/10 15:54:18     LOG [NestFactory] Starting Nest application...
init CatService
[Nest] 8005  - 2021/11/10 15:54:18     LOG [InstanceLoader] AppModule dependencies initialized +57ms
[Nest] 8005  - 2021/11/10 15:54:18     LOG [InstanceLoader] CatModule dependencies initialized +0ms
[Nest] 8005  - 2021/11/10 15:54:18     LOG [RoutesResolver] AppController {/}: +6ms
[Nest] 8005  - 2021/11/10 15:54:18     LOG [RouterExplorer] Mapped {/, GET} route +4ms
[Nest] 8005  - 2021/11/10 15:54:18     LOG [RoutesResolver] CatController {/cat}: +1ms
[Nest] 8005  - 2021/11/10 15:54:18     LOG [RouterExplorer] Mapped {/cat, GET} route +0ms
[Nest] 8005  - 2021/11/10 15:54:18     LOG [NestApplication] Nest application successfully started +4ms
```

アプリケーションの開始時に `CatService` のインスタンスが作成されたことが確認できました。

## 2. モジュールを遅延読み込みする

では `CatService` を遅延読み込みにしてみます。

まず、`CatModule` の `provider` 部分を、新しく作った `CatLazyModule` に移します。

```typescript:src/cat/cat.module.ts
@Module({
  controllers: [CatController],
})
export class CatModule {}
```

```typescript:src/cat/cat.lazy.module.ts
@Module({
  providers: [CatService],
})
export class CatLazyModule {}
```

次に `CatController` で `LazyModuleLoader` をインジェクトします。
`CatService` の読み込みは、初回のリクエストが来てから行われるようにします。

```typescript:src/cat/cat.controller.ts
@Controller('cat')
export class CatController {
  private catService: CatService;
  constructor(private lazyModuleLoader: LazyModuleLoader) {}

  @Get()
  async index() {
    await this.lazyInit();
    return this.catService.run();
  }

  async lazyInit() {
    if(this.catService) return; // 初期化されていたら何もしない

    const { CatLazyModule } = await import('./cat.lazy.module');
    const moduleRef = await this.lazyModuleLoader.load(() => CatLazyModule);

    const { CatService } = await import('./cat.service');
    this.catService = moduleRef.get(CatService);
  }
}
```

アプリケーションを起動します。

```bash
$ npm run start:dev

[Nest] 15415  - 2021/11/10 16:04:25     LOG [NestFactory] Starting Nest application...
[Nest] 15415  - 2021/11/10 16:04:25     LOG [InstanceLoader] AppModule dependencies initialized +40ms
[Nest] 15415  - 2021/11/10 16:04:25     LOG [InstanceLoader] CatModule dependencies initialized +0ms
[Nest] 15415  - 2021/11/10 16:04:25     LOG [RoutesResolver] AppController {/}: +5ms
[Nest] 15415  - 2021/11/10 16:04:25     LOG [RouterExplorer] Mapped {/, GET} route +3ms
[Nest] 15415  - 2021/11/10 16:04:25     LOG [RoutesResolver] CatController {/cat}: +0ms
[Nest] 15415  - 2021/11/10 16:04:25     LOG [RouterExplorer] Mapped {/cat, GET} route +1ms
[Nest] 15415  - 2021/11/10 16:04:25     LOG [NestApplication] Nest application successfully started +2ms
```

まだ `CatService` は初期化されていません。

`http://localhost:3000/cat/` にリクエストすると、以下のログが出力されます。

```bash
init CatService
[Nest] 15415  - 2021/11/10 16:04:46     LOG [LazyModuleLoader] CatLazyModule dependencies initialized
```

この時点で、 `CatService` のインスタンスが作成されたことが確認できました。

## 3. その他の方法

ユースケースが少し違いますが、リクエストが来たときにプロバイダーのインスタンスを生成するだけで良ければ、`@Injectable`のパラメータで `scope` を `REQUEST` にするという方法もあります。`scope` を `REQUEST` にすると、リクエストのたびにインジェクトするインスタンスが生成・破棄されます。

ソースコードの状態は、1 の続きからと考えてください。

```typescript:src/cat/cat.service.ts
@Injectable({ scope: Scope.REQUEST })
export class CatService {
  constructor() {
    console.log('init CatService');
  }

  run() {
    return 'nyan';
  }
}
```

アプリケーションを起動します。

```bash
$ npm run start:dev

[Nest] 36104  - 2021/11/10 16:36:30     LOG [NestFactory] Starting Nest application...
[Nest] 36104  - 2021/11/10 16:36:30     LOG [InstanceLoader] AppModule dependencies initialized +38ms
[Nest] 36104  - 2021/11/10 16:36:30     LOG [InstanceLoader] CatModule dependencies initialized +0ms
[Nest] 36104  - 2021/11/10 16:36:30     LOG [RoutesResolver] AppController {/}: +5ms
[Nest] 36104  - 2021/11/10 16:36:30     LOG [RouterExplorer] Mapped {/, GET} route +2ms
[Nest] 36104  - 2021/11/10 16:36:30     LOG [RoutesResolver] CatController {/cat}: +0ms
[Nest] 36104  - 2021/11/10 16:36:30     LOG [RouterExplorer] Mapped {/cat, GET} route +1ms
[Nest] 36104  - 2021/11/10 16:36:30     LOG [NestApplication] Nest application successfully started +2ms
```

遅延読み込みと同様に、まだ `CatService` は初期化されていません。

`http://localhost:3000/cat/` にリクエストすると、以下のログが出力されます。

```bash
init CatService
```

## まとめ

Module を遅延読み込みする方法が確認できました。ちょっとまだ Module（DI）のメリットが分かっていないので、ここまでするくらいなら Module を使わなくても良いのでは？（例えば Controller の constructor で普通に new するでも良い？）とも思ってしまうのですが...ご意見ありましたら教えて下さい！
