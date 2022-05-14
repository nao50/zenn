---
title: "AngularでChart.js v3を使う"
emoji: "🅰️"
type: "tech"
topics: ["angular"]
published: true
---

![angular Chart.js](/images/angular-chartjs-01.png)

# はじめに
今回は[Angular](https://angular.io/)で[Chart.js](https://www.chartjs.org/)を使う方法をまとめます。  
また、今回のサンプルは[stackblitz](https://stackblitz.com/edit/angular-ivy-7zu6jy?file=src/app/app.component.ts)で公開しています。

https://stackblitz.com/edit/angular-ivy-7zu6jy?file=src/app/app.component.ts

# Chart.js インストール
chart.jsをnpmでインストールします。
`3.7.1`が入りました。いつの間にか型定義ファイルも公式から提供されてますね。  

```bash
$ npm install chart.js
```

# Chart.jsを使う
AngularからChart.jsを利用していきます。
Angularからcanvasへ描画を行う場合、canvasタグに`#canvas01`というシンボルを与え、ViewChildから参照できます。　　

```html:src/app/app.component.html
<div>
    <canvas #canvas01></canvas>
</div>
```

Component側から`@ViewChild()`で`#canvas01`のElementRefを取得しグラフを描きます。

```typescript:src/app/app.component.ts
import { Component, AfterViewInit, ViewChild, ElementRef } from '@angular/core';
import { Chart } from 'chart.js';

@Component({
  selector: 'my-app',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css'],
})
export class AppComponent implements AfterViewInit {
  context01: CanvasRenderingContext2D;
  @ViewChild('canvas01') canvas01: ElementRef;

  ngAfterViewInit() {
    this.context01 = this.canvas01.nativeElement.getContext('2d');
    let canvas01 = new Chart(this.context01, {
      type: 'line',
      data: {
        datasets: [
          {
            label: 'Data01',
            backgroundColor: 'rgba(255, 255, 153, 0.5)',
            borderColor: 'rgba(255, 99, 132, 0)',
            pointRadius: 0,
            fill: true,
            data: [
              { x: 0, y: 2 },
              { x: 1, y: 1 },
              { x: 2, y: 2.5 },
              { x: 3, y: 5 },
              { x: 4, y: 3 },
              { x: 5, y: 4 },
              { x: 6, y: 9 },
              { x: 7, y: 7 },
              { x: 8, y: 12 },
            ],
          },
          {
            label: 'Data02',
            backgroundColor: 'rgba(153, 255, 255, 0.5)',
            borderColor: 'rgba(255, 99, 132, 0)',
            pointRadius: 0,
            fill: true,
            data: [
              { x: 0, y: 1 },
              { x: 1, y: 4 },
              { x: 2, y: 8 },
              { x: 3, y: 12 },
              { x: 4, y: 1 },
              { x: 5, y: 5 },
              { x: 6, y: 2 },
              { x: 7, y: 3 },
              { x: 8, y: 1 },
            ],
          },
        ],
      },
      options: {
        responsive: true,
        plugins: {
          title: {
            display: true,
            text: 'Angular & Chart.js',
          },
          tooltip: {
            mode: 'index',
            intersect: false,
          },
        },
        scales: {
          x: {
            display: true,
            type: 'linear',
            position: 'bottom',
            title: {
              display: true,
              text: 'X',
            },
          },
          y: {
            type: 'linear',
            title: {
              display: true,
              text: 'Y',
            },
          },
        },
      },
    });
  }
}
```

# まとめ
AngularからChart.jsを使う方法をまとめました。
今回はLine Chartのコードを掲載しましたが、サンプルコードには「Bar Chart」「Pie Chart」「Sanky Chart」を掲載してます。
ぜひ眺めてみてください。  

https://stackblitz.com/edit/angular-ivy-7zu6jy?file=src/app/app.component.ts