---
post_title: 'Day 6｜使用者互動：事件處理與 Signal 初探'
author1: '阿蘇'
post_slug: 'day06-user-interaction-signal'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Angular
  - Signals
  - EventBinding
  - TemplateSyntax
ai_note: 'Assisted by AI and reviewed by the author.'
summary: '在英雄列表中加入點擊事件，使用 signal 管理選中狀態，並以 @if 顯示當前選中英雄。為 Day07 的 ngModel 編輯鋪路。'
post_date: '2025-09-20'
---

哈囉，各位邦友們！
昨天用 @for 把英雄清單渲染到畫面，也學會 track、$index、@empty。
今天來加上使用者互動，讓點擊列表中的英雄時，會在下方畫面顯示選中資訊。

## 今天要做什麼？
1. 在 App 新增 `selectedHero`（signal<Hero | null>）與 `onSelect(hero)`。
2. 在 `<li>` 綁定 `(click)`，點擊時設定選中狀態。
3. 用 `[class.selected]` 標示選中項目，並用 `@if` 顯示選中資訊。
4. 再次點擊同一項目可取消選取（設回 null）。

## 前置需求
- 已完成 [Day05](https://ithelp.ithome.com.tw/articles/10383468) 的 `heroes` 清單並正常渲染。
- 專案可 `ng serve`，`<app-hero-badge>` 正常顯示。

**一、在 App 建立選中狀態與事件處理**

```ts
// src/app/app.ts

// ...existing code...

export class App {
  // ...existing code...

  // 新增：目前選中的英雄
  protected readonly selectedHero = signal<Hero | null>(null);

  // ...existing code...

  // 新增：點擊處理
  onSelect(hero: Hero) {
    const cur = this.selectedHero();
    this.selectedHero.set(cur?.id === hero.id ? null : hero);
  }
}
```

**二、更新範本：綁定點擊、樣式與顯示選中資訊**

```html
<!-- src/app/app.html -->
<!-- ...existing code... -->
<section>
  <h2>Heroes</h2>
  <ul>
    @for (h of heroes(); track h.id; let i = $index; let c = $count) {
      <li
        (click)="onSelect(h)"
        [class.is-a]="h.rank === 'A' || h.rank === 'S'"
        [class.selected]="selectedHero()?.id === h.id"
        [attr.data-id]="h.id"
        [attr.aria-current]="selectedHero()?.id === h.id ? 'true' : null">
        <span class="no">{{ i + 1 }}/{{ c }}</span>
        <span class="name">{{ h.name }}</span>
        @if (h.rank) { <span class="rank">[{{ h.rank }}]</span> }
      </li>
    } @empty {
      <li class="muted">No heroes.</li>
    }
  </ul>

  @if (selectedHero(); as s) {
    <aside class="panel">
      <h3>Selected</h3>
      <p>
        #{{ s.id }} - {{ s.name }}
        @if (s.rank) { <span class="rank">[{{ s.rank }}]</span> }
      </p>
    </aside>
  }
</section>
<!-- ...existing code... -->
```

**三、新增樣式**

```scss
// src/app/app.scss
/* ...existing code... */
.selected {
  outline: 2px solid #58a;
  background: #eef6ff;
}

.panel {
  margin-top: 12px;
  padding: 8px 12px;
  border: 1px solid #dde6f2;
  border-radius: 6px;
  background: #fafcff;
}
```

**驗收清單：**
- 點擊任一項目，該項加上 `.selected` 樣式並於下方顯示選中資訊。
![https://ithelp.ithome.com.tw/upload/images/20250920/20159238OJNXpddmCF.png](https://ithelp.ithome.com.tw/upload/images/20250920/20159238OJNXpddmCF.png)
- 若再次點擊同一個則取消選取
![https://ithelp.ithome.com.tw/upload/images/20250920/20159238leBiZ1WmvT.png](https://ithelp.ithome.com.tw/upload/images/20250920/20159238leBiZ1WmvT.png)
- 重新點擊其他項目，選中狀態正確切換。
![https://ithelp.ithome.com.tw/upload/images/20250920/20159238oeNizcU0Ix.png](https://ithelp.ithome.com.tw/upload/images/20250920/20159238oeNizcU0Ix.png)
- `@empty` 在清單為空時顯示「No heroes.」。
![https://ithelp.ithome.com.tw/upload/images/20250920/201592386Nk91rNrmm.png](https://ithelp.ithome.com.tw/upload/images/20250920/201592386Nk91rNrmm.png)

**常見錯誤與排查：**
- 畫面未更新：Signal 需以函式呼叫，請使用 `selectedHero()` 而非屬性存取。
- 事件無效：檢查 `(click)` 綁定與 `onSelect` 方法名稱是否一致。
- 樣式未生效：確認類別名稱 `.selected` 與綁定一致。

**今日小結：**
今天學會用 signal 管理選中狀態，並透過事件綁定與 @for/@if 搭配 `[class.selected]` 完成互動與視覺回饋。
明天會學習使用 FormsModule，用 ngModel 在「Selected」面板直接編輯英雄名稱，實作雙向綁定的即時更新。

**參考資料：**
- 事件綁定（Angular 官方文件）:
  [https://dev.angular.tw/guide/templates/event-binding](https://dev.angular.tw/guide/templates/event-binding)
- Control Flow（控制流）: 
  [https://dev.angular.tw/guide/templates/control-flow](https://dev.angular.tw/guide/templates/control-flow)
- Signals（訊號）:
  [https://dev.angular.tw/guide/signals](https://dev.angular.tw/guide/signals)