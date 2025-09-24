---
post_title: 'Day 9｜非同步處理：RxJS Observable 入門'
author1: '阿蘇'
post_slug: 'day-09-rxjs-observable-basics'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - RxJS
  - Observable
  - Async
  - takeUntilDestroyed
  - Signals
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '將 HeroService 的資料讀取改為 Observable，於元件中訂閱並處理 Loading 與 Error，為 Day10 路由與 Day12 HTTP 鋪路。'
post_date: '2025-09-23'
---

哈囉，各位邦友們！
昨天我們把資料抽到 HeroService，並用 inject() 在元件中取用。
今天來試著把同步的 getAll 改為 Observable，並在元件中處理 loading 與 error，學習非同步資料流的使用。

## 今天要做什麼？
1. 在 HeroService 新增 `getAll$()`，使用 RxJS 模擬非同步。
2. 在 App 元件訂閱資料流，管理 `loading/error` 狀態。
3. 在範本顯示 Loading／Error／List 的 UI。

## 前置需求
- 已完成 [Day05](https://ithelp.ithome.com.tw/articles/10383468)（清單）、[Day06](https://ithelp.ithome.com.tw/articles/10384071)（選取）、[Day07](https://ithelp.ithome.com.tw/articles/10384956)（編輯）。
- 已完成 [Day08](https://ithelp.ithome.com.tw/articles/10385262)（HeroService 與 inject()）。
- 專案可 ng serve，清單與互動正常。

**一、HeroService：將目前同步資料包成 Observable 模擬延遲**
```ts
// src/app/hero.service.ts
// 新增：引入 rxjs 工具
import { delay, of, throwError } from 'rxjs';
// ...existing code...

@Injectable({ providedIn: 'root' })
export class HeroService {
  // ...existing code...

  // 新增：以 Observable 回傳資料，加入 delay 模擬網路延遲
  getAll$() {
    return of(this.getAll()).pipe(delay(2000));
    // return throwError(() => new Error(`此為人工製造錯誤`));
  }

  // ...existing code...
}
```

**重點：**
- 使用 `of()` 將目前同步資料包成 Observable。
- 使用 `delay(2000)` 模擬 2 秒網路延遲。

**二、App 元件：訂閱資料流與狀態管理**
```ts
// src/app/app.ts
import { Component, DestroyRef, inject, signal } from '@angular/core';
// ...existing code...
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  // ...existing code...
})
export class App {
  // 注入服務與 DestroyRef
  private readonly heroService = inject(HeroService);
  private readonly destroyRef = inject(DestroyRef);

  // 狀態：標題、英雄清單、目前選中的英雄、載入、錯誤
  protected readonly title = signal('hero-journey');
  protected readonly heroes = signal<Hero[]>([]);
  protected readonly selectedHero = signal<Hero | null>(null);
  protected readonly heroesLoading = signal(true);
  protected readonly heroesError = signal<string | null>(null);

  constructor() {
    // 從 Observable 取得資料
    this.heroService
      .getAll$()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (list) => {
          this.heroes.set(list);
          this.heroesLoading.set(false);
        },
        error: (err) => {
          this.heroesError.set(String(err ?? 'Unknown error'));
          this.heroesLoading.set(false);
        }
      });
  }

  // ...existing code...
}
```

**重點：**
- 使用 `takeUntilDestroyed(this.destroyRef)` 自動解除訂閱。
- 初始化 `heroesLoading = true`，完成或失敗後關閉 loading。

**三、範本 UI：Loading／Error／List**
```html
<!-- src/app/app.html -->
<!-- ...existing code... -->

@if (heroesLoading()) {
  <p class="muted">Loading heroes...</p>
} @else if (heroesError(); as e) {
  <p class="error">Load failed: {{ e }}</p>
} @else {
  <section>
    <!-- ...existing list code... -->
  </section>
}

<!-- ...existing code... -->
```

**四、樣式：錯誤狀態**
```scss
/* src/app/app.scss */
/* ...existing code... */
.error {
  color: #c33;
}
/* ...existing code... */
```

**驗收清單：**
- 首次載入時先顯示「Loading heroes...」，約 2 秒後出現清單。
![https://ithelp.ithome.com.tw/upload/images/20250923/20159238fJ6R2vKwoX.png](https://ithelp.ithome.com.tw/upload/images/20250923/20159238fJ6R2vKwoX.png)
- 人工製造錯誤時（把 `getAll$()` 改為 `throwError`），
  會顯示「Load failed: xxx」。
![https://ithelp.ithome.com.tw/upload/images/20250923/20159238obgxHlFTBC.png](https://ithelp.ithome.com.tw/upload/images/20250923/20159238obgxHlFTBC.png)
- 既有互動與編輯（updateName）仍可運作。
![https://ithelp.ithome.com.tw/upload/images/20250923/20159238x34Zpz8rhA.png](https://ithelp.ithome.com.tw/upload/images/20250923/20159238x34Zpz8rhA.png)

**常見錯誤與排查：**
- 忘記引入 `takeUntilDestroyed`：從 `@angular/core/rxjs-interop` 匯入。
- 畫面不更新：在範本中使用 `heroes()` 與 `heroesLoading()`。
- 訂閱未釋放：確認有使用 `takeUntilDestroyed(this.destroyRef)`。
- 型別錯誤：統一從 `hero.service.ts` 匯出 `Hero` 型別。

**今日小結：**
今天把 HeroService 從原本的同步讀取改成用 Observable 的 getAll$()，試著用訂閱的方式來處理非同步資料，並且搭配 signals 來管理 loading 和 error 狀態。
明天會學習如何加上路由，讓載入流程在切換頁面的時候能順暢地銜接。

## 參考資料
- RxJS Observable：
  [https://rxjs.dev/guide/observables](https://rxjs.dev/guide/observable)
- takeUntilDestroyed（自動解除訂閱）：
  [https://dev.angular.tw/api/core/rxjs-interop/takeUntilDestroyed](https://dev.angular.tw/api/core/rxjs-interop/takeUntilDestroyed)
- RxJS of：
  [https://rxjs.dev/api/index/function/of](https://rxjs.dev/api/index/function/of)
- RxJS delay：
  [https://rxjs.dev/api/operators/delay](https://rxjs.dev/api/operators/delay)