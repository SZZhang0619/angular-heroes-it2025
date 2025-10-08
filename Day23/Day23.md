---
post_title: 'Day 23｜非同步：resource() API'
author1: '阿蘇'
post_slug: 'day-23-angular-resource'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Signals
  - Resource
  - RxJS
  - AsyncState
  - Experimental
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '介紹 Angular 實驗性的 resource() API，對照現行 Signals + RxJS 作法並示範如何在 Heroes 專案中重構非同步狀態。'
post_date: '2025-10-07'
---

哈囉，各位邦友們！
昨天我們把 Heroes 專案裡分散的 RxJS pipeline 收斂進 `toSignal`，讀寫邏輯簡潔不少。
但每次遇到非同步資料，還是得自建 loading／error／ready union 型別，並額外處理退訂與錯誤回復。
今天就在專案導入 `resource()` 與 `rxResource()`，透過`resource()` API 標準化這些繁瑣流程。

## 今天要做什麼？
1. 了解 `resource()`／`rxResource()` 如何用宣告式 API 管理非同步狀態與取消控制。
2. 調整 `HeroService.getById()` 支援 `AbortSignal`，讓框架能自動中斷請求。
3. 用 `rxResource()` 重構 `HeroDetail`，並更新模板讀取 `loading`／`error`／`value` 狀態。

## 前置需求
- 已完成 [Day22](https://ithelp.ithome.com.tw/articles/10394269) 的 `toSignal` 重構，熟悉目前的狀態模型。
- 了解 [Day09](https://ithelp.ithome.com.tw/articles/10386399) 的 Observable 概念。

---

### 一、`resource()` 與 `rxResource()` 想解決什麼痛點？
Signals 上線後，官方觀察到：
- 開發者仍需手刻 loading／error／ready union 型別。
- 需要自行處理取消訂閱或 `AbortController`。
- 要保留上一筆成功資料時還得維護快取欄位。

`resource()` 專為非同步狀態打造，提供 `value()`、`error()`、`isLoading()` 等 signal 介面；
`rxResource()` 則是針對 RxJS 情境的包裝，讓既有 Observable 服務不用改成 Promise，就能享受 `resource()` 的能力。

透過 DemoResourceComponent 來說名簡化後的概念：

```ts
import { Component, resource, signal } from '@angular/core';

@Component({
  selector: 'demo-resource',
  template: `
    @if (user.isLoading()) {
      <p>Loading user...</p>
    } @else if (user.error(); as err) {
      <p class="error">{{ err instanceof Error ? err.message : err }}</p>
    } @else if (user.value(); as detail) {
      <p>{{ detail.name }} ({{ detail.rank }})</p>
    }
    <button type="button" (click)="user.reload()">Reload</button>
  `,
})
export class DemoResourceComponent {
  private readonly selectedId = signal(1);

  readonly user = resource({
    params: () => this.selectedId(),
    loader: async ({ params, abortSignal }) => {
      const response = await fetch(`/api/heroes/${params}`, { signal: abortSignal });
      if (!response.ok) {
        throw new Error(`Failed to load hero #${params}`);
      }
      return (await response.json()) as { id: number; name: string; rank: string };
    },
    defaultValue: null,
  });

  select(id: number) {
    this.selectedId.set(id);
  }
}
```

**說明重點：**
- `params()`（或 `source`）會觀察 signal，變化時觸發 `loader()`。
- `loader()` 會自帶 `AbortSignal`，路由切換時可自動取消請求。
- `defaultValue`、`error()`、`isLoading()` 提供統一的 UI 狀態讀取接口。
- `reload()` 讓我們能以指令式方式重新拉資料。

### 二、讓 `HeroService.getById()` 支援 `AbortSignal`
`rxResource()` 在每次請求時都會提供 `abortSignal`，我們需要把它往 `HttpClient` 傳遞。
唯有 `withFetch()` 的 `HttpClient` 實作能支援 `AbortSignal`，專案已在 `app.config.ts` 啟用。

```ts
// src/app/hero.service.ts
getById(id: number, options?: { signal?: AbortSignal }): Observable<Hero> {
  const cached = this.heroesById().get(id);
  if (cached) {
    return of(cached);
  }

  const httpOptions: Record<string, unknown> = {};
  if (options?.signal) {
    (httpOptions as { signal?: AbortSignal }).signal = options.signal;
  }

  return this.http.get<Hero>(`${this.baseUrl}/${id}`, httpOptions).pipe(
    tap((hero) => {
      this.heroes.update((current) => {
        const exists = current.some((item) => item.id === hero.id);
        return exists ? current : [...current, hero];
      });
    })
  );
}
```

### 三、用 `rxResource()` 重構 `HeroDetail`
取代 Day22 的 `toSignal + switchMap` pipeline，讓 `rxResource()` 主動管理 loading 與錯誤狀態。

```ts
// src/app/hero-detail/hero-detail.ts
import { Component, computed, inject, input } from '@angular/core';
import { NgOptimizedImage } from '@angular/common';
import { RouterModule } from '@angular/router';
import { rxResource } from '@angular/core/rxjs-interop';

