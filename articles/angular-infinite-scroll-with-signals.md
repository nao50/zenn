---
title: "Angular19で無限スクロールを作ってみる"
emoji: "🅰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["angular", "typescript"]
published: true
---

今年の[Angular](https://angular.dev/)は盛りだくさんでしたね！
これは[Angularアドベントカレンダー](https://qiita.com/advent-calendar/2024/angular) 17日目の記事です！

https://qiita.com/advent-calendar/2024/angular

# はじめに
今回は[Angular signals](https://angular.jp/guide/signals)を使った無限スクロールを実装します。  
実装コードは末尾にレポジトリを置いておきます。  

![angular http response](/images/angular-infinite-scroll-with-signals-02.png)

サーバーサイドは以下のポケモンAPIを使わせていただきます。  

https://pokeapi.co/

## 無限スクロールの追加読み込みの実装
ポケモンAPIからデータ取得を行います。
画面遷移後も取得したデータを保持したいのでservice側のsignalsで状態管理します。

```ts:api.service.ts
export class ApiService {
  #http = inject(HttpClient);
  items = signal<ItemDetail[]>([]);
  count = signal(0);
  lastIndex = signal(0);
  next = signal('');

  getItems(next?: string) {
    const limit = 15;
    const offset = 0;

    const url = this.next() ? this.next() : `https://pokeapi.co/api/v2/pokemon/?limit=${limit}&offset=${offset}`

    this.#http.get<ItemDetail>(url).subscribe((res) => {
      this.count.update(() => res.count);
      this.lastIndex.update(lastIndex => lastIndex + res.details.length);
      this.next.update(() => res.next ? res.next : '');
      this.items.update(i => {
        return [...i, ...res.details];
      });
    });
  }
}
```

取得したデータをTimeline Componentで表示します。  
また子コンポーネントの`app-item`から`(loaded)`されたitemのidを受け取るようにします。  
読み込んだデータの末尾から5個前のitemがviewportに入ったら追加の読み込みが走ります。  

```ts:timeline.component.ts
@Component({
  selector: 'app-timeline',
  imports: [RouterLink, ItemComponent],
  template: `
  @for (item of items(); track item.id) { @defer (on viewport) {
    <app-item [item]="item" (loaded)="loaded(item.id)" [id]="item.id" />
  } @placeholder (minimum 500ms) {
    // ~ loading animation ~
  } }
  `,
})

export class TimelineComponent implements OnInit {
  apiService = inject(ApiService);
  items = computed(() => this.apiService.items());
  count = computed(() => this.apiService.count());
  lastIndex = computed(() => this.apiService.lastIndex());

  ngOnInit() {
    // ① データ読み込み
    this.apiService.getItems();
  }

  // ② データの追加読み込み
  loaded(id: number) {
    if ( this.lastIndex() < this.count() && this.items().at(-5)?.id === id) {
      this.getItems();
    }
  }
}
```

Item Componentでは以下のように`effect()`内で`this.item()`を呼ぶことで`item()`が更新される度に（スクロールされitemがviewportに入る度に）親コンポーネントの`loaded()`が呼ばれます。  

```ts:item.component.ts
export class ItemComponent {
  readonly item = input.required<ItemDetail>();
  loaded = output<number>();

  constructor() {
    effect(() => {
      this.loaded.emit(this.item().id);
    });
  }
}
```

## 画面遷移後もスクロール位置を保持する実装
無限スクロールでは画面遷移後にスクロール位置を復元したいことがあります。  
これもservice側のsignalsで状態管理することで簡単に実現できます。  

```ts:api.service.ts
@Injectable({
  providedIn: 'root'
})
export class ApiService {
  scrollY = signal(0);
}
```

コンポーネント側で以下のように`scrollY`を更新/復元することができます。   

```ts:timeline.component.ts
@Component({
  selector: 'app-timeline',
})
export class TimelineComponent implements OnInit, AfterViewInit {
  scrollY = computed(() => this.apiService.scrollY());
  @HostListener('window:scroll', ['$event'])
  onScroll() {
    // スクロール位置を更新
    this.apiService.scrollY.update(() => window.scrollY);
  }

  ngOnInit() {
    // 初回読み込みは y = 0
    if (this.items().length == 0) {
      this.apiService.scrollY.update(() => 0);
    }
  }

  // スクロール位置を復元
  ngAfterViewInit(): void {
    this.viewportScroller.scrollToPosition([0, this.apiService.scrollY()]);
  }
}
```


# まとめ
Angular signalsを使った無限スクロールを実装しました。  
コードはこちら。  

https://github.com/nao50/zenn-infinite-scroll
