---
post_title: 'Day 12｜HTTP 通訊：HttpClient 與模擬 API'
author1: '阿蘇'
post_slug: 'day-12-httpclient-in-memory-web-api'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
	- Angular
tags:
	- HttpClient
	- provideHttpClient
	- InMemoryWebApi
	- RxJS
	- Standalone
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '導入 provideHttpClient()，以 HttpClient 取代本地模擬資料，並使用 angular-in-memory-web-api 建立可開發用的假後端。為 Day13/14 的 CRUD 與錯誤處理奠基。'
post_date: '2025-09-26'
---

哈囉，各位邦友們！
昨天我們完成了 `/detail/:id`，並用 `withComponentInputBinding()` 讓路由參數直綁到元件。
今天會在專案中導入 `provideHttpClient()`，將 `HeroService` 改為透過 `HttpClient` 存取遠端 API，並先用 `angular-in-memory-web-api` 模擬後端。

## 今天要做什麼？
1. 安裝並設定 `angular-in-memory-web-api`，建立 `InMemoryDataService` 當作假後端。
2. 在 `app.config.ts` 啟用 `provideHttpClient(withFetch())`。
3. 重構 `HeroService`，以 `HttpClient` 實作 `getAll$()`、`getById$()`，並調整`updateName(id: number, name: string)`。
4. 驗證 Heroes 與 Detail 頁能從 HTTP 取得資料、保留 loading/error 顯示。

## 前置需求
- 已完成 [Day10](https://ithelp.ithome.com.tw/articles/10386810)（路由）與 [Day11](https://ithelp.ithome.com.tw/articles/10387939)（詳細頁）。
- `app.config.ts` 已註冊 `provideRouter(routes, withComponentInputBinding())`。

**一、安裝並建立 InMemory 假後端**
```sh
npm i -D angular-in-memory-web-api
ng g serviice InMemoryData
```

```typescript
// src/app/in-memory-data.ts
import { Injectable } from '@angular/core';
import { InMemoryDbService } from 'angular-in-memory-web-api';

export type Hero = { id: number; name: string; rank?: string };

@Injectable({
  providedIn: 'root'
})
export class InMemoryData implements InMemoryDbService {
	createDb() {
		const heroes: Hero[] = [
			{ id: 11, name: 'Dr Nice', rank: 'B' },
			{ id: 12, name: 'Narco', rank: 'A' },
			{ id: 13, name: 'Bombasto', rank: 'S' },
			{ id: 14, name: 'Celeritas', rank: 'A' },
			{ id: 15, name: 'Magneta', rank: 'B' },
		];
		return { heroes };
	}
}
```

**二、在 app.config.ts 啟用 HttpClient 與 InMemory Web API**

```typescript
// src/app.config.ts
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
      })
    ),
    provideClientHydration(withEventReplay())
  ],
};

```

**三、重構 HeroService：改用 HttpClient**
將原先模擬的 Observable 改為 HTTP 請求。

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

  updateName(id: number, name: string): Hero | undefined {
    const hero = this.cache.get(id);

    if (!hero) return undefined;

    const updated = { ...hero, name: name.trim() };
    this.cache.set(updated.id, updated);
    return updated;
  }
}
```

**驗收清單：**
- 進入 `/heroes` 能看到由 HTTP 取得的英雄清單。
![https://ithelp.ithome.com.tw/upload/images/20250926/20159238ZCo0TUsZjL.png](https://ithelp.ithome.com.tw/upload/images/20250926/20159238ZCo0TUsZjL.png)
- 點擊某項的 View 前往 `/detail/:id`，可顯示單筆資料。
![https://ithelp.ithome.com.tw/upload/images/20250926/201592387RmhiRHB7e.png](https://ithelp.ithome.com.tw/upload/images/20250926/201592387RmhiRHB7e.png)

**常見錯誤與排查：**
- 仍使用舊的模擬資料：確認 `HeroService` 已改為走 `HttpClient` 的 `getAll$()` 與 `getById$()`。

**今日小結：**
今天我們導入了 `provideHttpClient()`，用 in-memory 假後端讓專案順利切換到透過 HTTP 取得資料，並維持原有的 RxJS 與 UI 狀態管理。
明天會接著完成 CRUD ，讓專案有一條完整的資料作業流程。

**參考資料：**
- provideHttpClient:
  [https://dev.angular.tw/api/common/http/provideHttpClient](https://dev.angular.tw/api/common/http/provideHttpClient)
- HttpClient: 
  [https://dev.angular.tw/guide/http](https://dev.angular.tw/guide/http)
- angular-in-memory-web-api:
  [https://github.com/angular/in-memory-web-api](https://github.com/angular/in-memory-web-api)
- importProvidersFrom:
  [https://dev.angular.tw/api/core/importProvidersFrom](https://dev.angular.tw/api/core/importProvidersFrom)