---
post_title: 'Day 10｜路由系統：provideRouter'
author1: '阿蘇'
post_slug: 'day-10-routing-provide-router'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Routing
  - Router
  - RouterLink
  - RouterOutlet
  - provideRouter
  - withComponentInputBinding
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '建立 app.routes.ts、註冊 provideRouter、加入 router-outlet 與導航，完成 Dashboard 與 Heroes 的頁面切換，為 Day11 的路由參數綁定鋪路。'
post_date: '2025-09-24'
---

哈囉，各位邦友們！
昨天把資料流改為 Observable 並處理了 Loading/Error。
今天來開始加入路由吧！
從建立儀表板（Dashboard）與英雄列表（Heroes）兩個頁面開始，再到設定 provideRouter 並在 App 中加入導航與 router-outlet，讓專案具備基本的頁面切換能力。
另外也會預先啟用 withComponentInputBinding()，讓之後能直接示範將路由參數綁定至 @Input。

## 今天要做什麼？
1. 建立 `Dashboard` 與 `Heroes` 兩個頁面元件（Standalone）。
2. 建立 app.routes.ts，定義 `dashboard` 與 `heroes` 兩個路由，並設定預設導向。
3. 在 app.config.ts 註冊 `provideRouter(routes, withComponentInputBinding())`。
4. 在 App 元件加入 `<nav>` 與 `<router-outlet>`，提供導航與渲染插槽。
5. 驗證點擊切換頁面、重新整理可維持路由狀態。

## 前置需求
- 已完成 [Day05～Day09](https://ithelp.ithome.com.tw/users/20159238/ironman/8553)，專案可 ng serve。

**一、建立頁面元件（Dashboard 與 Heroes）**
```sh
ng g c dashboard
```

```typescript
// src/app/dashboard/dashboard.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'app-dashboard',
  imports: [],
  templateUrl: './dashboard.component.html',
  styleUrl: './dashboard.component.scss'
})
export class DashboardComponent {

}
```

```html
<!-- src/app/dashboard/dashboard.component.html -->
<section>
  <h2>Dashboard</h2>
  <p class="muted">Welcome! 從這裡開始導航你的英雄世界。</p>
</section>
```

```sh
ng g c heros
```

```typescript
// src/app/heroes/heroes.component.ts
import { Component, DestroyRef, inject, signal } from '@angular/core';
import { HeroService, Hero } from '../hero.service';
import { FormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-heroes',
  imports: [FormsModule],
  templateUrl: './heroes.component.html',
  styleUrl: './heroes.component.scss',
})
export class HeroesComponent {
  // 注入服務與 DestroyRef
  private readonly heroService = inject(HeroService);
  private readonly destroyRef = inject(DestroyRef);

  // 狀態：英雄清單、目前選中的英雄、載入、錯誤
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
        },
      });
  }

  // 點擊處理
  onSelect(hero: Hero) {
    const cur = this.selectedHero();
    this.selectedHero.set(cur?.id === hero.id ? null : hero);
  }

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

```html
<!-- src/app/heroes/heroes.component.html -->
@if (heroesLoading()) {
  <p class="muted">Loading heroes...</p>
} @else if (heroesError(); as e) {
  <p class="error">Load failed: {{ e }}</p>
} @else {
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

        <label for="hero-name">Edit name：</label>
        <input
          id="hero-name"
          name="hero-name"
          type="text"
          placeholder="enter new name"
          required
          minlength="3"
          #nameCtrl="ngModel"
          [ngModel]="s.name"
          (ngModelChange)="updateName($event)"
          [attr.aria-invalid]="nameCtrl.invalid && nameCtrl.touched"
          aria-describedby="hero-name-errors" />

        @if (nameCtrl.invalid && nameCtrl.touched) {
          <ul id="hero-name-errors" class="errors">
            @if (nameCtrl.hasError('required')) {
              <li>請輸入名稱</li>
            }
            @if (nameCtrl.hasError('minlength')) {
              <li>至少 3 個字</li>
            }
          </ul>
        }
      </aside>
    }
  </section>
}
```

```scss
// src/app/heroes/heroes.component.scss
.error {
  color: #c33;
}

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

