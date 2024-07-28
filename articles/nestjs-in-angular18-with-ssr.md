---
title: "Nestjs で @angular/ssr すると ES Module 周りでエラーが出る話"
emoji: "🦁"
type: "tech"
topics: ["nestjs", "angular", "typescript"]
published: false
---

# はじめに
[Nestjs](https://nestjs.com/) は2024年7月現在、ES Moduleの対応がないので @angular/ssr がそのまま import できないよ。という話です。

https://github.com/nestjs/nest/issues/13319


# やりたい構成
[Angular](https://angular.dev/)は標準のCLIから生成されるSSRのコードはexpressベースです。  
ただし、@angular/ssr 自体にフレームワーク依存があるわけではないので、NestJSでも使いたくなるわけです。

今回はNestjsのプロジェクト内にAngularのプロジェクトを入れ込み `localhost:3000/` にアクセスされた時にAngularのSSRされたWebページを返す構成を考えていきます。  

```sh
$ tree
.
├── README.md
├── dist
├── nest-cli.json
├── node_modules
├── package-lock.json
├── package.json
├── src
│   ├── angular-frontend       # ここにAngularのファイル
│   ├── app.controller.spec.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   └── main.ts
├── test
├── tsconfig.build.json
└── tsconfig.json
```

# Nestjs 内に Angular のプロジェクトを作る
Nestjs も Angular も CLI があるので、コードを生成していきます。  

```sh
$ nest new nestjs-in-angular18-with-ssr
$ cd nestjs-in-angular18-with-ssr/src
$ ng new angular-frontend
```

Nestjs と Angular では Typescript の compilerOptions が何から何まで異なるため、Nestjs側の `tsconfig.build.json` に対して `exclude` の追加を行います。  

```json: tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts", "src/angular-frontend"]
}
```

# Nestjs に Angular SSR の設定
Nestjs で Angular SSR の設定を行います。
必要なモジュールをインストールします。  

```sh
$ cd .. # nestjs-in-angular18-with-ssr のプロジェクトルートでインストール
$ npm install --save @nestjs/serve-static
$ npm i @angular/ssr
```

基本的には Angular CLI が生成する `server.ts` のコードを Nestjs 側に移植するイメージです。  
Nestjsの `main.ts` を修正します。  

```typescript: main.ts
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  app.set('view engine', 'html');
  app.set('views', './angular-frontend/dist/angular-frontend/browser');

  await app.listen(3000);
}
bootstrap();
```

Nestjs 側の `app.module.ts` に static コンテンツのパスを指定しておきます。  

```typescript: app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, 'angular-frontend', 'dist', 'angular-frontend', 'browser'),
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

いよいよ、Nestjs に Angular SSR の設定を追加します。  
こちらも Angular CLI が生成する `server.ts` のコードを Nestjs 側に移植します。  
簡略化のため controller に全てのコードを書いています。  

```typescript: app.controller.ts
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';
import { AppService } from './app.service';
// 
import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import bootstrap from './angular-frontend/src/main.server';
import { dirname, join, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';

@Controller()
export class AppController {
  commonEngine = new CommonEngine();

  constructor(private readonly appService: AppService) {}

  @Get('')
  frontend(@Req() req: Request): any {
    const serverDistFolder = dirname(fileURLToPath('./angular-frontend/dist/angular-frontend/server'));
    const browserDistFolder = resolve(serverDistFolder, '../browser');
    const indexHtml = join(serverDistFolder, 'index.server.html');

    const { protocol, originalUrl, baseUrl, headers } = req;
  
    this.commonEngine.render({
      bootstrap,
      documentFilePath: indexHtml,
      url: `${protocol}://${headers.host}${originalUrl}`,
      publicPath: browserDistFolder,
      providers: [{ provide: APP_BASE_HREF, useValue: baseUrl }],
    });
  }

  @Get('hello')
  getHello(): string {
    return this.appService.getHello();
  }
}
```

# 発生するエラー
Nestjs の開発環境を起動すると以下のようなエラーが吐かれます。  

```sh
$ npm run start

> nestjs-in-angular18-with-ssr@0.0.1 start
> nest start

node:internal/modules/cjs/loader:1205
    throw new ERR_REQUIRE_ESM(filename, true);
    ^

Error [ERR_REQUIRE_ESM]: require() of ES Module /<PROJECT PATH>/nestjs-in-angular18-with-ssr/node_modules/@angular/common/fesm2022/common.mjs not supported.
Instead change the require of /<PROJECT PATH>/nestjs-in-angular18-with-ssr/node_modules/@angular/common/fesm2022/common.mjs to a dynamic import() which is available in all CommonJS modules.
    at Object.<anonymous> (/<PROJECT PATH>/nestjs-in-angular18-with-ssr/dist/app.controller.js:18:18) {
  code: 'ERR_REQUIRE_ESM'
}
```

Nestjs は ESM のサポートが現時点でないようです。

https://github.com/nestjs/nest/issues/13319

また、Nestjs は [nestjs/ng-universal](https://github.com/nestjs/ng-universal/issues/1125) というライブラリを提供していましたが、こちらも Angular17 以降、動かない状況です。  

https://github.com/nestjs/ng-universal/issues/1125


# まとめ
Nestjs は2024年7月現在、ES Moduleの対応がないので @angular/ssr がそのまま import できないよ。という話でした。

ご参考までに作ったレポジトリはこちら
https://github.com/nao50/nestjs-in-angular18-with-ssr