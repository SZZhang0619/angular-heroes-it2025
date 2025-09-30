---
post_title: 'Day 16｜非同步處理：RxJS Operators 即時體驗'
author1: '阿蘇'
post_slug: 'day-16-rxjs-operators-search'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - RxJS
  - Operators
  - Search
  - switchMap
  - debounceTime
  - distinctUntilChanged
  - HttpClient
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '在 HeroService 與 Heroes 頁導入搜尋流程，利用 Subject 搭配 debounceTime/distinctUntilChanged/switchMap，打造即時又節流的英雄搜尋體驗。'
post_date: '2025-09-30'
---

哈囉，各位邦友們！
昨天我們在 Heroes 頁加上 Loading 與 Empty 狀態，讓使用者能明確知道目前資料的狀態。
今天則是要來實作即時搜尋，會透過 RxJS 的 `Subject` 與 `debounceTime`、`distinctUntilChanged`、`switchMap` 等操作符，將輸入內容轉成 HTTP 的查詢流程。

## 今天要做什麼？
1. 在 `HeroService` 新增 `search$()`，呼叫 in-memory API 並同步快取。
2. 在 `HeroesComponent` 建立 `Subject` 搜尋流，運用 RxJS Operators 去做節流。
3. 更新 Heroes 模板與樣式，顯示搜尋中的 Spinner、查詢結果與錯誤提示。

