---
post_title: 'Day 22｜toSignal：Observable 與 Signal 整合'
author1: '阿蘇'
post_slug: 'day-22-to-signal'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Signals
  - RxJS
  - toSignal
  - StateManagement
  - Interop
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '示範如何利用 toSignal 將 Heroes 專案中的 RxJS 資料流轉成 signal，統一掌握 loading、error 與內容狀態。'
post_date: '2025-10-06'
---

哈囉，各位邦友們！
昨天我們把 `signal`、`computed` 與 `effect` 的分工重新梳理過一遍，讓資料在服務與元件之間的流向更清楚。
不過專案裡仍有幾段 RxJS pipeline，像是 HeroDetail 裡的 `loadHero()`、Heroes 搜尋的 `searchTerms`）需要手動 `.subscribe()` 才能把結果寫回 signal，loading／error 也得自己維護。
今天來透過`toSignal`把 Observable 世界的資料接上 signal，實現同一份狀態、兩種語法都能存取。

## 今天要做什麼？
1. 建立 `HeroDetailState`，用 `toSignal` 接手整段載入流程。
2. 把 Heroes 搜尋的 RxJS pipeline 收斂成 `SearchState` signal。
3. 調整模板判斷，直接對新的狀態物件宣告式渲染。

## 前置需求
- 完成 [Day21](https://ithelp.ithome.com.tw/articles/10393880) 的 Signals 三件套重構。
- 熟悉 [Day09](https://ithelp.ithome.com.tw/articles/10386399) 的 Observable 概念與運算子。
- HeroService 已具備 Day17~Day20 的快取、CRUD 與表單驗證重構成果。

### 一、`HeroDetail` 交給 `HeroDetailState` 應付所有情境
`HeroDetail` 的 `constructor` 先前會在 `id` 改變時手動呼叫 `loadHero()`，逐一 `.set()` loading／error／hero。
改寫後我們直接建立 `HeroDetailState` union 型別，並用 `toSignal` 把 `HeroService.getById()` 的 Observable 轉成 signal

```ts
// src/app/hero-detail/hero-detail.ts
import { Component, computed, inject, input, signal } from '@angular/core';
import { RouterModule } from '@angular/router';
import { NgOptimizedImage } from '@angular/common';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { catchError, map, of, startWith, switchMap } from 'rxjs';
import { Hero, HeroService } from '../hero.service';
import { LoadingSpinner } from '../ui/loading-spinner/loading-spinner';
import { MessageBanner } from '../ui/message-banner/message-banner';

export type HeroDetailState =
  | { status: 'loading'; hero: null; error: null }
  | { status: 'ready'; hero: Hero; error: null }
  | { status: 'error'; hero: null; error: string };

@Component({
  selector: 'app-hero-detail',
  imports: [RouterModule, NgOptimizedImage, LoadingSpinner, MessageBanner],
  templateUrl: './hero-detail.html',
  styleUrl: './hero-detail.scss',
})
export class HeroDetail {
  readonly id = input.required<number>();
  private readonly heroService = inject(HeroService);

  readonly heroState = toSignal(
    toObservable(this.id).pipe(
      switchMap((id) =>
        this.heroService.getById(id).pipe(
          map((hero) => ({ status: 'ready', hero, error: null } as const)),
          startWith({ status: 'loading', hero: null, error: null } as const),
          catchError((err) =>
            of({
              status: 'error',
              hero: null,
              error: String(err ?? 'Unknown error'),
            } as const)
          )
        )
      )
    ),
    { initialValue: { status: 'loading', hero: null, error: null } }
  );

  readonly hero = computed(() => this.heroState().hero);
  readonly loading = computed(() => this.heroState().status === 'loading');
  readonly error = computed(() => this.heroState().error);
  readonly avatarUrl = computed(() => {
    const detail = this.heroState();
    if (detail.status !== 'ready') {
      return null;
    }
    const seed = encodeURIComponent(detail.hero.name);
    return `https://api.dicebear.com/7.x/bottts-neutral/png?seed=${seed}&size=320&background=%23eef3ff`;
  });
}
```

**說明：**
- `toSignal` 會在元件銷毀時自動取消訂閱，不再需要 `takeUntilDestroyed`。
- loading／ready／error 被整合成單一狀態物件，Template 只要對著 `heroState()` 宣告式判斷。
- Avatar 來源改成從狀態物件讀取 hero，再用 `computed` 派生，避免未載入時就嘗試產圖。

template 的寫法也同步換成 `@switch`

```html
<!-- src/app/hero-detail/hero-detail.html -->
@switch (heroState().status) {
  @case ('loading') {
    <app-loading-spinner label="Loading hero..."></app-loading-spinner>
  }
  @case ('error') {
    <app-message-banner type="error">{{ heroState().error }}</app-message-banner>
  }
  @case ('ready') {
    @if (hero(); as currentHero) {
      <section class="detail">
        @if (avatarUrl(); as src) {
          <figure class="detail__media">
            <img
              [ngSrc]="src"
              width="320"
              height="320"
              [attr.fetchpriority]="currentHero.rank === 'S' ? 'high' : null"
              alt="{{ currentHero.name }} portrait" />
          </figure>
        }
        <h2>Hero #{{ currentHero.id }}</h2>
        <p>
          <strong>{{ currentHero.name }}</strong>
          @if (currentHero.rank) { <span class="rank">[{{ currentHero.rank }}]</span> }
        </p>
        <a routerLink="/heroes">← Back to list</a>
      </section>
    } @else {
      <p class="muted">No hero found.</p>
    }
  }
}
```

### 二、搜尋流程收斂成單一 `SearchState`
之前實作雖然補齊了快取與表單驗證，但搜尋那段 Observable 仍需要手動同步四顆 signal。
現在我們用一個 `SearchState` 物件包住 term、heroes、顯示訊息與錯誤，再利用 `toSignal` 直接把結果轉成 signal

```ts
// src/app/heroes/heroes.component.ts
import { takeUntilDestroyed, toSignal } from '@angular/core/rxjs-interop';
import {
  catchError,
  debounceTime,
  distinctUntilChanged,
  filter,
  map,
  of,
  startWith,
  switchMap,
  tap,
} from 'rxjs';

