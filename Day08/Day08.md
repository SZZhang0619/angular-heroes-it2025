---
post_title: 'Day 8｜架構設計：Service 與依賴注入'
author1: '阿蘇'
post_slug: 'day08-service-di-inject'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Angular
  - Service
  - DependencyInjection
  - inject
  - Signals
ai_note: 'Assisted by AI and reviewed by the author.'
summary: '把英雄資料搬到 HeroService，比較 constructor 與 inject() 注入方式並維持既有互動。'
post_date: '2025-09-22'
---

哈囉，各位邦友們！
昨天我們用 FormsModule 在 Selected 區塊即時編輯英雄名稱，同步更新清單與選取畫面。
接下來要把英雄資料抽到 HeroService 使其與元件分離，改為透過依賴注入（DI）在元件中取用。
這是走向真實專案架構的第一步。

## 今天要做什麼？
1. 建立 HeroService，將英雄資料從元件抽離。
2. 認識 DI 與 providedIn。
3. 在元件中使用 inject(HeroService) 取得服務。

## 前置需求
- 已完成 [Day05](https://ithelp.ithome.com.tw/articles/10383468)（清單）、[Day06](https://ithelp.ithome.com.tw/articles/10384071)（選取）、[Day07](https://ithelp.ithome.com.tw/articles/10384956)（編輯）。
- 專案可 ng serve，互動流程正常。

**一、建立 HeroService**
把資料與邏輯放進服務，讓元件專注於template的畫面顯示。

```sh
ng g service hero
```

```typescript
// src/app/hero.service.ts
import { Injectable } from '@angular/core';

export type Hero = { id: number; name: string; rank?: string };

@Injectable({ providedIn: 'root' })
export class HeroService {
  private readonly data: Hero[] = [
    { id: 11, name: 'Dr Nice', rank: 'B' },
    { id: 12, name: 'Narco', rank: 'A' },
    { id: 13, name: 'Bombasto' },
    { id: 14, name: 'Celeritas', rank: 'S' }
  ];

  getAll(): Hero[] {
    return this.data;
  }

  getById(id: number): Hero | undefined {
    return this.data.find((hero) => hero.id === id);
  }

  updateName(id: number, name: string): Hero | undefined {
    const hero = this.getById(id);
    if (!hero) return undefined;
    hero.name = name.trim();
    return hero;
  }
}
```

**二、在元件使用 inject() 取得服務**
以 inject() 取用 HeroService，初始化 heroes，並調整 updateName（）使其能更新 Service 內資料。

```typescript
// src/app/app.ts
import { Component, inject, signal } from '@angular/core';
import { HeroService, Hero } from './hero.service';
// ...existing code...

export class App {
  // 使用 inject() 取得 HeroService
  private readonly heroService = inject(HeroService);

  protected readonly title = signal('hero-journey');

  // 由服務提供初始資料
  protected readonly heroes = signal<Hero[]>(this.heroService.getAll());

  // 目前選中的英雄
  protected readonly selectedHero = signal<Hero | null>(null);

  // ...existing code...

  // 調整：讓服務處理邏輯，再回傳至元件將資料顯示
  updateName(name: string) {
    const selected = this.selectedHero();
    if (!selected) {
      return;
    }

    const updated = this.heroService.updateName(selected.id, name);
    if (!updated) {
      return;
    }

    this.heroes.update((list) =>
      list.map((hero) => (hero.id === selected.id ? { ...hero, name: updated.name } : hero))
    );
    this.selectedHero.set({ ...selected, name: updated.name });
  }
}
```

**說明：**
- providedIn: 'root'：在根級別提供服務，建立單個共享實例，並將其注入到任何請求它的類別中。
- Component.providers：在 @Component 級別提供服務，使服務可用於此元件的所有實例以及在範本中使用的其他元件和指令。
- 資料與處理邏輯放服務；畫面顯示與UX互動留在元件。

**驗收清單：**
- heroes 改由 HeroService 提供，畫面與互動維持不變。
- 編輯名稱可即時同步清單與 Selected 區塊。
![https://ithelp.ithome.com.tw/upload/images/20250921/20159238xwdKloo6FF.png](https://ithelp.ithome.com.tw/upload/images/20250921/20159238xwdKloo6FF.png)

**常見錯誤與排查：**
- 注入失敗：確認 @Injectable 與 providedIn: 'root' 設定正確。
- 畫面不更新：請使用 this.heroes.update() 與 this.selectedHero.set()，。
- 型別不一致：從 hero.service.ts 匯出並統一使用 Hero 型別。
- 匯入路徑錯誤：import { HeroService } from './hero.service' 檔名需對應。

**今日小結：**
今天初步理解了 DI 與 inject()，並且把資料抽到 HeroService 改由服務更新資料，再同步至 signals，使元件更乾淨。
明天會將服務改成回傳 Observable，學會處理非同步資料流與基本訂閱。

**參考資料：**
- 依賴注入（DI）:
  [https://dev.angular.tw/guide/di](https://dev.angular.tw/guide/di)
- inject:
  [https://dev.angular.tw/api/core/inject](https://dev.angular.tw/api/core/inject)
- Injectable:
  [https://dev.angular.tw/api/core/Injectable](https://dev.angular.tw/api/core/Injectable)
