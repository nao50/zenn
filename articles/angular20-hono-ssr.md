---
title: "Angular20 を Hono で SSR する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular", "hono", "typescript", "javascript"]
published: false
---

# はじめに
[Hono](https://hono.dev/)と[Angular20](https://angular.dev/)でServer Side Rendering(SSR)します。 
Angular18 でSSRした記事は以下にあります。  

https://zenn.dev/nao50/articles/angular18-hono-ssr


# Angular を Hono で SSR
Angularはプロジェクト生成時にSSRを指定することでexpressベースのサーバーコードが生成されます。

最近は以下の記事を参考にnpxでプロジェクトを作成しました。  

https://kasaharu.hatenablog.com/entry/20241222/1734839047

```sh
$ npx @angular/cli@latest new <project-name> --ssr
```

::::details 生成されたサーバーコード
```ts:server.ts
import {
  AngularNodeAppEngine,
  createNodeRequestHandler,
  isMainModule,
  writeResponseToNodeResponse,
} from '@angular/ssr/node';
import express from 'express';
import { join } from 'node:path';

const browserDistFolder = join(import.meta.dirname, '../browser');

const app = express();
const angularApp = new AngularNodeAppEngine();

/**
 * Example Express Rest API endpoints can be defined here.
 * Uncomment and define endpoints as necessary.
 *
 * Example:
 * ```ts
 * app.get('/api/{*splat}', (req, res) => {
 *   // Handle API request
 * });
 * ```
 */

/**
 * Serve static files from /browser
 */
app.use(
  express.static(browserDistFolder, {
    maxAge: '1y',
    index: false,
    redirect: false,
  }),
);

/**
 * Handle all other requests by rendering the Angular application.
 */
app.use((req, res, next) => {
  angularApp
    .handle(req)
    .then((response) =>
      response ? writeResponseToNodeResponse(response, res) : next(),
    )
    .catch(next);
});

/**
 * Start the server if this module is the main entry point.
 * The server listens on the port defined by the `PORT` environment variable, or defaults to 4000.
 */
if (isMainModule(import.meta.url)) {
  const port = process.env['PORT'] || 4000;
  app.listen(port, (error) => {
    if (error) {
      throw error;
    }

    console.log(`Node Express server listening on http://localhost:${port}`);
  });
}

/**
 * Request handler used by the Angular CLI (for dev-server and during build) or Firebase Cloud Functions.
 */
export const reqHandler = createNodeRequestHandler(app);

```
::::

[Angular18](https://zenn.dev/nao50/articles/angular18-hono-ssr)の時とはずいぶん変わりましたね...  
今回はこのexpressベースのサーバーコードをHonoベースに置き換えていきます。  

## Hono の設定
まずはHonoをインストールします。  

```sh
$ npm i hono
$ npm i @hono/node-server
```

`server.ts`にSSR含むサーバーコードを書きます。
試行錯誤の結果、`AngularNodeAppEngine`ではなく`AngularAppEngine`を使うことでうまくSSRできました。

https://angular.jp/guide/ssr#non-node-js

```ts:server.ts
import { Hono } from 'hono';
import { serve } from '@hono/node-server';
import { serveStatic } from '@hono/node-server/serve-static';

import { isMainModule } from '@angular/ssr/node';
import { AngularAppEngine, createRequestHandler } from '@angular/ssr';
import { join } from 'node:path';

export const app = new Hono()

// json を返す hello API
app.get('/hello', (c) => c.json({
  hello: 'world!',
}))

/**
 * 静的ファイルのパスを指定
 */
app.use(
  '*',
  serveStatic({
    root: join(import.meta.dirname, '../browser'),
    onFound: (path, c) => {
      c.header('Cache-Control', `public, immutable, max-age=31536000`);
    },
    onNotFound: () => {
    },
  })
);

/**
 * SSRするパスを指定
 */
app.use('*', async (c, next) => {
  const angularApp = new AngularAppEngine();
  const response = await angularApp.handle(c.req.raw);
  if (response) {
    return response;
  }
  return next();
});

/**
 * 404  Not found
 */
app.notFound((c) => {
  return c.text('404 - Not found', 404);
});

/**
 * 500 Internal Server Error
 */
app.onError((error, c) => {
  console.error(`${error}`);
  return c.text('Internal Server Error', 500);
});

/**
 * サーバーのエントリーポイント
 */
if (isMainModule(import.meta.url)) {
  const port = Number(process.env['PORT'] || 4000);
  serve({
    fetch: app.fetch,
    port,
  }, (info) => {
    console.log(`Hono server listening on http://localhost:${info.port}`);
  });
}

/**
 * Request handler used by the Angular CLI (for dev-server and during build) or Firebase Cloud Functions.
 */
export const reqHandler = createRequestHandler(app.fetch);
```

## Angular の設定
Angular側は簡単にOnInitでhello APIからデータをGETするコードとします。  


```ts:app.component.ts
import { Component, inject, OnInit  } from '@angular/core';
import { JsonPipe } from '@angular/common';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-root',
  imports: [JsonPipe],
  template: `
    <div>message: {{ res | json }}</div>
  `
})
export class App implements OnInit {
  res: any;
  #http = inject(HttpClient);

  ngOnInit() {
    this.#http.get<any>('http://localhost:4000/hello').subscribe((data) => {
      this.res = data;
    });
  }
}
```

## ビルドと実行
Angularは `ng serve`コマンドでもSSRできるようですが、ビルドと実行をそれぞれ実行します。  

```sh
$ npm run build
$ npm run serve:ssr:<YOUR PROJECT>
```

Dev Consoleを開きながら `http://localhost:4000/` にアクセスすると以下のようにSSR済のHTMLがレンダリングされてきます。  

![angular20 hono ssr 01](/images/angular18-hono-ssr-01.png)

# まとめ
[Hono](https://hono.dev/)と[Angular](https://angular.dev/)20でSSRする方法をまとめました。  

コードはこちら

https://github.com/nao50/angular20-hono-ssr