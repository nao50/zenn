---
title: "Angular Flex-Layout 入門"
emoji: "🅰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular"]
published: false
---

![angular FlexLayout](/images/angular-flexlayout-01.png)

# はじめに
Angularでは[flex-layout](https://github.com/angular/flex-layout)というFlexbox と Responsive API を使って Layout してくてるディレクティブが提供されています。  
使いこなすことでCSSの量が減らすことが可能なので入門してみます。  


また、本家に素晴らしい[demoページ](https://tburleson-layouts-demos.firebaseapp.com) があるのでそちらも参照ください。  
例のごとく[stackblitz](https://stackblitz.com/edit/angular-flexlayouts-sample?file=src/app/app.component.ts)上にデモを作成しています。  

 - 列と行が入れ子になった flex-layout の作り方
 - スマートデバイス、PC等の画面サイズが異なる場合のレイアウトの作り方
 - グリッドリストと flex-layout の組み合わせ方

https://stackblitz.com/edit/angular-flexlayouts-sample?file=src/app/app.component.ts

# Angular flex-layout
例のごとく stackblitz 上に Demo を作成しています。



https://stackblitz.com/edit/angular-flexlayouts-sample