import { Hero, HeroService } from '../hero.service';
import { LoadingSpinner } from '../ui/loading-spinner/loading-spinner';
import { MessageBanner } from '../ui/message-banner/message-banner';

@Component({
  selector: 'app-hero-detail',
  imports: [RouterModule, NgOptimizedImage, LoadingSpinner, MessageBanner],
  templateUrl: './hero-detail.html',
  styleUrl: './hero-detail.scss',
})
export class HeroDetail {
  readonly id = input.required<number>();
  private readonly heroService = inject(HeroService);

  readonly heroResource = rxResource<Hero | null, number>({
    params: () => this.id(),
    stream: ({ params, abortSignal }) =>
      this.heroService.getById(params, { signal: abortSignal }),
    defaultValue: null,
  });

  readonly hero = computed<Hero | null>(() => this.heroResource.value());
  readonly loading = computed(() => this.heroResource.isLoading());
  readonly errorMessage = computed(() => {
    const err = this.heroResource.error();
    if (!err) {
      return null;
    }
    return err instanceof Error ? err.message : String(err);
  });
  readonly avatarUrl = computed(() => {
    const detail = this.hero();
    if (!detail) {
      return null;
    }

    const seed = encodeURIComponent(detail.name);
    return `https://api.dicebear.com/7.x/bottts-neutral/png?seed=${seed}&size=320&background=%23eef3ff`;
  });

  reload() {
    this.heroResource.reload();
  }
}
```

**要點說明：**
- `rxResource` 直接吃 Observable，不需再手動 `firstValueFrom`。
- `stream` 回傳 `HeroService.getById()`，同時把 `abortSignal` 往下傳。
- `error()` 不做預設處理，由 `errorMessage` 計算後提供給模板。
- `reload()` 可讓使用者重新整理細節資料。


### 四、template 改用 `resource()` 狀態
`hero-detail.html` 只需觀察 `loading()`、`errorMessage()`、`hero()`，維持宣告式風格。

```html
<!-- src/app/hero-detail/hero-detail.html -->
@if (loading()) {
  <app-loading-spinner label="Loading hero..."></app-loading-spinner>
} @else if (errorMessage(); as err) {
  <app-message-banner type="error">{{ err }}</app-message-banner>
} @else if (hero(); as detail) {
  <section class="detail">
    @if (avatarUrl(); as src) {
      <figure class="detail__media">
        <img
          [ngSrc]="src"
          width="320"
          height="320"
          [attr.fetchpriority]="detail.rank === 'S' ? 'high' : null"
          alt="{{ detail.name }} portrait" />
      </figure>
    }
    <header class="detail__header">
      <h2>{{ detail.name }}</h2>
      <small>Rank: {{ detail.rank ?? 'N/A' }}</small>
    </header>
    <a routerLink="/heroes">← Back to list</a>
  </section>
} @else {
  <p class="muted">No hero found.</p>
}

<button type="button" class="muted" (click)="reload()">重新整理</button>
```

`resource()` 會保留上一筆成功資料，因此在 loading 狀態下不會閃空白；若真的抓不到資料，就顯示提示文字。

**今日小結:**
今天透過 `resource()` API，將非同步狀態管理標準化，不再需要手刻 loading／error 處理邏輯。`rxResource()` 讓既有的 Observable 服務能無痛接入，`HeroDetail` 的狀態管理也因此大幅簡化。

**參考資料：**
- resource()：
  [https://dev.angular.tw/guide/signals/resource](https://dev.angular.tw/guide/signals/resource)
- firstValueFrom：
  [https://rxjs.dev/api/index/function/firstValueFrom](https://rxjs.dev/api/index/function/firstValueFrom)