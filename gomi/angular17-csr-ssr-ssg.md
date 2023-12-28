---
title: "Angular17でCSR/SSR/SSGを組み合わせる"
emoji: "🅰️"
type: "tech"
topics: ["angular", "typescript", "signals"]
published: false
---

# はじめに
Angular17で **Client Side Rendering** (CSR) / **Server Side Rendering** (SSR) / **Static Site Generation** (SSG) を1つのWebアプリに共存させる時の方法をまとめます。

また、実用を見据え「**interceptorを用いたキャッシュ**」「**signalsを用いた状態管理**」「**deferを用いた遅延読み込み**」を行います。  

# Angularプロジェクトの作成
まずはAngularのCLIからアプリの雛形を生成します。  
今回は `angular17-ssr` というプロジェクト名とします。  

生成時点で **SSRする？** と問われるので **YES** としておきましょう。  

```sh
$ npm install -g @angular/cli # CLIのインストール (Angular CLI: 17.0.8)
$ ng new angular17-ssr
? Which stylesheet format would you like to use? SCSS
? Do you want to enable Server-Side Rendering (SSR) and Static Site Generation (SSG/Prerendering)? Yes
$ cd angular17-ssr && ng serve
```

`http://localhost:4200` にアクセスすると初期画面が表示されます。  

![angular 17 start image](/images/angular17-start.png)

# AngularでClient Side Rendering (CSR)する
Angular CLIからClient Side Rendering (CSR)するページを作成します。  
今回は `csr` という名前のコンポーネント名とします。  

```sh
$ ng generate component csr
CREATE src/app/csr/csr.component.scss (0 bytes)
CREATE src/app/csr/csr.component.html (18 bytes)
CREATE src/app/csr/csr.component.spec.ts (575 bytes)
CREATE src/app/csr/csr.component.ts (223 bytes)
```

生成したコンポーネントを表示するための設定を行います。  
まずは初期画面が不要なので `app.component.html` の `router-outlet` 以外を削除します。  

```html: app.component.html
<router-outlet></router-outlet>
```

`router-outlet` 内に `csr` コンポーネントを表示させるため `app.routes.ts`を設定します。  
今回はトップ画面として `csr` コンポーネントを表示するためリダイレクトの設定を行います。  

```ts: app.routes.ts
import { Routes } from '@angular/router';
import { CsrComponent } from './csr/csr.component';

export const routes: Routes = [
  {
    path: 'csr',
    component: CsrComponent,
  },
  {
    path: '**',
    redirectTo: 'csr',
  },
];
```

`http://localhost:4200/csr` にアクセスすると `csr works!` と表示されます。  

## 外部APIからデータ取得し表示する
今回は外部APIとして `jsonplaceholder` 様を利用させていただきます。  

https://jsonplaceholder.typicode.com

今回は `/todos` APIを使うためAngular CLIからインターフェースを作成します。  
今回は `todo` という名前のインターフェースとします。  

```sh
$ ng generate interface interface/todo
CREATE src/app/interface/todo.ts (26 bytes)
```

以下のようなデータフォーマットを定義します。  

```ts: interface/todo.ts
export interface Todo {
  userId: number;
  id: number;
  title: string;
  completed: boolean;
}
```

次にAngular CLIからデータ取得を行うサービスクラスを作成します。  
今回は `todo` という名前のサービス名とします。  

```sh
$ ng generate service service/todo
CREATE src/app/service/todo.service.spec.ts (347 bytes)
CREATE src/app/service/todo.service.ts (133 bytes)
```

以下のようなサービスクラスとします。

```ts: service/todo.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Todo } from '../interface/todo';

@Injectable({
  providedIn: 'root'
})
export class TodoService {
  #http = inject(HttpClient);

  findOneTodo(id: number): Observable<Todo> {
    return this.#http.get<Todo>('https://jsonplaceholder.typicode.com/todos/' + `${id}`)
  }
}
```

このままでも動きますが、[公式ドキュメント](https://angular.io/api/common/http/provideHttpClient#description)によれば標準の[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)を使うことが強く推奨されています。  
`app.config.ts` にFetch APIを使うための設定`withFetch()`を追加します。  

```ts: app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';

import { routes } from './app.routes';
import { provideClientHydration } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideClientHydration(),
    provideHttpClient(
      withFetch(),
    ),
  ]
};
```

ようやくClient Side Rendering (CSR)する準備が整いました。  
`csr.component.ts`から`todo.service.ts`を利用し外部APIからデータ取得を行います。  

```ts: csr/csr.component.ts
import { Component, inject } from '@angular/core';
import { TodoService } from '../service/todo.service';
import { Todo } from '../interface/todo';

@Component({
  selector: 'app-csr',
  standalone: true,
  imports: [],
  providers: [TodoService],
  templateUrl: './csr.component.html',
  styleUrl: './csr.component.scss'
})
export class CsrComponent {
  todoService = inject(TodoService);
  todo!: Todo;

  ngOnInit(): void {
    const id = 1;
    this.todoService.findOneTodo(id).subscribe((res) => {
      this.todo = res;
    })
  }
}
```

`todo`というプロパティにデータが格納されているので`csr.component.html`で表示します。

```html: csr/csr.component.html
<p>csr works!</p>

<div><b>userId:</b> {{ todo.userId }}</div>
<div><b>id:</b> {{ todo.id }}</div>
<div><b>title:</b> {{ todo.title }}</div>
<div><b>completed:</b> {{ todo.completed }}</div>
```

外部APIから取得したデータが表示できました。  

![angular 17 csr image](/images/angular17-csr.png)

## interceptor を用いたキャッシュを設定する
Angular CLIから通信の制御を行う関数を作成します。  
今回は `cache` という名前のインターセプタとします。  

```sh
$ ng generate interceptor interceptor/cache
CREATE src/app/interceptor/cache.interceptor.spec.ts (480 bytes)
CREATE src/app/interceptor/cache.interceptor.ts (150 bytes)
```

キャッシュするデータフォーマットを規定するために `cache`という名前のインターフェースを作成します。  

```sh
$ ng generate interface interface/todo
CREATE src/app/interface/todo.ts (26 bytes)
```

以下のようなデータフォーマットを定義します。  

```ts: interface/todo.ts
export interface Todo {
  userId: number;
  id: number;
  title: string;
  completed: boolean;
}
```