## 前置需求
- 已完成 [Day15](https://ithelp.ithome.com.tw/articles/10390073) 的 Loading / Empty 狀態，並有 `LoadingSpinner` 共用元件。
- 專案有在 [Day12](https://ithelp.ithome.com.tw/articles/10388716) 接上 in-memory 假後端，`HeroService` 具備 `getAll$()`、`getById$()` 等方法。

**一、HeroService：新增 `search$()` 串接查詢 API**
把搜尋邏輯放進服務，並沿用快取機制，讓常用英雄在其他頁面也能即時取得。

```typescript
// src/app/hero.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { delay, of, tap } from 'rxjs';

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
      delay(2000),
      tap((heroes) => {
        this.cache.clear();
        for (const hero of heroes) {
          this.cache.set(hero.id, hero);
        }
      })
    );
  }

  // ...getById$ / create$ / update$ / delete$ 與原本相同...

  search$(term: string) {
    const keyword = term.trim();
    if (!keyword) {
      return of<Hero[]>([]);
    }

    const params = new HttpParams().set('name', keyword);

    return this.http.get<Hero[]>(this.baseUrl, { params }).pipe(
      tap((heroes) => {
        for (const hero of heroes) {
          this.cache.set(hero.id, hero);
        }
      })
    );
  }
}
```

**重點：**
- 透過 `HttpParams` 組出 `?name=keyword` 查詢，在 in-memory web API 會以 `name` 欄位進行前綴比對。
- 空字串直接回傳 `of([])`，避免多餘請求，並且能讓前端快速清空結果。
- 用 `tap()` 同步快取，之後若從 detail 頁訪問同一位英雄便能直接讀取快取，省下一次 API 查詢。

**二、HeroesComponent：建立 `Subject` 與 RxJS 資料流**
我們將鍵盤輸入交給 `Subject`，再透過 Operators 進行 trim。

```typescript
// src/app/heroes/heroes.component.ts
import { Component, DestroyRef, effect, inject, signal } from '@angular/core';
import { HeroService, Hero } from '../hero.service';
import { FormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { RouterModule } from '@angular/router';
import { HeroBadge } from '../hero-badge/hero-badge';
import {
  EMPTY,
  Subject,
  catchError,
  debounceTime,
  distinctUntilChanged,
  finalize,
  map,
  of,
  startWith,
  switchMap,
  tap,
} from 'rxjs';
import { LoadingSpinner } from '../ui/loading-spinner/loading-spinner';

@Component({
  selector: 'app-heroes',
  imports: [HeroBadge, FormsModule, RouterModule, LoadingSpinner],
  templateUrl: './heroes.component.html',
  styleUrl: './heroes.component.scss',
})
export class HeroesComponent {
  // ...既有狀態 (list / selection / CRUD signals)...
  protected readonly heroes = signal<Hero[]>([]);
  protected readonly selectedHero = signal<Hero | null>(null);
  protected readonly heroesLoading = signal(true);
  protected readonly heroesError = signal<string | null>(null);
  protected readonly editName = signal('');
  protected readonly saving = signal(false);
  protected readonly saveError = signal<string | null>(null);
  protected readonly newHeroName = signal('');
  protected readonly creating = signal(false);
  protected readonly createError = signal<string | null>(null);
  protected readonly deletingId = signal<number | null>(null);
  protected readonly deleteError = signal<string | null>(null);
  protected readonly feedback = signal<string | null>(null);

  private readonly heroService = inject(HeroService);
  private readonly destroyRef = inject(DestroyRef);
  private readonly searchTerms = new Subject<string>();
  protected readonly searchKeyword = signal('');
  protected readonly searchResults = signal<Hero[]>([]);
  protected readonly searching = signal(false);
  protected readonly searchError = signal<string | null>(null);

  constructor() {
    // 既有：載入英雄清單並寫入狀態
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

    this.searchTerms
      .pipe(
        map((term) => term.trim()),
        startWith(''),
        debounceTime(300),
        distinctUntilChanged(),
        tap((term) => {
          this.searchKeyword.set(term);
          this.searchError.set(null);
          this.searchResults.set([]);
        }),
        switchMap((term) => {
          if (!term) {
            this.searching.set(false);
            return of<Hero[]>([]);
          }

          this.searching.set(true);
          return this.heroService.search$(term).pipe(
            catchError((err) => {
              this.searchError.set(String(err ?? 'Unknown error'));
              return of<Hero[]>([]);
            })
          );
        }),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe((heroes) => {
        this.searchResults.set(heroes);
        this.searching.set(false);
      });
  }

  // ...既有 CRUD 方法 (onSelect / saveSelected / addHero / removeHero)...

  search(term: string) {
    this.searchTerms.next(term);
  }
}
```

**說明：**
- `startWith('')` 讓流程自動初始化，畫面開啟時搜尋結果為空陣列。
- `debounceTime(300)` 避免每個鍵盤事件都打 API，`distinctUntilChanged()` 能忽略連續相同輸入。
- `switchMap()` 可自動取消前一次尚未完成的 HTTP 呼叫，確保最新輸入永遠優先。
- `catchError` 將錯誤轉成空結果，並寫入 `searchError` 讓畫面顯示錯誤訊息。

**三、範本：顯示搜尋輸入、結果與狀態**
在列表上方加入搜尋區，包含輸入框、載入指示、錯誤提示與搜尋結果清單。

```html
<!-- src/app/heroes/heroes.component.html -->
<section class="search" role="search">
  <label for="hero-search">Search heroes</label>
  <input
    id="hero-search"
    type="search"
    placeholder="輸入關鍵字 (至少 1 個字)"
    autocomplete="off"
    (input)="search($any($event.target).value)" />

  @if (searching()) {
    <app-loading-spinner label="Searching..."></app-loading-spinner>
  } @else if (searchKeyword()) {
    <p class="muted">搜尋「{{ searchKeyword() }}」</p>
  }

  @if (searchError(); as err) {
    <p class="error">Search failed: {{ err }}</p>
  }

  @if (searchKeyword() && !searching()) {
    <ul class="results">
      @for (hero of searchResults(); track hero.id) {
        <li>
          <button type="button" (click)="onSelect(hero)">
            {{ hero.name }}
            @if (hero.rank) { <span class="rank">[{{ hero.rank }}]</span> }
          </button>
          <a [routerLink]="['/detail', hero.id]" (click)="$event.stopPropagation()">View</a>
        </li>
      } @empty {
        <li class="muted">找不到符合「{{ searchKeyword() }}」的英雄。</li>
      }
    </ul>
  }
</section>

<!-- 既有的 @if (heroesLoading()) ... 保持不變 -->
```

**重點：**
- `@if (searchKeyword())` 控制結果區塊，空字串時自動收起清單。
- 點擊結果時沿用現有的 `onSelect()` 邏輯，同步更新選取區與 `routerLink` 詳細頁。

**四、新增樣式：搜尋區域與搜尋結果**

```scss
/* src/app/heroes/heroes.component.scss */
.search {
  margin-bottom: 24px;
  padding: 16px;
  border: 1px solid #e2e6f0;
  border-radius: 8px;
  background: #ffffff;
  display: grid;
  gap: 12px;
}

.search label {
  font-weight: 600;
  color: #344054;
}

.search input {
  padding: 8px 12px;
  border-radius: 6px;
  border: 1px solid #ccd5e4;
  font-size: 1rem;
}

.search .results {
  display: flex;
  flex-direction: column;
  gap: 8px;
  list-style: none;
  padding: 0;
  margin: 0;
}

.search .results li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 10px 12px;
  border: 1px solid #eef1f9;
  border-radius: 8px;
  background: #fafbff;
}

.search .results li button {
  flex: 1;
  text-align: left;
  border: none;
  background: transparent;
  color: #1b365d;
  font-weight: 600;
  cursor: pointer;
}

.search .results li button:hover {
  text-decoration: underline;
}

.search .results li a {
  font-size: 0.9rem;
  color: #4a6078;
}

.search .results li .rank {
  margin-left: 4px;
  font-size: 0.85rem;
}
```

**驗收清單：**
- 快速輸入「ma」再刪除，觀察僅送出最後一次請求，搜尋期間會顯示 `Searching...` Spinner。
![https://ithelp.ithome.com.tw/upload/images/20250930/20159238m9jledAdlH.png](https://ithelp.ithome.com.tw/upload/images/20250930/20159238m9jledAdlH.png)
- 輸入存在的英雄（例如 `nar`）會在 300ms 後列出結果，點擊可立即在右側選取區看到對應英雄資訊。
![https://ithelp.ithome.com.tw/upload/images/20250930/20159238TxTfAL2Wok.png](https://ithelp.ithome.com.tw/upload/images/20250930/20159238TxTfAL2Wok.png)
- 清除輸入框內容後，搜尋結果區塊自動收起並恢復完整清單顯示。
![https://ithelp.ithome.com.tw/upload/images/20250930/20159238gM44wRONKQ.png](https://ithelp.ithome.com.tw/upload/images/20250930/20159238gM44wRONKQ.png)

**常見錯誤與排查：**
- 搜尋永遠沒有結果：檢查是否有把 `HttpParams` 加入 `get` 呼叫，或是 in-memory `heroes` 集合是否存在對應名稱。
- Spinner 不會消失：記得在 `switchMap` 的空字串分支回傳 `of([])` 並把 `searching` 設為 `false`。

**今日小結：**
我們把鍵盤輸入轉成事件流，並透過 RxJS Operators，實作出即時搜尋功能。
這些技巧同樣適用於表單的其他情境，是前端掌握畫面互動與資料處理的一環。

**參考資料：**
- RxJS `debounceTime`：
  [https://rxjs.dev/api/operators/debounceTime](https://rxjs.dev/api/operators/debounceTime)
- RxJS `distinctUntilChanged`：
  [https://rxjs.dev/api/operators/distinctUntilChanged](https://rxjs.dev/api/operators/distinctUntilChanged)
- RxJS `switchMap`：
  [https://rxjs.dev/api/operators/switchMap](https://rxjs.dev/api/operators/switchMap)
- Subject：
  [https://rxjs.dev/guide/subject](https://rxjs.dev/guide/subject)