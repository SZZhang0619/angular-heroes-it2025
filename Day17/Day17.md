---
post_title: 'Day 17｜整合與優化：提升使用者體驗'
author1: '阿蘇'
post_slug: 'day-17-challenge-day'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Exercise
  - Signals
  - RxJS
  - UI
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '整合服務快取、分類篩選、搜尋提示與共用元件，全面提升列表體驗。'
post_date: '2025-10-01'
---

哈囉，各位邦友們！
昨天我們把 Heroes 完成了即時搜尋，透過 `Subject`、`debounceTime`、`switchMap` 組成一條資料流。
目前累積使用了不少東西，因此今天將目前所學到的整合起來，除了新增分類篩選功能外，也進一步優化搜尋，並將常用到的程式片段共用元件化。

## 今天要做什麼？
1. 將快取移至服務層，並讓細節頁改用快取 API 避免重複請求
2. 增加分類篩選功能
3. 讓新增與編輯表單支援 Rank 下拉
4. 調整即時搜尋，加入最短字數、狀態提示與錯誤處理
5. 抽出共用 UI 元件（Badge、ListItem、MessageBanner）
6. 統一樣式色系提升一致性

## 前置需求
- 已完成 [Day16](https://ithelp.ithome.com.tw/articles/10391138) 的即時搜尋功能，並理解 `Subject` 與 `switchMap` 的使用方式。
- [Day08](https://ithelp.ithome.com.tw/articles/10385262) 建立的 `HeroService` 與 in-memory API 正常運作，可提供完整英雄清單。
- 理解 [Day05](https://ithelp.ithome.com.tw/articles/10383468) (`@for` / `track`) 與 [Day14](https://ithelp.ithome.com.tw/articles/10390010) (錯誤處理) 的程式碼運作邏輯。

**一、讓服務層與細節頁同步**
我們把快取邏輯搬到服務層，並讓細節頁改用新的 API，後續元件只要讀取 signal 就能取得資料。

```ts
// src/app/hero.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { computed, inject, Injectable, signal } from '@angular/core';
import { Observable, of, tap } from 'rxjs';

// ...existing code...

export class HeroService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = 'api/heroes';
  private readonly heroes = signal<Hero[]>([]);
  private readonly heroesById = computed(() => {
    const map = new Map<number, Hero>();
    for (const hero of this.heroes()) {
      map.set(hero.id, hero);
    }
    return map;
  });

  readonly heroesState = this.heroes.asReadonly();

  loadAll(): Observable<Hero[]> {
    return this.http.get<Hero[]>(this.baseUrl).pipe(
      tap((list) => this.heroes.set(list))
    );
  }

  getById(id: number): Observable<Hero> {
    const cached = this.heroesById().get(id);
    if (cached) {
      return of(cached);
    }

    return this.http.get<Hero>(`${this.baseUrl}/${id}`).pipe(
      tap((hero) => {
        this.heroes.update((current) => {
          const exists = current.some((item) => item.id === hero.id);
          return exists ? current : [...current, hero];
        });
      })
    );
  }

  create(hero: Pick<Hero, 'name' | 'rank'>): Observable<Hero> {
    const payload = {
      name: hero.name.trim(),
      ...(hero.rank ? { rank: hero.rank } : {}),
    } as Partial<Hero>;

    return this.http.post<Hero>(this.baseUrl, payload).pipe(
      tap((created) => {
        this.heroes.update((current) => [...current, created]);
      })
    );
  }

  update(id: number, changes: Partial<Hero>): Observable<Hero> {
    const cached = this.heroesById().get(id);
    const payload = { ...(cached ?? { id }), ...changes, id } as Partial<Hero> & { id: number };
    if (payload.rank === '' || payload.rank == null) {
      delete payload.rank;
    }

    return this.http.put<Hero>(`${this.baseUrl}/${id}`, payload).pipe(
      tap((updated) => {
        this.heroes.update((current) =>
          current.map((hero) => (hero.id === updated.id ? updated : hero))
        );
      })
    );
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`).pipe(
      tap(() => {
        this.heroes.update((current) => current.filter((hero) => hero.id !== id));
      })
    );
  }

  search$(term: string): Observable<Hero[]> {
    const keyword = term.trim();
    if (!keyword) {
      return of([]);
    }

    const params = new HttpParams().set('name', keyword);

    return this.http.get<Hero[]>(this.baseUrl, { params }).pipe(
      tap((heroes) => {
        if (!heroes.length) {
          return;
        }

        this.heroes.update((current) => {
          const map = new Map(current.map((hero) => [hero.id, hero] as const));
          for (const hero of heroes) {
            map.set(hero.id, hero);
          }
          return Array.from(map.values());
        });
      })
    );
  }

  // ...existing code...
}
```

```ts
// src/app/hero-detail/hero-detail.ts
// ...existing code...
export class HeroDetail {
  // ...existing code...