.is-a {
  color: #225;
  background: #eef;
  padding: 4px 8px;
  border-radius: 4px;
}

.muted {
  color: #888;
}

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

.errors {
  color: #c33;
}
```

**二、建立路由定義（app.routes.ts）**
```typescript
// src/app.routes.ts
import { Route } from "@angular/router";
import { DashboardComponent } from "./dashboard/dashboard.component";
import { HeroesComponent } from "./heroes/heroes.component";

export const routes: Route[] = [
  { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
  { path: 'dashboard', component: DashboardComponent, title: 'Dashboard' },
  { path: 'heroes', component: HeroesComponent, title: 'Heroes' },
];
```

**三、在 app.config.ts 註冊 provideRouter**
```typescript
// src/app.config.ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners, provideZonelessChangeDetection } from '@angular/core';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    // ...existing providers...
  ],
};
```

**四、App 元件加入 Router**
```typescript
// src/app/app.ts
import { Component, signal } from '@angular/core';
import { HeroBadge } from './hero-badge/hero-badge';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [HeroBadge, RouterLink, RouterLinkActive, RouterOutlet],
  templateUrl: './app.html',
  styleUrl: './app.scss',
})
export class App {
  protected readonly title = signal('hero-journey');
}
```

```html
<!-- src/app/app.html -->
<h1>Hello, {{ title() }}</h1>

<nav class="nav">
  <a routerLink="/dashboard" routerLinkActive="active" ariaLabel="nav-dashboard">Dashboard</a>
  <a routerLink="/heroes" routerLinkActive="active" ariaLabel="nav-heroes">Heroes</a>
</nav>

<main class="page">
  <router-outlet></router-outlet>
</main>

<!-- 保留 Day03 範例 -->
<app-hero-badge></app-hero-badge>
```

```scss
/* src/app/app.scss */
/* ...existing code... */
.nav {
  display: flex;
  gap: 12px;
  margin: 12px 0;
}

.nav .active {
  font-weight: 700;
  text-decoration: underline;
}

.page {
  padding: 8px 0;
}
```

**驗收清單：**
- 進入根路徑時自動導向 `/dashboard`。
![https://ithelp.ithome.com.tw/upload/images/20250924/20159238OCUTsAVqUr.png](https://ithelp.ithome.com.tw/upload/images/20250924/20159238OCUTsAVqUr.png)
- 點擊導覽連結可在 Dashboard 與 Heroes 之間切換。
![https://ithelp.ithome.com.tw/upload/images/20250924/20159238NArAbZD4qm.png](https://ithelp.ithome.com.tw/upload/images/20250924/20159238NArAbZD4qm.png)
- 重新整理仍停留在當前路由。

**常見錯誤與排查：**
- 沒有加入 `provideRouter(...)`：請檢查 app.config.ts。
- App 未匯入 `RouterOutlet/RouterLink/RouterLinkActive`：請確認 App 元件的 imports。
- 路徑錯誤或檔案命名不一致：請比對路由檔與頁面元件的匯入路徑。
- 連結沒有樣式變化：檢查 `routerLinkActive="active"` 與樣式是否一致。

**今日小結：**
已完成路由設定、導航與頁面切換。
明天接著將在讓路由參數直接綁定到元件的 @Input，實作 `/detail/:id`。

**參考資料：**
- provideRouter：
  [https://dev.angular.tw/api/router/provideRouter](https://dev.angular.tw/api/router/provideRouter)
- RouterLink：
  [https://dev.angular.tw/api/router/RouterLink](https://dev.angular.tw/api/router/RouterLink)
- RouterOutlet：
  [https://dev.angular.tw/api/router/RouterOutlet](https://dev.angular.tw/api/router/RouterOutlet)
- Routing：
  [https://dev.angular.tw/guide/routing](https://dev.angular.tw/guide/routing)