---
title: "AngularのHttpClientのエラーハンドリングを理解する"
emoji: "🅰️"
type: "tech"
topics: ["angular"]
published: false
---

# はじめに
今回は[Angular](https://angular.io/)におけるHTTP Clientのエラーハンドリングについてまとめます。  
Angularではフレームワークとして`HttpClient`クラスが提供されているので利用します。

また、今回のサンプルは[stackblitz](https://stackblitz.com/)で公開しています。

https://stackblitz.com/edit/angular-ivy-w4dtzi?file=src/app/data.service.ts

## Angularについて
AngularではHTTP Client、ルーティング、フォーム、PWAなどが1st partyとして提供されます。
これらが[Angular CLI](https://angular.io/cli)の強力な`ng update`コマンドでバージョンアップへ追従できます。

また、[日本のミュニティ](https://community.angular.jp/)もあり初心者や中級者が次のステップに踏み出せる機会が多い印象です。
ぜひ「何から始めたらいいかわからない」というあなた！
Angularでフロントエンド開発を一通り体験してみてはいかがでしょうか。

https://community.angular.jp/

# Angular HttpClientの復習
[日本語ドキュメント](https://angular.jp/guide/http) で HttpClientを復習します。  

## データ型の定義
まずデータ型をInterfaceとして定義しておきます。  

```typescript:src/app/data.ts
export interface Data {
    key1: string;
    key2: string;
}
```

## サービス
このInterfaceをServiceでhttp.getの型パラメーターとして指定します。  
observeオプションとして `{ observe: 'response' }` を指定することでheaders含むResponse全体を読むことができます。  

```typescript:src/app/data.service.ts
dataUrl = 'assets/data.json';

getDataResponse(): Observable<HttpResponse<Data>> {
  return this.http.get<Data>(this.dataUrl, { observe: 'response' });
}
```

## コンポーネント
このサービスをComponent側でsubscribeします。  
responseにbodyとheadersが入っているのでbodyをデータに格納しています。  

```typescript:src/app/app.component.ts
data: Data;

getData() {
  this.dataService.getDataResponse().subscribe({
    next: (response: HttpResponse<Data>) => {
      console.log('response: ', response);
      this.data = response.body;
    },
    error: (e) => {
      switch (e.status) {
        default:
          console.log('error: ', e);
          break;
      }
    },
    complete: () => console.info('complete'),
  });
}
```

実際のレスポンスのデータ構造は以下の通りです。

![angular http response](/images/angular-httpclient-error-handling-01.png)

## テンプレート
html側では以下のようにデータをbindすることができます。

```html:typescript:src/app/app.component.html
key1: {{ data?.key1 }}
key2: {{ data?.key2 }}
```

# Angular HttpClientのエラーハンドリング
私は上記コードの通りComponent側でsubscribeできるerrorでエラーメッセージを出したりログアウト処理を行なったりしていました。
ですが最近、[公式ドキュメント](https://angular.jp/guide/http#%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%AE%E8%A9%B3%E7%B4%B0%E3%82%92%E5%8F%96%E5%BE%97%E3%81%99%E3%82%8B)を眺めていたところ以下の文章を発見。

> アプリケーションは、データアクセスが失敗したときに、ユーザーに役立つフィードバックを提供する必要があります。 生のエラーオブジェクトは、フィードバックとして特に役立ちません。 エラーが発生したことを検出することに加えて、エラーの詳細を取得し、それらの詳細を使用してユーザーフレンドリーなレスポンスを作成する必要があります。

なるほどたしかに。
でもResponse statusに応じてメッセージの出し分けならComponent側でもできます。

> 2種類のエラーが発生する可能性があります。
> - サーバーバックエンドがリクエストを拒否し、404や500などのステータスコードでHTTPレスポンスを返す場合があります。これらはエラーレスポンスです。
> - 要求を正常に完了できないネットワークエラーや、RxJSオペレーター内でスローされた例外など、クライアント側で問題が発生する可能性があります。これらのエラーは0に設定されたstatusと、ProgressEventオブジェクトが含まれるerrorプロパティを持っています。ProgressEventオブジェクトのtypeは詳細な情報を提供する可能性があります。

なるほどたしかに。
クライアント側でのエラーはResponseを待ってのハンドリングだけでは必要十分とは言えない気がします。

> HttpClientは、HttpErrorResponseで両方の種類のエラーをキャプチャします。そのレスポンスを調べて、エラーの原因を特定できます。

なるほどたしかに。
というわけでService側のでエラーハンドリングを調査してみます。

## Serviceでのエラーハンドリング
[公式ドキュメント](https://angular.jp/guide/http#%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%AE%E8%A9%B3%E7%B4%B0%E3%82%92%E5%8F%96%E5%BE%97%E3%81%99%E3%82%8B)ではServiceとしてhandleErrorを定義することを提案しているようです。

```typescript:src/app/data.service.ts
getDataResponse(): Observable<HttpResponse<Data>> {
  return this.http.get<Data>(this.dataUrl, { observe: 'response' })
  .pipe(
    catchError(this.handleError),
  );
}

private handleError(error: HttpErrorResponse) {
  // クライアント側あるいはネットワークによるエラー
  if (error.status === 0) {
    console.error('An error occurred:', error.error.message);
  // サーバー側から返却されるエラー
  } else {
    console.error(`Backend returned code ${error.status}, body was: `, error.error.message);
  }
  // エラーメッセージの返却
  return throwError(() => new Error('Something bad happened; please try again later.')
}
```

試しに誤ったURLを入力してみるとサーバーサイドのエラーとしてstatusとerror内容を取得し、Component側へも任意のエラーメッセージを返却できていることがわかります。  

![angular error response](/images/angular-httpclient-error-handling-02.png)

また、最低限抑えておきたいTimeout処理とRetry処理もrxjsのoperatorを活用することで簡単に実現できそうです。

```typescript:src/app/data.service.ts
getDataResponse(): Observable<HttpResponse<Data>> {
  return this.http.get<Data>(this.dataUrl, { observe: 'response' })
  .pipe(
    timeout(2500), // タイムアウト処理
    retry(3), // リトライ処理
    catchError(this.handleError),
  );
}
```

# まとめ
AngularにおけるHTTP Clientのエラーハンドリングについてまとめました。
Angularには[Interceptor](https://angular.jp/guide/http#%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%82%BB%E3%83%97%E3%82%BF%E3%83%BC%E3%82%92%E6%9B%B8%E3%81%8F)というロギング/認証処理などの共通処理をまとめる機能があり、コード量が減って良いので次回以降にまとめていきます。

## 参考URL
https://angular.jp/guide/http#http%E3%81%AB%E3%82%88%E3%82%8B%E3%83%90%E3%83%83%E3%82%AF%E3%82%A8%E3%83%B3%E3%83%89%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%81%A8%E3%81%AE%E9%80%9A%E4%BF%A1

https://stackblitz.com/edit/angular-ivy-w4dtzi?file=src/app/data.service.ts