  private loadHero(id: number) {
    // ...existing code...
    this.heroService
      .getById(id)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (h) => {
          this.hero.set(h ?? null);
          this.loading.set(false);
        },
        // ...existing code...
      });
  }
}
```

**說明：**
- `signal` 與 `computed` 讓服務層可以同時提供陣列以及索引查找結果。
- `heroesState` 暴露唯讀 signal，元件可直接以 `computed` 或 `effect` 監聽。
- 細節頁改呼叫 `getById`，避免重複請求並拿掉開發時才需要的 `console.log`。

**二、英雄分類篩選按鈕**
接著在列表頁引入分類邏輯，同時讓搜尋結果也能套用相同條件。

```ts
// src/app/heroes/heroes.component.ts
import {
  Component,
  DestroyRef,
  computed,
  effect,
  inject,
  signal,
} from '@angular/core';
// ...existing code...
import { HeroListItem } from '../ui/hero-list-item/hero-list-item';
import { MessageBanner } from '../ui/message-banner/message-banner';

@Component({
  selector: 'app-heroes',
  imports: [
    HeroBadge,
    FormsModule,
    RouterModule,
    LoadingSpinner,
    MessageBanner,
    HeroListItem,
  ],
  templateUrl: './heroes.component.html',
  styleUrl: './heroes.component.scss',
})
export class HeroesComponent {
  private readonly heroService = inject(HeroService);
  private readonly destroyRef = inject(DestroyRef);

  protected readonly heroes = this.heroService.heroesState;
  protected readonly activeRank = signal<string>('ALL');
  protected readonly rankOptions = computed(() => {
    const ranks = new Set<string>();
    for (const hero of this.heroes()) {
      if (hero.rank) {
        ranks.add(hero.rank);
      }
    }
    return ['ALL', ...Array.from(ranks).sort()];
  });

  protected readonly filteredHeroes = computed(() => {
    const rank = this.activeRank();
    const list = this.heroes();
    if (rank === 'ALL') {
      return list;
    }
    return list.filter((hero) => hero.rank === rank);
  });

  protected readonly rawSearchResults = signal<Hero[]>([]);
  protected readonly filteredSearchResults = computed(() => {
    const rank = this.activeRank();
    const results = this.rawSearchResults();
    if (rank === 'ALL') {
      return results;
    }
    return results.filter((hero) => hero.rank === rank);
  });

  protected setRankFilter(option: string) {
    this.activeRank.set(option);
  }

  protected rankLabel(option: string) {
    return option === 'ALL' ? '全部' : option;
  }

  // ...existing code...
}
```

```html
<!-- src/app/heroes/heroes.component.html -->
@if (heroesLoading()) {
  <!-- ...existing code... -->
} @else if (heroesError(); as e) {
  <app-message-banner type="error">Load failed: {{ e }}</app-message-banner>
} @else {
  <section class="filters" aria-label="Filter heroes by rank">
    <span class="filters__label">分類：</span>
    <div class="filters__group">
      @for (option of rankOptions(); track option) {
        <button
          type="button"
          (click)="setRankFilter(option)"
          [class.active]="activeRank() === option">
          {{ rankLabel(option) }}
        </button>
      }
    </div>
  </section>

  <!-- ...existing code... -->
}
```

**說明：**
- 透過 `heroesState` 取得服務層快取後，再用 `computed` 做二次過濾即可。
- `filteredSearchResults` 與 `filteredHeroes` 共用相同的 Rank 條件，介面不會出現列表與搜尋結果不同步的狀況。
- `rankLabel` 保留 `ALL` 的易讀標籤，同時讓 HTML 中的按鈕維持簡潔。

**三、新增表單 Rank 下拉：讓新增與編輯可以設定Rank**
讓新增與編輯流程共用同一組 Rank 選項，並在儲存時避免不必要的 API 呼叫。

```ts
// src/app/heroes/heroes.component.ts
// ...existing code...
type HeroRank = '' | 'S' | 'A' | 'B' | 'C';