type SearchState =
  | { status: 'idle'; term: ''; heroes: Hero[]; message: string | null; error: null }
  | { status: 'loading'; term: string; heroes: Hero[]; message: string; error: null }
  | { status: 'success'; term: string; heroes: Hero[]; message: string; error: null }
  | { status: 'error'; term: string; heroes: Hero[]; message: string; error: string };

export class HeroesComponent {
  private readonly searchTerms = new Subject<string>();

  protected readonly searchState = toSignal(
    this.searchTerms.pipe(
      map((term) => term.trim()),
      debounceTime(300),
      distinctUntilChanged(),
      filter((term) => term.length === 0 || term.length >= 2),
      tap(() => this.feedback.set(null)),
      switchMap((term) => {
        if (!term) {
          return of<SearchState>({
            status: 'idle',
            term: '',
            heroes: [],
            message: null,
            error: null,
          });
        }

        return this.heroService.search$(term).pipe(
          map((heroes) => ({
            status: 'success',
            term,
            heroes,
            message: heroes.length
              ? `命中 ${heroes.length} 位英雄`
              : '沒有符合條件的英雄，試著換個關鍵字。',
            error: null,
          }) satisfies SearchState),
          startWith<SearchState>({
            status: 'loading',
            term,
            heroes: [],
            message: '搜尋中...',
            error: null,
          }),
          catchError((err) =>
            of<SearchState>({
              status: 'error',
              term,
              heroes: [],
              message: '查詢失敗，可稍後重試。',
              error: String(err ?? 'Unknown error'),
            })
          )
        );
      })
    ),
    {
      initialValue: {
        status: 'idle',
        term: '',
        heroes: [],
        message: null,
        error: null,
      } as SearchState,
    }
  );

  protected readonly searchMessage = computed(() => this.searchState().message);
  protected readonly searchError = computed(() => this.searchState().error);
  protected readonly searching = computed(() => this.searchState().status === 'loading');
  protected readonly searchKeyword = computed(() => this.searchState().term);
  protected readonly filteredSearchResults = computed(() => {
    const rank = this.activeRank();
    const results = this.searchState().heroes;
    if (rank === 'ALL') {
      return results;
    }
    return results.filter((hero) => hero.rank === rank);
  });
}
```
所有與搜尋有關的欄位都改成 `computed` 從 `searchState` 派生，不必再自己 `.set()`。

另外也將 `showEmpty` 改寫成 derived signal

```ts
// src/app/heroes/heroes.component.ts
protected readonly showEmpty = computed(() => {
  const baseList = this.filteredHeroes();
  const search = this.searchState();
  if (search.status === 'success' && search.term) {
    return baseList.length === 0 && search.heroes.length === 0;
  }
  return baseList.length === 0;
});
```

### 三、template 直接讀取狀態物件
有了新的狀態物件後，HTML 只需要用 `@switch` / `@if` 判斷 state，就能把 loading／error／空資料的呈現集中管理

```html
<!-- src/app/heroes/heroes.component.html -->
@switch (searchState().status) {
  @case ('loading') { <app-loading-spinner label="Searching..."></app-loading-spinner> }
  @case ('error') { <app-message-banner type="error">{{ searchError() }}</app-message-banner> }
  @case ('success') { <app-message-banner type="info">{{ searchMessage() }}</app-message-banner> }
  @case ('idle') { @if (searchMessage()) { <app-message-banner type="info">{{ searchMessage() }}</app-message-banner> } }
}

@if (searchKeyword() && searchState().status !== 'loading') {
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

@if (showEmpty()) {
  <li class="empty-state" aria-live="polite">
    <h3>目前還沒有英雄</h3>
    <p>點擊上方的 Add 建立第一位夥伴，或在 in-memory API 填入假資料。</p>
  </li>
}
```

搜尋結果區塊的訊息與 loading 整合在 `@switch`，列表尾端的空狀態則交給 `showEmpty()` 決定，避免 rank／搜尋結果互相打架。
這一輪重構後，`HeroDetail` 與 `Heroes` 都只需要讀取 signal，不再關心 Observable 的生命週期，之後就能讓其它服務也比照辦理。

**今日小結：**
`toSignal` 是串起 RxJS 與 Signals 的橋梁，只要把 Observable 的生命週期交給它，就能專心的處理狀態，HeroDetail 因此擺脫了 `loadHero()`，搜尋流程也從多個 signal 收斂成單一 `searchState`，搭配 `computed` 讓 template 讀取想要的內容。
接下來就能在維持同一份狀態的前提下，導入更多 RxJS 流或把資料換成真正的後端。

**參考資料：**
- `toSignal`：
  [https://dev.angular.tw/ecosystem/rxjs-interop](https://dev.angular.tw/ecosystem/rxjs-interop)
