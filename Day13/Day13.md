---
post_title: 'Day 13｜CRUD 實作：新增 (POST) 與更新 (PUT)'
author1: '阿蘇'
post_slug: 'day-13-crud-create-update'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - HttpClient
  - POST
  - PUT
  - RxJS
  - Signals
  - FormsModule
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '在 HeroService 實作 create$/update$，於 Heroes 頁整合表單與狀態，完成 POST/PUT 的資料流，為 Day14 的刪除與錯誤處理鋪路。'
post_date: '2025-09-27'
---

哈囉，各位邦友們！
昨天我們把資料讀取改成 `HttpClient` + in-memory 假後端。
今天要讓「新增」與「更新」也透過 POST/PUT 將使用者的輸入送到 API，並把回傳結果同步回畫面。

## 今天要做什麼？
1. 在 `HeroService` 新增 `create$()` 與 `update$()`，移除舊的同步 `updateName()` 實作。
2. 於 `HeroesComponent` 建立「編輯名稱」的儲存流程，避免每個字元都打出一個 HTTP 請求。
3. 在 Heroes 頁加入新增英雄表單，成功後自動更新清單與選取狀態。
4. 加上簡單的 `saving/creating`、錯誤訊息提示，讓 UI 回饋更明確。

## 前置需求
- 已完成 [Day12](https://ithelp.ithome.com.tw/articles/10388716) 的 `HttpClient` 導入與 in-memory 假後端。
- 專案可執行 `ng serve`，`/heroes` 與 `/detail/:id` 皆能載入資料。
- `HeroService` 仍維持昨天的 `getAll$()`、`getById$()` 與快取結構。

**一、HeroService：新增 create$/update$ 並更新快取**
```typescript
// src/app/hero.service.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { tap } from 'rxjs';

export type Hero = { id: number; name: string; rank?: string };

@Injectable({
  providedIn: 'root',
})
export class HeroService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = 'api/heroes';
  private readonly cache = new Map<number, Hero>();

  getAll$() {
    return this.http.get<Hero[]>(this.baseUrl).pipe(
      tap((heroes) => {
        this.cache.clear();
        for (const hero of heroes) {
          this.cache.set(hero.id, hero);
        }
      })
    );
  }

  getById$(id: number) {
    return this.http.get<Hero>(`${this.baseUrl}/${id}`).pipe(
      tap((hero) => {
        if (!hero) {
          return;
        }
        this.cache.set(hero.id, hero);
      })
    );
  }

  create$(name: string) {
    const payload = { name: name.trim() };
    return this.http.post<Hero>(this.baseUrl, payload).pipe(
      tap((created) => {
        this.cache.set(created.id, created);
      })
    );
  }

  update$(id: number, changes: Partial<Hero>) {
    const cached = this.cache.get(id);
    const payload = { ...(cached ?? { id }), ...changes, id };

    return this.http.put<Hero>(`${this.baseUrl}/${id}`, payload).pipe(
      tap((updated) => {
        this.cache.set(updated.id, updated);
      })
    );
  }
}
```

**二、HeroesComponent：建立儲存與新增的狀態流程**
```typescript
// src/app/heroes/heroes.component.ts
import { Component, DestroyRef, effect, inject, signal } from '@angular/core';
import { HeroService, Hero } from '../hero.service';
import { FormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { RouterModule } from '@angular/router';
import { HeroBadge } from '../hero-badge/hero-badge';

@Component({
  selector: 'app-heroes',
  imports: [HeroBadge, FormsModule, RouterModule],
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

  // 新增：編輯與儲存狀態
  protected readonly editName = signal('');
  protected readonly saving = signal(false);
  protected readonly saveError = signal<string | null>(null);

  // 新增：建立英雄表單狀態
  protected readonly newHeroName = signal('');
  protected readonly creating = signal(false);
  protected readonly createError = signal<string | null>(null);

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

    effect(() => {
      const current = this.selectedHero();
      this.editName.set(current?.name ?? '');
      this.saveError.set(null);
    });
  }

  // 點擊處理
  onSelect(hero: Hero) {
    this.selectedHero.set(hero);
  }

  saveSelected() {
    const current = this.selectedHero();
    if (!current) {
      return;
    }

    const name = this.editName().trim();
    if (name.length < 3 || name === current.name) {
      return;
    }

    this.saving.set(true);
    this.saveError.set(null);

    this.heroService
      .update$(current.id, { name })
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (updated) => {
          this.heroes.update((list) =>
            list.map((hero) => (hero.id === updated.id ? updated : hero))
          );
          this.selectedHero.set(updated);
          this.editName.set(updated.name);
          this.saving.set(false);
        },
        error: (err) => {
          this.saveError.set(String(err ?? 'Unknown error'));
          this.saving.set(false);
        },
      });
  }

  addHero() {
    const name = this.newHeroName().trim();
    if (name.length < 3) {
      return;
    }

    this.creating.set(true);
    this.createError.set(null);

    this.heroService
      .create$(name)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (created) => {
          this.heroes.update((list) => [...list, created]);
          this.newHeroName.set('');
          this.selectedHero.set(created);
          this.editName.set(created.name);
          this.creating.set(false);
        },
        error: (err) => {
          this.createError.set(String(err ?? 'Unknown error'));
          this.creating.set(false);
        },
      });
  }
}
```

**三、Heroes 模板：新增表單與儲存按鈕**
```html
<!-- src/app/heroes/heroes.component.html -->
@if (heroesLoading()) {
  <p class="muted">Loading heroes...</p>
} @else if (heroesError(); as e) {
  <p class="error">Load failed: {{ e }}</p>
} @else {
  <section class="create">
  <app-hero-badge></app-hero-badge>
    <form (ngSubmit)="addHero()">
      <label for="new-hero">Name：</label>
      <input
        id="new-hero"
        name="new-hero"
        placeholder="enter new hero"
        required
        minlength="3"
        #newCtrl="ngModel"
        [ngModel]="newHeroName()"
        (ngModelChange)="newHeroName.set($event)" />
      <button type="submit" [disabled]="creating() || newCtrl.invalid">
        @if (creating()) { Saving... } @else { Add }
      </button>
    </form>
    @if (createError(); as err) {
      <p class="error">Create failed: {{ err }}</p>
    }
  </section>

  <section class="list">
    <!-- 既有清單保留 Day10 寫法 -->
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
          <span class="actions">
            <a [routerLink]="['/detail', h.id]" (click)="$event.stopPropagation()">View</a>
          </span>
        </li>
      } @empty {
        <li class="muted">No heroes.</li>
      }
    </ul>

    @if (selectedHero(); as s) {
      <aside class="panel">
        <h3>Edit</h3>
        <p>
          #{{ s.id }} - {{ s.name }}
          @if (s.rank) { <span class="rank">[{{ s.rank }}]</span> }
        </p>

        <label for="hero-name">Name：</label>
        <input
          id="hero-name"
          name="hero-name"
          type="text"
          required
          minlength="3"
          #nameCtrl="ngModel"
          [ngModel]="editName()"
          (ngModelChange)="editName.set($event)"
          [attr.aria-invalid]="nameCtrl.invalid && nameCtrl.touched" />

        <button
          type="button"
          (click)="saveSelected()"
          [disabled]="saving() || nameCtrl.invalid || editName().trim() === s.name">
          @if (saving()) { Saving... } @else { Save }
        </button>

        @if (saveError(); as err) {
          <p class="error">Save failed: {{ err }}</p>
        }
      </aside>
    }
  </section>
}
```

**四、In-memory API：回傳更新值搭配預設資料**
```typescript
// src/app/app.config.ts
import { ApplicationConfig, importProvidersFrom, provideBrowserGlobalErrorListeners, provideZonelessChangeDetection } from '@angular/core';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { HttpClientInMemoryWebApiModule } from 'angular-in-memory-web-api';
import { routes } from './app.routes';
import { InMemoryData } from './in-memory-data';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideBrowserGlobalErrorListeners(),
    provideZonelessChangeDetection(),
    provideHttpClient(withFetch()),
    importProvidersFrom(
      HttpClientInMemoryWebApiModule.forRoot(InMemoryData, {
        dataEncapsulation: false,
        delay: 300,
        post204: false,
        put204: false,
      })
    ),
    provideClientHydration(withEventReplay())
  ],
};
```

```typescript
// src/app/in-memory-data.ts
import { Injectable } from '@angular/core';
import { InMemoryDbService } from 'angular-in-memory-web-api';

export type Hero = { id: number; name: string; rank?: string };

const DEFAULT_HEROES: Hero[] = [
  { id: 11, name: 'Dr Nice', rank: 'B' },
  { id: 12, name: 'Narco', rank: 'A' },
  { id: 13, name: 'Bombasto', rank: 'S' },
  { id: 14, name: 'Celeritas', rank: 'A' },
  { id: 15, name: 'Magneta', rank: 'B' },
];

@Injectable({
  providedIn: 'root',
})
export class InMemoryData implements InMemoryDbService {
  createDb() {
    return {
      heroes: DEFAULT_HEROES.map((hero) => ({ ...hero })),
    };
  }

  genId(collection: Hero[]): number {
    if (collection.length === 0) {
      return 11;
    }

    const maxId = collection.reduce(
      (max, hero) => (hero.id > max ? hero.id : max),
      0
    );

    return maxId + 1;
  }
}
```

**重點：**
- `post204/put204` 設為 `false`，確保 POST/PUT 會回傳實體資料，方便畫面同步更新。
- 假後端保留一份預設英雄清單，每次啟動時重新產生，對應練習需求即可。

**五、樣式：讓新增區域與錯誤訊息易讀**
```scss
/* src/app/heroes/heroes.component.scss */
/* ...existing code... */
.create {
  margin-bottom: 24px;
  padding: 16px;
  border: 1px solid #e2e6f0;
  border-radius: 8px;
  background: #fafbff;

  form {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
  }
}

.panel button {
  margin-top: 12px;
}

.error {
  color: #c33;
}
```

**驗收清單：**
- 於 Heroes 頁輸入名稱並按 Add，會看到新 ID 的英雄加入列表且自動選中。
![https://ithelp.ithome.com.tw/upload/images/20250927/20159238jtcACMbleA.png](https://ithelp.ithome.com.tw/upload/images/20250927/20159238jtcACMbleA.png)![https://ithelp.ithome.com.tw/upload/images/20250927/20159238hNExwnVGi1.png](https://ithelp.ithome.com.tw/upload/images/20250927/20159238hNExwnVGi1.png)
- 選中某個英雄後修改名稱並按 Save，列表與選取區同時更新。
![https://ithelp.ithome.com.tw/upload/images/20250927/20159238mDhlj0pXeF.png](https://ithelp.ithome.com.tw/upload/images/20250927/20159238mDhlj0pXeF.png)![https://ithelp.ithome.com.tw/upload/images/20250927/20159238Wdn0Bfit76.png](https://ithelp.ithome.com.tw/upload/images/20250927/20159238Wdn0Bfit76.png)
**常見錯誤與排查：**
- 按下按鈕毫無反應：檢查 `ngSubmit="addHero()"` 與 `(click)="saveSelected()"` 是否綁定正確，或是否因為 `minlength` 驗證沒通過。
- 儲存後畫面未更新：確保在訂閱成功回呼中有呼叫 `heroes.update(...)` 與 `selectedHero.set(updated)`。

**今日小結：**
我們讓 HeroService 能透過 POST/PUT 操作資料，並在 Heroes 頁建立對應資料出哩，讓使用者輸入後透過 HTTP 更新資料。
明天會接著完成 DELETE 與錯誤處理，讓 CRUD 告一段落。

**參考資料：**
- HttpClient POST：
  [https://dev.angular.tw/api/common/http/HttpClient#post](https://dev.angular.tw/api/common/http/HttpClient#post)
- HttpClient PUT：
  [https://dev.angular.tw/api/common/http/HttpClient#put](https://dev.angular.tw/api/common/http/HttpClient#put)
- In-memory Web API：
  [https://github.com/angular/in-memory-web-api](https://github.com/angular/in-memory-web-api)
- takeUntilDestroyed：
  [https://dev.angular.tw/api/core/rxjs-interop/takeUntilDestroyed](https://dev.angular.tw/api/core/rxjs-interop/takeUntilDestroyed)