export class HeroesComponent {
  // ...existing code...
  private readonly fallbackRanks: HeroRank[] = ['S', 'A', 'B', 'C'];
  protected readonly formRankOptions = computed<HeroRank[]>(() => {
    const derived = this.rankOptions().filter((option) => option !== 'ALL') as HeroRank[];
    return derived.length ? derived : this.fallbackRanks;
  });

  protected readonly newHeroRank = signal<HeroRank>('');
  protected readonly editRank = signal<HeroRank>('');

  constructor() {
    // ...existing code...

    effect(() => {
      const options = this.formRankOptions();
      const current = this.newHeroRank();
      if (current !== '' && !options.includes(current) && options.length) {
        this.newHeroRank.set(options[0]);
      }
    });

    effect(() => {
      const selected = this.selectedHero();
      this.editName.set(selected?.name ?? '');
      this.editRank.set((selected?.rank as HeroRank) ?? '');
      this.saveError.set(null);
    });

    // ...existing code...
  }

  protected resetCreateForm() {
    this.newHeroName.set('');
    this.newHeroRank.set('');
  }

  protected addHero() {
    const name = this.newHeroName().trim();
    if (!name) {
      return;
    }

    const payload: Pick<Hero, 'name' | 'rank'> = {
      name,
      rank: this.newHeroRank() || undefined,
    };

    this.creating.set(true);
    this.createError.set(null);
    this.feedback.set(null);

    this.heroService
      .create(payload)
      .pipe(
        finalize(() => this.creating.set(false)),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe({
        next: (created) => {
          this.feedback.set('新增英雄成功！');
          this.resetCreateForm();
          this.selectedId.set(created.id);
        },
        error: (err) => {
          this.createError.set(String(err ?? 'Unknown error'));
        },
      });
  }

  protected saveSelected() {
    const hero = this.selectedHero();
    if (!hero) {
      return;
    }

    const name = this.editName().trim();
    const rank = this.editRank();
    if (name === hero.name && rank === (hero.rank ?? '')) {
      return;
    }

    this.saving.set(true);
    this.saveError.set(null);
    this.feedback.set(null);

    this.heroService
      .update(hero.id, { name, rank: rank || undefined })
      .pipe(
        finalize(() => this.saving.set(false)),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe({
        next: (updated) => {
          this.feedback.set('更新英雄成功！');
          this.selectedId.set(updated.id);
        },
        error: (err) => {
          this.saveError.set(String(err ?? 'Unknown error'));
        },
      });
  }

  // ...existing code...
}
```

```html
<!-- src/app/heroes/heroes.component.html -->
<!-- ...existing code... -->

<section class="create" id="create">
  <form (ngSubmit)="addHero()">
    <label for="new-hero">Name：</label>
    <!-- ...existing code... -->
    <label for="new-hero-rank">Rank：</label>
    <select
      id="new-hero-rank"
      name="new-hero-rank"
      [ngModel]="newHeroRank()"
      (ngModelChange)="newHeroRank.set($event)">
      <option [ngValue]="''">未指定</option>
      @for (rank of formRankOptions(); track rank) {
        <option [ngValue]="rank">{{ rankLabel(rank) }}</option>
      }
    </select>
    <button type="submit" [disabled]="creating() || newCtrl.invalid">
      @if (creating()) { Saving... } @else { Add }
    </button>
  </form>
  @if (createError(); as err) {
    <app-message-banner type="error">Create failed: {{ err }}</app-message-banner>
  }
</section>

<!-- ...existing code... -->

  @if (selectedHero(); as s) {
    <aside class="panel">
      <!-- ...existing code... -->
      <label for="hero-rank">Rank：</label>
      <select
        id="hero-rank"
        name="hero-rank"
        [ngModel]="editRank()"
        (ngModelChange)="editRank.set($event)">
        <option [ngValue]="''">未指定</option>
        @for (rank of formRankOptions(); track rank) {
          <option [ngValue]="rank">{{ rankLabel(rank) }}</option>
        }
      </select>
      <button
        type="button"
        (click)="saveSelected()"
        [disabled]="
          saving() ||
          nameCtrl.invalid ||
          (editName().trim() === s.name && editRank() === (s.rank ?? ''))
        ">
        @if (saving()) { Saving... } @else { Save }
      </button>
      @if (saveError(); as err) {
        <app-message-banner type="error">Save failed: {{ err }}</app-message-banner>
      }
    </aside>
  }
```

**說明：**
- `formRankOptions` 會優先使用列表中已存在的 Rank；如果資料較少則回到預設清單。
- `effect` 負責同步選單與 signal，確保切換英雄時 Rank 不會殘留舊值。
- 儲存前先比對名稱與 Rank，避免傳送完全相同的 payload。

**四、即時搜尋進階調整**
延續昨天的即時搜尋，加入最短字數、訊息提示與錯誤處理。

```ts
// src/app/heroes/heroes.component.ts
// ...existing code...
export class HeroesComponent {
  // ...existing code...
  private readonly searchTerms = new Subject<string>();
  protected readonly searchKeyword = signal('');
  protected readonly searchMessage = signal<string | null>(null);
  protected readonly searchError = signal<string | null>(null);
  protected readonly searching = signal(false);

  constructor() {
    // ...existing code...

    this.searchTerms
      .pipe(
        map((term) => term.trim()),
        debounceTime(300),
        distinctUntilChanged(),
        filter((term) => term.length === 0 || term.length >= 2),
        tap((term) => {
          this.searchKeyword.set(term);
          this.searchError.set(null);
          this.searchMessage.set(term ? '搜尋中...' : null);
          this.feedback.set(null);
          if (!term) {
            this.rawSearchResults.set([]);
          }
          this.searching.set(term.length > 0);
        }),
        switchMap((term) => {
          if (!term) {
            return of<Hero[]>([]);
          }

          return this.heroService.search$(term).pipe(
            tap((heroes) => {
              if (heroes.length) {
                this.searchMessage.set(`命中 ${heroes.length} 位英雄`);
              } else {
                this.searchMessage.set('沒有符合條件的英雄，試著換個關鍵字。');
              }
            }),
            catchError((err) => {
              this.searchError.set(String(err ?? 'Unknown error'));
              this.searchMessage.set('查詢失敗，可稍後重試。');
              return of<Hero[]>([]);
            }),
            finalize(() => this.searching.set(false))
          );
        }),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe((heroes) => {
        this.rawSearchResults.set(heroes);
        this.searching.set(false);
      });
  }

  protected search(term: string) {
    this.searchTerms.next(term);
  }

  // ...existing code...
}
```

```html
<!-- src/app/heroes/heroes.component.html -->
<!-- ...existing code... -->

<section class="search" role="search">
  <!-- ...existing code... -->
  @if (searching()) {
    <app-loading-spinner label="Searching..."></app-loading-spinner>
  }

  @if (searchError(); as err) {
    <app-message-banner type="error">Search failed: {{ err }}</app-message-banner>
  } @else if (searchMessage(); as msg) {
    <app-message-banner type="info">{{ msg }}</app-message-banner>
  }

  @if (searchKeyword() && !searching()) {
    <ul class="results">
      @for (hero of filteredSearchResults(); track hero.id) {
        <li>
          <app-hero-list-item
            [hero]="hero"
            [selectedId]="selectedId()"
            (pick)="onSelect($event)"></app-hero-list-item>
          <a [routerLink]="['/detail', hero.id]">View</a>
        </li>
      } @empty {
        <li class="muted">找不到符合「{{ searchKeyword() }}」的英雄。</li>
      }
    </ul>
  }
</section>

<!-- ...existing code... -->
```

**說明：**
- `filter` 決定只有兩個字以上才打 API，避免短字造成大量請求。
- `searchMessage` 專門負責顯示狀態提示，不需要再額外寫 `<p>` 或 `div`。
- 把結果寫回 `rawSearchResults` 後，再交給 Rank 篩選套用相同邏輯。

**五、抽出共用的 UI Component：統一互動樣式**
常見的 badge 與列表項目改成獨立元件，再加上一個 `MessageBanner` 管理訊息。

```ts
// src/app/hero-badge/hero-badge.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-hero-badge',
  standalone: true,
  imports: [],
  templateUrl: './hero-badge.html',
  styleUrl: './hero-badge.scss',
})
export class HeroBadge {
  // 顯示英雄等級，例如 'S' | 'A' | 'B' | 'C'
  readonly rank = input<string | undefined>();
}
```

```html
<!-- src/app/hero-badge/hero-badge.html -->
<span class="badge">
  @if (rank()) { {{ rank() }} } @else { New Hero }
</span>
```

```ts
// src/app/ui/hero-list-item/hero-list-item.ts
import { Component, computed, input, output } from '@angular/core';
import { Hero } from '../../hero.service';
import { HeroBadge } from '../../hero-badge/hero-badge';

@Component({
  selector: 'app-hero-list-item',
  standalone: true,
  imports: [HeroBadge],
  templateUrl: './hero-list-item.html',
  styleUrl: './hero-list-item.scss',
})
export class HeroListItem {
  readonly hero = input.required<Hero>();
  readonly selectedId = input<number | null>(null);
  readonly selected = computed(() => this.hero().id === this.selectedId());
  readonly pick = output<number>();

