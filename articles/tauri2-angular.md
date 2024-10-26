---
title: "TAURI でも最新の Angular が使いたい"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular", "typescript", "javascript", "tauri"]
published: true
---

# はじめに
[TAURI v2.0](https://v2.tauri.app/)が安定版としてリリースされました。  
TAURIで最新版のAngularを使う初期設定をまとめます。  

https://v2.tauri.app/

## TAURI の初期設定
TAURIの[初期設定](https://v2.tauri.app/start/create-project/)には2種類あリます。  

https://v2.tauri.app/start/create-project/

### create-tauri-app コマンドを使う方法
`create-tauri-app`コマンドを使うとターミナル上から開発環境やフレームワークを選択できます。  


```sh
$ npm create tauri-app@latest
✔ Project name · tauri-app
✔ Identifier · com.tauri-app.app
✔ Choose which language to use for your frontend · TypeScript / JavaScript - (pnpm, yarn, npm, deno, bun)
✔ Choose your package manager · npm
✔ Choose your UI template · Angular - (https://angular.dev/)

$ cd tauri-app
$ npm install
$ npm run tauri dev
```

24年10月時点の[Angularの最新](https://github.com/angular/angular/releases)はv18.2.9でしたが、`create-tauri-app`で入るAngularは `v17.0.0` でした。  

最新のAngularに追従したい場合はもうひとつのマニュアルセットアップが必要そうです。  

### マニュアルセットアップ (Tauri CLI)
Angularでマニュアルセットアップを行う場合、まずは最新の[Angular CLI](https://github.com/angular/angular-cli)からプロジェクトを作成し、そのプロジェクト内でTAURIの設定を入れていきます。  

`npx tauri init`でAngularの作法に沿って選択していきます。  

```sh
$ ng new tauri-app
$ cd tauri-app
$ npm install -D @tauri-apps/cli@latest
$ npx tauri init
✔ What is your app name? · tauri-app
✔ What should the window title be? · tauri-app
? Where are your web assets (HTML/CSS/JS) located, relative to the "<current dir>/src-tauri/tauri.conf.json" fil✔ Where are your web assets (HTML/CSS/JS) located, relative to the "<current dir>/src-tauri/tauri.conf.json" file that will be created? · ../dist/tauri-app//browser
✔ What is the url of your dev server? · http://localhost:4200
✔ What is your frontend dev command? · npm run start
✔ What is your frontend build command? · npm run build
```

これで初期設定は完了です。  
以下のコマンドでGUIが立ち上がるのを確認します。  

```sh
$ npx tauri dev
```
![tauri2 with newest angular](/images/tauri2-angular-01.png)


# まとめ
TAURIで最新版のAngularを使う初期設定をまとめました。  
Githubでコードを公開しておきます。

https://github.com/nao50/zenn-tauri-angular
