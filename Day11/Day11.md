---
post_title: 'Day 11｜路由參數綁定：withComponentInputBinding 與 Detail 頁'
author1: '阿蘇'
post_slug: 'day-11-router-params-to-input'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
	- Angular
tags:
	- Routing
	- Router
	- withComponentInputBinding
	- ActivatedRoute
	- RouterLink
	- Standalone
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '新增 /detail/:id 路由，透過 withComponentInputBinding 將路由參數自動綁到元件 @Input（或 signal input），完成英雄詳細頁並從清單導覽。'
post_date: '2025-09-25'
---

哈囉，各位邦友們！
昨天完成了 provideRouter 與兩個頁面（Dashboard / Heroes）。
今天要把「路由參數」接進來，實作 `/detail/:id` 英雄詳情頁。
重點是使用 `withComponentInputBinding()` 讓路由參數自動綁到元件的 `input()` signal，不需要手動從 `ActivatedRoute` 取值。

## 今天要做什麼？
1. 以 CLI 建立 `HeroDetail`，並在元件內以 `withComponentInputBinding()` 自動接收 `id`，查詢服務並顯示資料。
2. 在 `app.routes.ts` 加入 `detail/:id` 路由。
3. 在 `Heroes` 清單加上連結導向詳細頁。
4. 驗證直接輸入網址 `/detail/12` 也能正確顯示。

