---
title: "Angular18 と Hono で SSR と RPC を試す"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular", "hono", "typescript", "javascript"]
published: true
---

# はじめに
[Hono](https://hono.dev/)と[Angular](https://angular.dev/)でSSRと[RPC機能](https://hono.dev/docs/guides/rpc)を試してみます。  

Angularはv19でSSR/SSG周りの[大きなアップデート](https://zenn.dev/lacolaco/articles/angular-v19-prerendering)を予定しているようですが、今回はv18で素振りをしておきましょう。

# Angular を Hono で SSR
Angularはプロジェクト生成時にSSRを指定することでexpressベースのサーバーコードが生成されます。

```sh
$ ng new --ssr
```

::::details 生成されたサーバーコード
```ts:server.ts
import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import express from 'express';
import { fileURLToPath } from 'node:url';
import { dirname, join, resolve } from 'node:path';
import bootstrap from './src/main.server';

// The Express app is exported so that it can be used by serverless Functions.
export function app(): express.Express {
  const server = express();
  const serverDistFolder = dirname(fileURLToPath(import.meta.url));
  const browserDistFolder = resolve(serverDistFolder, '../browser');
  const indexHtml = join(serverDistFolder, 'index.server.html');

  const commonEngine = new CommonEngine();

  server.set('view engine', 'html');
  server.set('views', browserDistFolder);

  // Example Express Rest API endpoints
  // server.get('/api/**', (req, res) => { });
  // Serve static files from /browser
  server.get('**', express.static(browserDistFolder, {
    maxAge: '1y',
    index: 'index.html',
  }));

  // All regular routes use the Angular engine
  server.get('**', (req, res, next) => {
    const { protocol, originalUrl, baseUrl, headers } = req;

    commonEngine
      .render({
        bootstrap,
        documentFilePath: indexHtml,
        url: `${protocol}://${headers.host}${originalUrl}`,
        publicPath: browserDistFolder,
        providers: [{ provide: APP_BASE_HREF, useValue: baseUrl }],
      })
      .then((html) => res.send(html))
      .catch((err) => next(err));
  });

  return server;
}

function run(): void {
  const port = process.env['PORT'] || 4000;

  // Start up the Node server
  const server = app();
  server.listen(port, () => {
    console.log(`Node Express server listening on http://localhost:${port}`);
  });
}

run();

```
::::

今回はこのexpressベースのサーバーコードをHonoベースに置き換えていきます。  

## Hono の設定
まずはHonoをインストールします。  

```sh
$ npm i hono
$ npm i @hono/node-server
```

Angular v18ではデフォルトで`prerender:true`となってますが、今回はSSRを確かめたいので`prerender:false`としておきます。  

```json:angular.json
{
  "projects": {
    "[YOUR PROJECT]": {
      "architect": {
        "build": {
          "options": {
            "prerender": false,
          },
        },
      },
    },
  },
}
```

`server.ts`にSSR含むサーバーコードを書きます。
SSRの動作確認用に `/hello` という雑なjsonを返すAPIも切っておきましょう。  

```ts:server.ts
import { Hono } from 'hono'
import { serve } from '@hono/node-server'
import { serveStatic } from '@hono/node-server/serve-static'

import { APP_BASE_HREF } from '@angular/common';
import { CommonEngine } from '@angular/ssr';
import { fileURLToPath } from 'node:url';
import { dirname, join, resolve } from 'node:path';
import bootstrap from './src/main.server';


export function app(): Hono {
  const app = new Hono()
  const serverDistFolder = dirname(fileURLToPath(import.meta.url));
  const browserDistFolder = resolve(serverDistFolder, '../browser');
  const indexHtml = join(serverDistFolder, 'index.server.html');

  const commonEngine = new CommonEngine();

  // json を返す hello API
  app.get('/hello', (c) => c.json({
    hello: 'world!',
  }))

  // SSRするパスを指定
  app.get('/', (c) => {
    const url = c.req.url;
    return commonEngine
      .render({
        bootstrap,
        documentFilePath: indexHtml,
        url: `${url}`,
        publicPath: browserDistFolder,
        providers: [{ provide: APP_BASE_HREF, useValue: '' }],
      })
      .then((html) => c.html(html))
      .catch((err) => {
        console.error('Error:', err);
      });
  })

  // 静的ファイルのパスを指定
  app.use('/*', serveStatic({
    root: './dist/zenn-hono-angular/browser',
    index: 'index.html',
    onNotFound: (path, c) => {
      console.log(`${path} is not found, request to ${c.req.path}`)
    },
  }))

  return app;
}

function run(): void {
  const server = app();
  serve({
    fetch: server.fetch,
    port: +process.env['PORT']! || 4000,
  })
}

run();
```

## Angular の設定
Angular側は簡単にOnInitでhello APIからデータをGETするコードとします。  


```ts:app.component.ts
import { Component, inject, OnInit  } from '@angular/core';
import { JsonPipe } from '@angular/common';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [JsonPipe],
  template: `
    <div>message: {{ res | json }}</div>
  `
})
export class AppComponent implements OnInit {
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

![angular18 hono ssr 01](/images/angular18-hono-ssr-01.png)


# Angular を Hono で RPC
せっかくなのでHonoの至高な機能のRPCも試してみましょう。  
[HonoのRPC機能](https://hono.dev/docs/guides/rpc)はざっくり言うとサーバ側とクライアント側でAPIの型を共有できる機能です。  

まずはHonoから提供されているzodのラッパーをインストールします。  

```sh
$ npm i @hono/zod-validator
```

サーバーサイドで以下のようなデータ型を定義し `app.route('/', posts)` としてルーティングに組み込みます。  
最後にデータ型を任意の名前でexportします。  

```ts:server.ts
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

...

const posts = new Hono()
  .post(
    '/posts',
    zValidator(
      'form',
      z.object({
        title: z.string(),
        body: z.string(),
      })
    ),
    (c) => {
      return c.json(
        {
          ok: true,
          message: 'Created!',
        },
        201
      )
    }
  )

export function app(): Hono {
  ...
  app.route('/', posts)
  ...
}

export type PostsType = typeof posts
```

Angular側でその型をimportすることができ、エディタ上も補完が効きます。  
これは便利ですね。  

```ts:app.component.ts
import { PostsType } from '../../server'
import { hc } from 'hono/client'

...

  async ngOnInit() {
    const client = hc<PostsType>('http://localhost:4000')
    const res = await client.posts.$post({
      form: {
        title: 'Hello',
        body: 'Hono is a cool project',
      },
    })
    if (res.ok) {
      const data = await res.json()
      console.log(data.message)
    }
  }


```

一点注意として、上記コードではサーバサイドとクライアントサイドで2回 `/posts` へのリクエストが走ってしまします。  

先ほどのAngularから提供される `import { HttpClient } from '@angular/common/http';` では内部的に2度リクエストが発火しない仕組みが入っていますが、`import { hc } from 'hono/client'` にはないので、[TransferState](https://angular.dev/api/core/TransferState)などを使って状態管理するなどの工夫が必要そうです。  

https://angular.dev/api/core/TransferState

# まとめ
[Hono](https://hono.dev/)と[Angular](https://angular.dev/)でSSRと[RPC機能](https://hono.dev/docs/guides/rpc)を試してみました。

コードはこちら

https://github.com/nao50/zenn-hono-angular