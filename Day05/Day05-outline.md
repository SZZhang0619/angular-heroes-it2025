# Day 5｜模板語法：@for 迴圈與 track

哈囉，各位邦友們！
昨天我們把單一 hero 綁到畫面、並用 @if/@else 控制顯示。今天把資料擴充為清單，學會用全新的 @for 渲染列表，並從第一天就用好用滿的 track，養成最佳化的好習慣。

### 今天要做什麼？
1. 在 App 建立 heroes 清單（signal<Hero[]>）
2. 用 @for 渲染清單，示範 $index/$count
3. 正確使用 track 以穩定 DOM（以 id 作為 key）
4. 用 @empty 處理空清單狀態

#### 前置需求
* 專案可 ng serve，Day04 的 hero 與 `<app-hero-badge>` 正常

---

一、建立 heroes 清單（signal）
```ts
// src/app/app.ts
import { Component, signal } from '@angular/core';
import { HeroBadge } from './hero-badge/hero-badge';
// ...existing code...

type Hero = { id: number; name: string; rank?: string };

@Component({
  selector: 'app-root',
  imports: [HeroBadge],
  templateUrl: './app.html',
  styleUrl: './app.scss'
})
export class App {
  protected readonly title = signal('hero-journey');
  // ...existing code...

  // 英雄清單（重點：用 signal 管理陣列狀態）
  protected readonly heroes = signal<Hero[]>([
    { id: 11, name: 'Dr Nice', rank: 'B' },
    { id: 12, name: 'Narco', rank: 'A' },
    { id: 13, name: 'Bombasto' },
    { id: 14, name: 'Celeritas', rank: 'S' }
  ]);

  // 提示（可留作之後互動使用）：以不可變方式更新
  // this.heroes.set([...this.heroes(), { id: 15, name: 'Magneta' }]);
}
```

二、範本用 @for + track + @empty
```html
<!-- src/app/app.html -->
<h1>Hello, {{ title() }}</h1>

<section>
  <h2>Heroes</h2>
  <ul>
    @for (h of heroes(); track h.id; let i = $index; let c = $count) {
      <li
        [class.is-a]="h.rank === 'A' || h.rank === 'S'"
        [attr.data-id]="h.id">
        <span class="no">{{ i + 1 }}/{{ c }}</span>
        <span class="name">{{ h.name }}</span>
        @if (h.rank) { <span class="rank">[{{ h.rank }}]</span> }
      </li>
    } @empty {
      <li class="muted">No heroes.</li>
    }
  </ul>
</section>

<!-- 保留 Day03 範例 -->
<app-hero-badge></app-hero-badge>
```

三、樣式（簡化可讀性）
```scss
/* src/app/app.scss */
/* ...existing code... */
ul { list-style: none; padding: 0; margin: 0; }
li { display: flex; gap: 8px; align-items: baseline; padding: 4px 0; }
.no { width: 56px; color: #789; }
.name { font-weight: 600; }
.rank { color: #445; background: #eef; padding: 2px 6px; border-radius: 4px; }
.is-a { color: #225; }
.muted { color: #888; }
```

重點說明：
* @for 基本形態：@for (item of list; track key; let i = $index) { ... } @empty { ... }
* track 選擇穩定且唯一的 key，建議使用資料的 id；不要用 $index 當 key，重新排序會導致整行被重建。
* Signals 陣列請用不可變更新（set/更新新陣列）；就地 push/splice 不會觸發變更。

驗收清單：
* 畫面顯示 Heroes 標題與 4 筆清單，含序號與 rank 標示。
* rank 為 A 或 S 的項目具 is-a 樣式。
* 將 heroes.set([]) 後，出現「No heroes.」。

常見錯誤與排查：
* heroes 是 signal，範本要呼叫 heroes() 而不是 heroes。
* track 少寫或用 $index 當 key：重新排序或插入會造成不必要的重繪。
* 更新清單後沒反應：確認用了 set 或建立新陣列；避免直接 mutate 原陣列。

今日小結：
* 你已能用現代的 @for 渲染清單，並用 track 取得穩定且高效的 DOM 更新。明天開始處理互動事件，讓清單能跟著你的操作動起來！