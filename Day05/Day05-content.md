# Day 5｜範本語法：@for 迴圈與 track

哈囉，各位邦友們！
昨天我們把單一 hero 綁到畫面，並用 @if/@else 來控制顯示。
今天把單一資料改成多筆，學習用 @for 來渲染。

### 今天要做什麼？
1. 在 App 建立 heroes 清單（signal<Hero[]>）
2. 用 @for 渲染清單，學會使用track以及$index、$count
3. 用 @empty 處理空清單狀態
#### 前置需求
* 專案可 ng serve
* Day04 的 hero 物件與 `<app-hero-badge>` 正常

**一、建立 heroes 清單**
```ts
// src/app/app.ts
import { Component, signal } from '@angular/core';
import { HeroBadge } from './hero-badge/hero-badge';

type Hero = { id: number; name: string; rank?: string };

@Component({
  selector: 'app-root',
  imports: [HeroBadge],
  templateUrl: './app.html',
  styleUrl: './app.scss'
})
export class App {
  protected readonly title = signal('hero-journey');

  // 英雄清單
  protected readonly heroes = signal<Hero[]>([
    { id: 11, name: 'Dr Nice', rank: 'B' },
    { id: 12, name: 'Narco', rank: 'A' },
    { id: 13, name: 'Bombasto' },
    { id: 14, name: 'Celeritas', rank: 'S' }
  ]);

//   constructor() {
//     this.heroes.set([]);
//   }
}
```

**二、範本用 @for + track + @empty**
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

**三、新增樣式）**
```scss
// src/app/app.scss
/* ...existing code... */
ul { 
    list-style: none; 
    padding: 0; 
    margin: 0;
}

li { 
    display: flex; 
    gap: 8px; 
    align-items: baseline; 
    padding: 4px 0; 
}

.no { 
    width: 56px;
    color: #789;
}

.name { 
    font-weight: 600;
}

.rank { 
    color: #445;
    background: #eef;
    padding: 2px 6px;
    border-radius: 4px;
}
/* ...existing code... */
```

**重點：**
* @for 基本形態：@for (item of list; track key; let i = $index) { ... } @empty { ... }
* track請用穩定唯一的 id；否則插入/排序會造成不必要的 DOM 重建。

**驗收清單：**
* 畫面顯示 Heroes 與 4 筆清單，含序號與 rank 標示。
* rank 為 A 或 S 的項目具 is-a 樣式。
* 將 heroes.set([]) 後，顯示「No heroes.」。

**常見錯誤與排查：**
* heroes 是 signal，範本需呼叫 heroes() 而非 heroes。

**今日小結：**
現在你已經會用 @for 渲染清單，並學會track以及$index、$count等變數的使用，其他的變數可參考[官網](https://dev.angular.tw/api/core/@for)的內容做學習。
明天會加入事件處理：點擊列表項目選擇英雄，並用 signal 管理「目前選中」的狀態。