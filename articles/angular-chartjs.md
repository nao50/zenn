---
title: "Angularã§Chart.js v3ãä½¿ã"
emoji: "ð°ï¸"
type: "tech"
topics: ["angular"]
published: true
---

![angular Chart.js](/images/angular-chartjs-01.png)

# ã¯ããã«
ä»åã¯[Angular](https://angular.io/)ã§[Chart.js](https://www.chartjs.org/)ãä½¿ãæ¹æ³ãã¾ã¨ãã¾ãã  
ã¾ããä»åã®ãµã³ãã«ã¯[stackblitz](https://stackblitz.com/edit/angular-ivy-7zu6jy?file=src/app/app.component.ts)ã§å¬éãã¦ãã¾ãã

https://stackblitz.com/edit/angular-ivy-7zu6jy?file=src/app/app.component.ts

# Chart.js ã¤ã³ã¹ãã¼ã«
chart.jsãnpmã§ã¤ã³ã¹ãã¼ã«ãã¾ãã
`3.7.1`ãå¥ãã¾ããããã¤ã®éã«ãåå®ç¾©ãã¡ã¤ã«ãå¬å¼ããæä¾ããã¦ã¾ãã­ã  

```bash
$ npm install chart.js
```

# Chart.jsãä½¿ã
AngularããChart.jsãå©ç¨ãã¦ããã¾ãã
Angularããcanvasã¸æç»ãè¡ãå ´åãcanvasã¿ã°ã«`#canvas01`ã¨ããã·ã³ãã«ãä¸ããViewChildããåç§ã§ãã¾ãããã

```html:src/app/app.component.html
<div>
    <canvas #canvas01></canvas>
</div>
```

Componentå´ãã`@ViewChild()`ã§`#canvas01`ã®ElementRefãåå¾ãã°ã©ããæãã¾ãã

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

# ã¾ã¨ã
AngularããChart.jsãä½¿ãæ¹æ³ãã¾ã¨ãã¾ããã
ä»åã¯Line Chartã®ã³ã¼ããæ²è¼ãã¾ãããããµã³ãã«ã³ã¼ãã«ã¯ãBar ChartããPie ChartããSanky Chartããæ²è¼ãã¦ã¾ãã
ãã²çºãã¦ã¿ã¦ãã ããã  

https://stackblitz.com/edit/angular-ivy-7zu6jy?file=src/app/app.component.ts