  triggerSelect() {
    this.pick.emit(this.hero().id);
  }
}
```

```html
<!-- src/app/ui/hero-list-item/hero-list-item.html -->
<button
  type="button"
  class="hero-list-item"
  (click)="triggerSelect()"
  [class.hero-list-item--active]="selected()">
  <span class="hero-list-item__name">{{ hero().name }}</span>
  @if (hero().rank) {
    <app-hero-badge [rank]="hero().rank"></app-hero-badge>
  }
</button>
```

```scss
// src/app/ui/hero-list-item/hero-list-item.scss
.hero-list-item {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid transparent;
  background: transparent;
  color: #1b365d;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease, border-color 0.2s ease;
}

.hero-list-item:hover,
.hero-list-item:focus-visible {
  border-color: #3b5ccc;
  background: #eef3ff;
  outline: none;
}

.hero-list-item--active {
  border-color: #2549b0;
  background: #dfe8ff;
}

.hero-list-item__name {
  flex: 1;
}
```

```ts
// src/app/ui/message-banner/message-banner.ts
import { Component, Input } from '@angular/core';
import { NgClass } from '@angular/common';

type BannerType = 'info' | 'error' | 'success';

@Component({
  selector: 'app-message-banner',
  standalone: true,
  imports: [NgClass],
  templateUrl: './message-banner.html',
  styleUrl: './message-banner.scss',
})
export class MessageBanner {
  @Input() type: BannerType = 'info';

