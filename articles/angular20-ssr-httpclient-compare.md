---
title: "Angular20 の SSR 時の HttpClient を考える"
emoji: "🅰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular", "hono", "typescript", "javascript"]
published: false
---

# はじめに
前回、[Hono](https://hono.dev/)と[Angular20](https://angular.dev/)でSSRする方法をまとめました。  

https://zenn.dev/nao50/articles/angular20-hono-ssr

今回はAngularのSSR時のHTTP Clientを比較していこうと思います。  

 - 'hono/client' の `hc` を使った[RPC](https://hono.dev/docs/guides/rpc)機能を使う
 - '@angular/common/http' の `HttpClient` を使う
 - '@angular/common/http' の `httpResource` を使う

# 'hono/client' の `hc` を使ったRPC機能を使う

# '@angular/common/http' の `HttpClient` を使う

# '@angular/common/http' の `httpResource` を使う


# まとめ
[Hono](https://hono.dev/)と[Angular20](https://angular.dev/)でSSRする方法をまとめました。  

コードはこちら

https://github.com/nao50/angular20-hono-ssr