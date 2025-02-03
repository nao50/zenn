---
title: "AngularのfileReplacementsでTAURIとコードを共有したり置換したりする話"
emoji: "🅰️"
type: "tech"
topics: ["angular", "typescript", "tauri"]
published: true
---

# はじめに
最近、[Tauri](https://v2.tauri.app/)への期待を込めて利用感を調査しています。  
TauriはFrontend Frameworkに依存せずMobile/DesktopのNativeアプリ開発が可能です。  

せっかく好きなFrameworkが使えるので、**極力コードを共通化し**、**1つのレポジトリでWebもネイティブも開発**したくなる訳ですが、以下のような困難があります。  

 - WebとNativeでは利用するAPIが異なる（HTTP ClientやGeolocationなど）
 - 現状、TauriはServer Side Rendering（SSR）に非対応

https://v2.tauri.app/start/frontend/


今回はAngularの[Build environments](https://angular.dev/tools/cli/environments
)を使って何とかしよう。という話です。

https://angular.dev/tools/cli/environments


# Angular Build environments
Angular CLIでプロジェクトを作成した場合、angular.jsonに`configurations`というパラメータがあります。  

```json:angular.json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:browser",
          "configurations": {
            "production": { … },
            "development": { 
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.development.ts"
                }
              ],
            },
            "staging": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.staging.ts"
                }
              ]
            }
          }
        },
      }
    }
  }
}
```

この`configurations`内の`fileReplacements`パラメータは、よく`environment`ファイルの切り替えに使われます。  

```sh
my-app/src/environments
├── environment.development.ts
├── environment.staging.ts
└── environment.ts
```

`ng build --configuration development` のようにBuildコマンドを発行すると `environment` として読み込まれるファイルを切り替えることができます。  

```ts
import { environment } from './../environments/environment';
// Fetches from `http://my-prod-url` in production, `http://my-dev-url` in development.
fetch(environment.apiUrl);
```

`fileReplacements` を使い **Web/NativeのAPI差分** と **SSRとCSR/SSGの切り替え** を実現します。  

## Web/NativeのAPI差分の吸収
Web/NativeのAPI差分 という課題を考えます。  
AngularにおけるService ClassをWeb用に `product.service.ts`、Native用に `product.service.tauri.ts` というファイルを作ります。  

今回は `HTTP Client` と `Geolocation` を使ってみます。  
関数の返り値を揃えることでComponent側での動作を極力揃えることができそうです。  

```ts:product.service.ts
import { Injectable, resource, ResourceRef, signal } from '@angular/core';

type Product = {
  title: string;
};

@Injectable({
  providedIn: 'root'
})
export class ProductService {
  id = signal(1);

  // web用のHTTP Client
  productResource: ResourceRef<Product> = resource({
    request: () => this.id(), 
    loader: async ({ request: id, abortSignal }) => {
      const resp = await fetch(`https://dummyjson.com/products/${id}`, {
        signal: abortSignal,
      });
      return resp.json() as Promise<Product>;
    },
  });

  // web用のGeolocation
  getCurrentPosition(): Promise<GeolocationPosition | undefined>{
    const positionOptions: PositionOptions = {
      enableHighAccuracy: true,
      maximumAge: 0, // Not use a cached position
      timeout: 100000 // ms
    };

    return new Promise(
      (
        resolve: (pos: GeolocationPosition) => void,
        reject: (err: GeolocationPositionError) => void
      ) => {
        if (navigator.geolocation) {
          const watchId = navigator.geolocation.getCurrentPosition(resolve, reject, positionOptions);
        }
      }
    );
  }
}
```

Native側にはTAURIからpluginとして[HTTP Client](https://v2.tauri.app/plugin/http-client/)と[Geolation](https://github.com/tauri-apps/plugins-workspace/tree/v2/plugins/geolocation)が提供されています。

https://v2.tauri.app/plugin/http-client/

https://github.com/tauri-apps/plugins-workspace/tree/v2/plugins/geolocation

```ts:product.service.tauri.ts
import { Injectable, resource, ResourceRef, signal } from '@angular/core';
import { fetch } from '@tauri-apps/plugin-http';
import {
  checkPermissions,
  requestPermissions,
  getCurrentPosition,
  watchPosition
} from '@tauri-apps/plugin-geolocation'

type Product = {
  title: string;
};

@Injectable({
  providedIn: 'root'
})
export class ProductService {
  id = signal(1);

  productResource: ResourceRef<Product> = resource({
    request: () => this.id(), 
    loader: async ({ request: id, abortSignal }) => {
      const resp = await fetch(`https://dummyjson.com/products/${id}`, {
        signal: abortSignal,
      });
      return resp.json() as Promise<Product>;
    },
  });

  async getCurrentPosition(): Promise<GeolocationPosition | undefined>{
    let permissions = await checkPermissions()
    if (permissions.location === 'prompt' || permissions.location === 'prompt-with-rationale') {
      permissions = await requestPermissions(['location'])
    }

    if (permissions.location === 'granted') {
      return getCurrentPosition()
    }
    return undefined
  }
}
```

これをangular.jsonで以下のようにfileReplacementsすることができます。

```json:angular.json
"tauri": {
  "fileReplacements": [
    {
      "replace": "src/app/app.routes.server.ts",
      "with": "src/app/app.routes.server.tauri.ts"
    },
  ]
}
```

## SSRとCSR/SSGの切り替え
先述の通り、TAURIは現状Server Side Rendering（SSR）に対応していません。  
よってWeb側でSSRしたい時、Native側ではCSRないしSSGに切り替える必要があります。  

Angular 19から[Server Routing](https://angular.dev/guide/hybrid-rendering)という概念が追加されました。  
この機能を使うことでSSRとCSR/SSGを切り替えることができそうです。  

通常の`app.routes.server.ts`では以下のようにSSRを指定し

```ts:app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  {
    path: 'top',
    renderMode: RenderMode.Server,
  },
  {
    path: '**',
    renderMode: RenderMode.Server
  }
];
```

`app.routes.server.tauri.ts`というファイルを作って同じパスに対してCSR/SSGを指定します。  

```ts:app.routes.server.tauri.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  {
    path: 'top',
    renderMode: RenderMode.Client,
  },
  {
    path: '**',
    renderMode: RenderMode.Prerender
  }
];
```

あとはangular.jsonでfileReplacementsするだけです。  

```json:angular.json
"tauri": {
  "fileReplacements": [
    {
      "replace": "src/app/app.routes.server.ts",
      "with": "src/app/app.routes.server.tauri.ts"
    }
  ]
}
```

## Build environments を ng serve から使う方法
余談ですが `ng serve` コマンドに対しても引数 `--configuration` の引数を取るためにはangular.jsonの`serve`に追加したconfigurationsを設定する必要がありました。  

```json:angular.json
"serve": {
  "builder": "@angular-devkit/build-angular:dev-server",
  "configurations": {
    "production.web": {
      "buildTarget": "angular19-tauri:build:production.web"
    },
    "production.tauri": {
      "buildTarget": "angular19-tauri:build:production.tauri"
    },
    "development.web": {
      "buildTarget": "angular19-tauri:build:development.web"
    },
    "development.tauri": {
      "buildTarget": "angular19-tauri:build:development.tauri"
    }
  },
  "defaultConfiguration": "development"
},
```

# まとめ
AngularのfileReplacementsでTAURIとコードを共有したり置換したりする方法をまとめました。  
コードは[こちら](https://github.com/nao50/angular19-tauri)  

https://github.com/nao50/angular19-tauri
