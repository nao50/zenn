---
title: "Angular17でCSR/SSR/SSGを組み合わせる"
emoji: "🅰️"
type: "tech"
topics: ["angular", "typescript", "signals"]
published: true
---

# はじめに
Angular17で **Client Side Rendering (CSR)** / **Server Side Rendering (SSR)** / **Static Site Generation (SSG)** を併用する方法をまとめます。

# Angular で CSR / SSR / SSG の併用
まずはAngularのプロジェクトに[SSRを導入](https://angular.dev/guide/ssr)します。  

https://angular.dev/guide/ssr

```sh
$ ng new --ssr # 新プロジェクトを生成
$ ng add @angular/ssr # 既存プロジェクトにSSRを導入
```

この時点で以下のようなExpressのサーバが生成されます。  
以下の通り全リクエストで dist 配下の `server/index.server.html` を返しています。  

```ts: server.ts
// 〜（略）〜
export function app(): express.Express {
  const server = express();
  // serverDistFolder → /{PJ-PATH}/dist/{PJ-NAME}/server
  const serverDistFolder = dirname(fileURLToPath(import.meta.url));
  // browserDistFolder → /{PJ-PATH}/dist/{PJ-NAME}/browser
  const browserDistFolder = resolve(serverDistFolder, '../browser');
  // indexHtml → /{PJ-PATH}/dist/{PJ-NAME}/server/index.server.html
  const indexHtml = join(serverDistFolder, 'index.server.html');

  // 〜（略）〜

  // 全リクエストで indexHtml すなわち dist 配下の server/index.server.html を返却
  server.get('*', (req, res, next) => {
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
// 〜（略）〜
```
## SSG の利用
SSG は `angular.json` に prerendering したいパスを指定するだけです。  

https://angular.dev/guide/prerendering

以下のようなテキストファイルを用意し

```txt: prerender-routes.txt
/ssg
```

`angular.json` の `prerender` にファイル名を記載します。  

```json
 "prerender": {
    "discoverRoutes": false,
    "routesFile": "prerender-routes.txt"
  },
```

`npm run build` すると `browser/ssg/index.html` というファイルができます。  
以下のように専用のパスを作ってもアクセスできますが、従来のSSRのパスの `commonEngine.render()` 経由でも問題なく SSG されます。  

```ts: server.ts
// 〜（略）〜
  server.get('/ssg', (req, res, next) => {
    res.sendFile(join(browserDistFolder, 'ssg/index.html'));
  });
// 〜（略）〜
```

> commonEngine チョットヨクワカラナイ...  

## CSR / SSR の併用
CSR / SSR の併用は **CSR したいパスへのリクエストに `browser/index.html` を返却** することで **SSR と CSR が共存** できそうです。
以下のように `server.ts` に `browser/index.html` を返却するパスを追加します。  

```ts: server.ts
// 〜（略）〜
  server.get('/csr', (req, res, next) => {
    res.sendFile(join(browserDistFolder, 'index.html'));
  });

  server.get('*', (req, res, next) => {
    // 〜（略）〜
  });
// 〜（略）〜
```

`http://{ SERVER }/csr` にアクセスすると `browser/index.html` を取得し、ngOnInit などに書かれた fetch などの処理が走ることが確認できます。  
`http://{ SERVER }/ssr` にアクセスすると、レンダリング済の `server/index.server.html` が表示されます。  

## CSR / SSR ページ間の遷移を確認
CSR と SSR 間のページ遷移を見てみましょう。  
routerLink での遷移は `<router-outlet>` 内に表示するコンポーネントを差し替えます。  

CSR 用の `browser/index.html` から SSR のパスへ routerLink で遷移すると`browser/index.html` のまま SSR 画面の表示を行うため、実際にはClient Side Renderingとなります。  

| | 初期表示 | 初期表示 | 遷移方法 | 遷移後 | 遷移後ページ |
| ---- | ---- | ---- | ---- | ---- | ---- |
| ① | CSR | browser/index.html | routerLink | SSR | browser/index.html |
| ② | CSR | browser/index.html | href | SSR | server/index.server.html |
| ③ | SSR | server/index.server.html | routerLink | CSR | server/index.server.html |
| ④ | SSR | server/index.server.html | href | CSR | browser/index.html |

:::message
[loadComponent](https://zenn.dev/lacolaco/books/angular-standalone-components/viewer/routing#%E3%82%B9%E3%82%BF%E3%83%B3%E3%83%89%E3%82%A2%E3%83%AD%E3%83%B3%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%9F%E3%83%AB%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)で遅延読み込みを行いレンダリング時間を抑えることができます。
**SSR を利用する目的を明確にして** 設計する必要があります。  
:::

## リクエストのキャッシュについて
SSG ページであっても ngOnInit に Fetch 処理が書かれていると、ブラウザに表示されてから再び Fetch 処理が走ってしまいます。  
`TransferState` と `isPlatformServer` を利用することでサーバサイドのみで Fetch し、そのデータを保持することができます。  

```ts: ssg.component.ts
// 〜（略）〜
export class SsgComponent {
  platformId = inject(PLATFORM_ID);
  transferState = inject(TransferState);
  stateKey = makeStateKey<Todo>('todo-state-key');

  td = signal<Todo>({});

  ngOnInit(): void {
    if(isPlatformServer(this.platformId)){
        this.todoService.findOneTodo().subscribe((res) => {
          this.td.set(res);
          this.transferState.set<Todo>(this.stateKey, res);
        }
      )
    }
    this.td.set(this.transferState.get<Todo>(this.stateKey, {}));
  }
}
```

また、Requestキャッシュについては別の記事を書きましたのでそちらをどうぞ。  

https://zenn.dev/nao50/articles/angular17-interceptor

# まとめ
Angular17で CSR / SSR / SSG の併用についてまとめました。
近年のAngularは [standalone components](https://angular.io/guide/standalone-components)や[signals](https://angular.dev/guide/signals)など開発体験の向上が凄まじいですね。  

[Roadmap](https://angular.dev/roadmap#improve-runtime-performance-and-make-zonejs-optional)のzone.jsからの脱却など更なるパフォーマンス向上が期待できそうです。
個人的にRxjsがなくなると、より初心者に優しくなると感じているので期待をしています。  

https://angular.dev/roadmap