  get roleAttr(): string | null {
    return this.type === 'error' ? 'alert' : null;
  }

  get ariaLiveAttr(): 'polite' | 'assertive' {
    return this.type === 'error' ? 'assertive' : 'polite';
  }
}
```

```html
<!-- src/app/ui/message-banner/message-banner.html -->
<div
  class="banner"
  [ngClass]="type"
  [attr.role]="roleAttr"
  [attr.aria-live]="ariaLiveAttr">
  <ng-content />
</div>
```

```scss
// src/app/ui/message-banner/message-banner.scss
.banner {
  margin: 12px 0;
  padding: 10px 14px;
  border-radius: 8px;
  border: 1px solid transparent;
  font-size: 0.95rem;
  line-height: 1.4;
}

.banner.info {
  background: #eef4ff;
  border-color: #ccdcff;
  color: #1d3a78;
}

.banner.error {
  background: #fdecea;
  border-color: #f2b8b5;
  color: #8b1b1b;
}

.banner.success {
  background: #edf9f2;
  border-color: #bfe5cd;
  color: #185c35;
}
```

**說明：**
- `HeroBadge` 接受可選的 Rank，缺值時顯示 `New Hero`，在列表與新增表單都能重用。
- `HeroListItem` 把名稱與 Badge 打包在一起，同時透過 `output` 回傳被選取的 ID。
- `MessageBanner` 統一 info/error/success 三種提示樣式，也處理對應的 ARIA 屬性。

**六、樣式整理**
最後整理 `heroes.component.scss`，讓新元件與按鈕有一致的色調。

```scss
// src/app/heroes/heroes.component.scss
.filters {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 20px;
}

.filters__group button {
  padding: 6px 12px;
  border-radius: 999px;
  border: 1px solid #d0d7ea;
  background: #fff;
  color: #3c4a69;
  cursor: pointer;
  transition: background-color 0.2s ease, color 0.2s ease, border-color 0.2s ease;
}

.filters__group button.active {
  background: #e4ebff;
  border-color: #5568d5;
  color: #1d2b6f;
}

.create {
  margin-bottom: 24px;
  padding: 16px;
  border: 1px solid #e2e6f0;
  border-radius: 8px;
  background: #fafbff;
  display: grid;
  gap: 12px;
}

.create form {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
}

.create select,
.create input {
  padding: 8px 12px;
  border-radius: 6px;
  border: 1px solid #ccd5e4;
  font-size: 1rem;
}

.create button {
  padding: 8px 16px;
  border-radius: 6px;
  border: 1px solid #3b5ccc;
  background: #3b5ccc;
  color: white;
  font-weight: 600;
  cursor: pointer;
}

.create button[disabled] {
  opacity: 0.6;
  cursor: not-allowed;
}

.list ul {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.list li {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 12px;
  padding: 8px 12px;
  border: 1px solid #e1e6f3;
  border-radius: 10px;
  background: #fff;
}

.selected {
  box-shadow: 0 0 0 2px #3b5ccc;
}

.actions {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

button.danger {
  padding: 6px 12px;
  border-radius: 6px;
  border: 1px solid #f1c4c1;
  background: #f7eceb;
  color: #a32020;
  cursor: pointer;
}

button.danger[disabled] {
  opacity: 0.7;
  cursor: progress;
}

.panel {
  margin-top: 16px;
  padding: 16px;
  border: 1px solid #dde6f2;
  border-radius: 8px;
  background: #fafcff;
  display: grid;
  gap: 12px;
}

.panel button {
  justify-self: start;
  padding: 8px 16px;
  border-radius: 6px;
  border: 1px solid #3b5ccc;
  background: #3b5ccc;
  color: white;
  font-weight: 600;
  cursor: pointer;
}

.panel button[disabled] {
  opacity: 0.6;
  cursor: not-allowed;
}

.muted {
  color: #888;
}
```

## 驗收清單
- 點擊 Rank 按鈕可正確過濾列表，選 ALL 時會顯示全部；切換 Rank 後，列表與搜尋結果保持一致，新增加或更新英雄後也會即時反映。
![https://ithelp.ithome.com.tw/upload/images/20251001/20159238F09lkMwroO.png](https://ithelp.ithome.com.tw/upload/images/20251001/20159238F09lkMwroO.png)
![https://ithelp.ithome.com.tw/upload/images/20251001/201592386CpFaQGsMX.png](https://ithelp.ithome.com.tw/upload/images/20251001/201592386CpFaQGsMX.png)
- 即時搜尋少於 2 字不會觸發請求，清空輸入時結果自動清除；搜尋中會顯示 Loading...，完成後依情況顯示命中或空結果，且搜尋結果會套用目前的 Rank 篩選。
![https://ithelp.ithome.com.tw/upload/images/20251001/20159238K0TA68LMhl.png](https://ithelp.ithome.com.tw/upload/images/20251001/20159238K0TA68LMhl.png)
![https://ithelp.ithome.com.tw/upload/images/20251001/20159238xfM71xcrn4.png](https://ithelp.ithome.com.tw/upload/images/20251001/20159238xfM71xcrn4.png)
- 新增英雄時選定的 Rank 會正確顯示在列表與徽章上。
![https://ithelp.ithome.com.tw/upload/images/20251001/20159238eJKPe6YAzY.png](https://ithelp.ithome.com.tw/upload/images/20251001/20159238eJKPe6YAzY.png)
![https://ithelp.ithome.com.tw/upload/images/20251001/201592382gD6VvysQu.png](https://ithelp.ithome.com.tw/upload/images/20251001/201592382gD6VvysQu.png)
- 選取英雄後變更 Rank 能正確更新，列表、細節與快取資料保持一致。
![https://ithelp.ithome.com.tw/upload/images/20251001/20159238UCII0srYRr.png](https://ithelp.ithome.com.tw/upload/images/20251001/20159238UCII0srYRr.png)
![https://ithelp.ithome.com.tw/upload/images/20251001/20159238GrE3IhGKIz.png](https://ithelp.ithome.com.tw/upload/images/20251001/20159238GrE3IhGKIz.png)

**常見錯誤與排查：**
- 分類後列表為空：檢查 `rankOptions` 是否包含 `ALL`，或資料中的 `rank` 欄位是否正確填寫。
- 搜尋訊息沒有更新：確認 `finalize` 仍會被呼叫，以及 `searching.set(false)` 是否在訂閱回呼中被覆寫。
- 共用元件沒有反應：確定父層已引入 `HeroListItem` 並傳入 `selectedId` 與 `pick` 事件。

**今日小結：**
今天我們優化了快取、篩選、表單、搜尋和 UI，讓流程更順也更好維護。

## 參考資料
- Angular Signals 指南：
  [https://dev.angular.tw/guide/signals](https://dev.angular.tw/guide/signals)
- RxJS Operators 速查：
  [https://rxjs.dev/guide/operators](https://rxjs.dev/guide/operators)
- Angular Standalone Component：
  [https://v17.angular.io/guide/standalone-components](https://v17.angular.io/guide/standalone-components)