## 前置需求
- 已完成 [Day10](https://ithelp.ithome.com.tw/articles/10386810) 的路由基礎配置，並於 `app.config.ts` 註冊：
	`provideRouter(routes, withComponentInputBinding())`。
- 有 `HeroService` 與 `getAll$()` 或同步資料來源可供查詢。

**一、服務：新增單筆查詢**

```typescript
// src/app/hero.service.ts
import { delay, map, of, throwError } from 'rxjs';'rxjs';
// ...existing code...

@Injectable({ providedIn: 'root' })
export class HeroService {
  // ...existing code...

	getById$(id: number) {
		return of(null).pipe(
			delay(200), // 模擬非同步
			map(() => this.getAll().find(h => h.id === id))
		);
	}

  // ...existing code...
}

```

**二、建立 HeroDetail 元件**
```sh
ng g c hero-detail
```

```typescript
// src/app/hero-detail/hero-detail.component.ts
import { Component, DestroyRef, effect, inject, input, signal } from '@angular/core';
import { Hero, HeroService } from '../hero.service';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { RouterModule } from '@angular/router';

@Component({
  selector: 'app-hero-detail',
  imports: [RouterModule],
  templateUrl: './hero-detail.html',
  styleUrl: './hero-detail.scss',
})
export class HeroDetail {
  // 由 withComponentInputBinding() 自動把 route param `id` 綁進來
  readonly id = input.required<number>();

  private readonly heroService = inject(HeroService);
  private readonly destroyRef = inject(DestroyRef);

  // 狀態
  readonly hero = signal<Hero | null>(null);
  readonly loading = signal(true);
  readonly error = signal<string | null>(null);

  constructor() {
    // 當 id 改變時重新載入
    effect(() => {
      const curId = Number(this.id());
      this.loading.set(true);
      this.error.set(null);

      // 以 Observable 取得單筆
      const sub = this.heroService
        .getById$(curId)
        .pipe(takeUntilDestroyed(this.destroyRef))
        .subscribe({
          next: (h) => {
            console.log('hero', h);
            this.hero.set(h ?? null);
            this.loading.set(false);
          },
          error: (e) => {
            this.error.set(String(e ?? 'Unknown error'));
            this.loading.set(false);
          },
        });

      return () => sub.unsubscribe();
    });
  }
}
```

```html
<!-- src/app/hero-detail/hero-detail.html -->
@if (loading()) {
	<p class="muted">Loading detail...</p>
} @else if (error(); as e) {
	<p class="error">Load failed: {{ e }}</p>
} @else if (hero(); as h) {
	<section class="detail">
		<h2>Hero #{{ h.id }}</h2>
		<p>
			<strong>{{ h.name }}</strong>
			@if (h.rank) { <span class="rank">[{{ h.rank }}]</span> }
		</p>
		<a routerLink="/heroes">← Back to list</a>
	</section>
} @else {
	<p class="muted">No hero found.</p>
}
```

```scss
/* src/app/hero-detail/hero-detail.scss */
.detail {
  padding: 8px 0;
}

.muted {
  color: #888;
}

.error {
  color: #c33;
}

.rank {
  color: #445;
  background: #eef;
  padding: 2px 6px;
  border-radius: 4px;
}
```

**三、在路由加入 detail/:id**

```typescript
// src/app/app.routes.ts
import { Route } from "@angular/router";
import { DashboardComponent } from "./dashboard/dashboard.component";
import { HeroesComponent } from "./heroes/heroes.component";
import { HeroDetail } from "./hero-detail/hero-detail";

export const routes: Route[] = [
  { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
  { path: 'dashboard', component: DashboardComponent, title: 'Dashboard' },
  { path: 'heroes', component: HeroesComponent, title: 'Heroes' },
  { path: 'detail/:id', component: HeroDetail, title: 'Hero Detail' },
];
```

**四、Heroes 清單加入導覽連結**
維持原本點擊 li 的行為，再額外提供「檢視」連結前往詳細頁。

```typescript
// src/app/heros.component.ts
// ...existing code...
import { RouterModule } from '@angular/router';

@Component({
  selector: 'app-heroes',
  imports: [FormsModule, RouterModule],
  templateUrl: './heroes.component.html',
  styleUrl: './heroes.component.scss',
})
export class HeroesComponent {
	// ...existing code...
}
	
```

```html
<!-- src/app/heroes/heroes.component.html -->
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

		<span class="actions">
			<a [routerLink]="['/detail', h.id]" (click)="$event.stopPropagation()">View</a>
		</span>
	</li>
} @empty {
	<li class="muted">No heroes.</li>
}
```

**驗收清單：**
- 於 `/heroes` 點擊某個項目的「View」能前往 `/detail/:id`。
![https://ithelp.ithome.com.tw/upload/images/20250925/20159238xbtcNx5yUE.png](https://ithelp.ithome.com.tw/upload/images/20250925/20159238xbtcNx5yUE.png)![https://ithelp.ithome.com.tw/upload/images/20250925/20159238goOCop8E2Z.png](https://ithelp.ithome.com.tw/upload/images/20250925/20159238goOCop8E2Z.png)
- 變更網址中 id（例如從 12 改 13）會重新載入對應資料。
![https://ithelp.ithome.com.tw/upload/images/20250925/20159238Cs9ZpxbCsh.png](https://ithelp.ithome.com.tw/upload/images/20250925/20159238Cs9ZpxbCsh.png)![https://ithelp.ithome.com.tw/upload/images/20250925/201592387CFvNWJAxI.png](https://ithelp.ithome.com.tw/upload/images/20250925/201592387CFvNWJAxI.png)
- 未找到資料時顯示「No hero found.」。
![https://ithelp.ithome.com.tw/upload/images/20250925/20159238pSQc9tlFir.png](https://ithelp.ithome.com.tw/upload/images/20250925/20159238pSQc9tlFir.png)

**常見錯誤與排查：**
- 未啟用 `withComponentInputBinding()`：請檢查 `app.config.ts` 的 `provideRouter(routes, withComponentInputBinding())` 是否存在。
- 路由參數名稱不一致：路由需為 `detail/:id`，而元件的 input 也必須命名為 `id` 才能自動綁定。
- 檔案匯入錯誤：`HeroDetail` 路徑、`app.routes.ts` 匯入與宣告是否一致。

**今日小結：**
我們完成了 `/detail/:id` 的詳細頁，並學會用 `withComponentInputBinding()` 讓路由參數無痛對應到元件輸入。
明天會開始整合 `HttpClient`，把資料來源換成真正的 HTTP 呼叫。

**參考資料：**
- withComponentInputBinding:
  [https://dev.angular.tw/api/router/withComponentInputBinding](https://dev.angular.tw/api/router/withComponentInputBinding)
- Signal Inputs:
  [https://dev.angular.tw/guide/components/inputs](https://dev.angular.tw/guide/components/inputs)
- RouterLink:
  [https://dev.angular.tw/api/router/RouterLink](https://dev.angular.tw/api/router/RouterLink)