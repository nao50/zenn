---
title: "Angular20 の SSR 時の HttpClient を考える"
emoji: "🅰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular", "hono", "typescript", "javascript"]
published: true
---

# はじめに
前回、[Hono](https://hono.dev/)と[Angular20](https://angular.dev/)でSSRする方法をまとめました。  

https://zenn.dev/nao50/articles/angular20-hono-ssr

今回はAngularのSSR時のHTTP Clientを比較していこうと思います。  

 - 'hono/client' の `hc` を使った[RPC](https://hono.dev/docs/guides/rpc)を使う
 - '@angular/common/http' の `HttpClient` を使う
 - '@angular/common/http' の `httpResource` を使う

# 'hono/client' の `hc` を使ったRPC機能を使う
Honoにはサーバー側で定義したAPI情報（schemaやroute情報）をフロントエンド側に伝搬する[RPC](https://hono.dev/docs/guides/rpc)という強力な機能があります。  
この機能は[zod](https://zod.dev/)など使い慣れたバリデータと組み合わせリクエスト/レスポンスのschemaの検証をすることができたり、どのAPIパスに対してどのレスポンスコードを返すか。などの検証までを行ってくれます。  

少し冗長ですが、以下のように schema → route → api を定義してRPC機能を体感しましょう。  

## hono rpc の schema
まずはschemaを作成します。  
[ドキュメント](https://hono.dev/docs/guides/rpc#server)の通り`zValidator` でも良いのですが、今回は `@hono/zod-openapi` を使いましょう。  

```ts:health.schema.ts
import { z } from '@hono/zod-openapi';

export const healthSchema = z.object({
  status: z.string().describe('Status of the health check'),
  message: z.string().describe('Message from the health check'),
  timestamp: z.iso.datetime().describe('Timestamp of the health check'),
});
```

## hono rpc の route
schema をインポートしてpathに対してschemaを紐付けます。  
今回はresponsesに200のみを規定していますが、ここに書いたレスポンスコードが実装されていないとIDE上で警告してもらえます。  

```ts:health.route.ts
import { createRoute } from "@hono/zod-openapi";
import { healthSchema } from "../schema/health.schema";

export const readHealthRoute = createRoute({
  method: "get",
  path: "/health",
  tags: ["health v1"],
  responses: {
    200: {
      description: "Client Information",
      content: {
        "application/json": {
          schema: healthSchema,
        },
      },
    }
  },
});
```

## hono rpc の api
route をインポートしてapiを実装します。  
今回はそのままjsonを返却するシンプルなものです。   

```ts:health.api.ts
import { OpenAPIHono } from "@hono/zod-openapi";
import { readHealthRoute } from "../route/health.route";

const app = new OpenAPIHono()
  .openapi(readHealthRoute, async (c) => {
    return c.json({
      status: 'ok',
      message: 'Server is running',
      timestamp: new Date().toISOString(),
    });
  });

export default app;
```

## hono rpc の index登録
最後に定義したapiを index.ts から読み込みましょう。

```ts:index.ts
import { OpenAPIHono } from "@hono/zod-openapi";
import health from './api/health.api'

export const app = new OpenAPIHono()
  .route('/api/v1', health)

~~ (snip) ~~

export default app
export type AppType = typeof app
```

## Angular側から 'hono/client' の `hc` を使ったRPC機能を使う
ではフロントエンド側から作ったサーバー型定義を読んでいきましょう。  
`hc<AppType>(サーバURL).api.v1.health.$get()` のようにサーバー側で定義したAPIのroute情報の型がフロントエンド側からも見える状態です。  

しかしいくつか気を付ける点があり、フロントエンド上で使うにあたってはAngularのsignalにマッピングする必要があり、signalの型を定義する必要がありそうです。（今回は`healthSchema`をインポートしてきました。）  

また、SSRを考えるとこのコードはサーバー側でもクライアント側でも2回呼ばれることになります。  
Angularには[TransferState](https://angular.jp/api/core/TransferState)という多重コールを避ける仕組みはありますが、これも毎回呼ぶのは記述量が増えてしまいそうですね。  

```ts
import { Component, signal, OnInit, inject } from '@angular/core';
import { z } from '@hono/zod-openapi';

import { healthSchema } from "../../server/schema/health.schema";

import { hc } from 'hono/client'
import { AppType } from '../../server/index'

@Component({
  selector: 'app-ssr',
  imports: [JsonPipe],
  template: `
    <div>Hono hc: {{ health() | json }}</div>
  `,
})

export class Ssr {
  health = signal<z.infer<typeof healthSchema>>({
    status: '', message: '', timestamp: new Date().toISOString()
  });

  async ngOnInit() {
    const data = await hc<AppType>(environment.apiUrl).api.v1.health.$get();
    if (data.ok) {
      this.health.set(await data.json());
    }
  }
}
```

# '@angular/common/http' の `HttpClient` を使う
次に '@angular/common/http' の `HttpClient` を使う方法です。  
こちらは従来のAngularの手法ですが、HttpClientには[TransferState](https://angular.jp/api/core/TransferState)が組み込まれているので少し記述量が減ったことがわかります。  

```ts
@Component({
  selector: 'app-ssr',
  imports: [JsonPipe],
  template: `
    <div>Angular HttpClient: {{ health() | json }}</div>
  `,
})

export class Ssr {
  #http = inject(HttpClient);
  health = signal<z.infer<typeof healthSchema>>({
    status: '', message: '', timestamp: new Date().toISOString()
  });

  async ngOnInit() {
    this.#http.get<z.infer<typeof healthSchema>>('/api/v1/health').subscribe((data) => {
      this.health.set(data!);
    });
  }
}
```

# '@angular/common/http' の `httpResource` を使う
最後に '@angular/common/http' の `httpResource` を使う方法です。  
現状[experimental](https://angular.jp/guide/http/http-resource)な扱いですが、以下のようにかなり記述量を減らすことができます。  

httpResourceは引数のsignalが更新される度にgetが走るイメージでしたが、引数なしでも動くようです。  
httpResourceはHttpClientが組み込まれておりTransferStateの実装なくSSRの多重発火は回避できます。  
また、zod schemaでparseすることができるのでデータ構造は共有することができそうです。  

```ts
import { Component } from '@angular/core';
import { httpResource } from "@angular/common/http";
import { JsonPipe } from '@angular/common';
import { healthSchema } from "../../server/schema/health.schema";

@Component({
  selector: 'app-ssr',
  imports: [JsonPipe],
  template: `
    <div>Angular httpResource: {{ healthResource.value() | json }}</div>
  `,
})
export class Ssr {
  healthResource = httpResource(
      () => '/api/v1/health',
      { parse: healthSchema.parse }
    );
}
```

# まとめ
AngularのSSR時のHTTP Clientを比較を行いました。  

現時点だとhttpResourceを使う方法がシンプルで良いと感じました。  
httpResourceはHttpClientをラップしており、Observableを意識しない実装が可能です。  
悪いものではないですが、個人的にrxjsに辛さを感じているのでhttpResourceを使ってsignalのみを意識した実装に期待しています。  


コードはこちら

https://github.com/nao50/angular20-hono-ssr