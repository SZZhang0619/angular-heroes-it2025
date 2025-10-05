---
post_title: 'Day 21｜響應式系統：Signals 核心 (signal, computed, effect)'
author1: '阿蘇'
post_slug: 'day-21-angular-signals-core'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Signals
  - Computed
  - Effect
  - StateManagement
  - RxJS
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '回顧 Day17 起導入的 Signals 實作，拆解 heroes 資料在 service、component、RxJS 管線之間的流向與心法。'
post_date: '2025-10-05'
---

哈囉，各位邦友們！
在 [Day17](https://ithelp.ithome.com.tw/articles/10391966) 時我們已經有把 `computed` 與 `effect` 帶進 Heroes 專案，只是當時聚焦在功能面。
今天我們換個角度，回頭解構這些 signal 如何真的把資料串起來：從服務、列表狀態，到 effect 如何讓表單、搜尋與路由同步。

## 今天要做什麼？
1. 梳理 `HeroService` 中 `signal`/`computed` 的快取機制，以及它如何和 CRUD、搜尋互動。
2. 解析 `HeroesComponent` 裡的派生狀態與 fallback 邏輯，釐清畫面如何得知該顯示什麼資料。
3. 說明 `effect` 與 RxJS 在元件 constructor 內的合作，舉例表單同步、搜尋流程與細節頁載入。

## 前置需求
- 專案已完成 [Day20](https://ithelp.ithome.com.tw/articles/10393462) 的 Validators 與錯誤訊息實作。
- 熟悉 [Day06](https://ithelp.ithome.com.tw/articles/10384071) 的 signal 基本操作。
- Heroes 頁面可正常進行 CRUD、即時搜尋與路由導向。

**一、HeroService：signal 與 computed 打造的資料快取**
服務層是所有資料的單一來源，我們在這裡用 signal 記錄英雄清單，並用 `computed` 建立查表快取，讓各元件都能以同步方式取得最新資料。

```ts
// src/app/hero.service.ts
@Injectable({
  providedIn: 'root',
})
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
    const payload = { ...(cached ?? { id }), ...changes, id } as Partial<Hero> & {
      id: number;
    };
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
}
```

**這段程式的資料流程可以拆成幾步：**
1. 所有 HTTP 請求回來後都透過 `this.heroes.update()` 或 `set()` 改寫 `signal`，這代表每個元件看到的都是同一份狀態。
2. `computed heroesById` 會在 `heroes` 更新時同步更新，讓我們作更新或細節查詢時不用手動維護 Map。
3. 搜尋結果利用 `Map` 合併到既有快取中，避免請求結果覆蓋掉先前載入的清單。

**二、HeroesComponent：computed 組裝畫面需要的視圖**
進到 HeroesComponent，我們沒有直接操作陣列，而是以 `computed` 描述畫面狀態，並提供 fallback 避免空資料。

```ts
// src/app/heroes/heroes.component.ts
export class HeroesComponent {
  private readonly heroService = inject(HeroService);
  private readonly destroyRef = inject(DestroyRef);
  private readonly fb = inject(FormBuilder);

  protected readonly heroes = this.heroService.heroesState;
  protected readonly heroesLoading = signal(true);
  protected readonly heroesError = signal<string | null>(null);

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

  private readonly fallbackRanks: HeroRank[] = ['S', 'A', 'B', 'C'];
  protected readonly formRankOptions = computed<HeroRank[]>(() => {
    const derived = this.rankOptions().filter((option) => option !== 'ALL') as HeroRank[];
    return derived.length ? derived : this.fallbackRanks;
  });

  // ...existing code...
}
```

**值得注意的幾個觀察：**
1. `heroesState` 是唯讀 signal，因此元件只能讀、不能直接覆寫服務層資料，單向資料流更明確。
2. 篩選與搜尋共用 `activeRank`，無論清單或搜尋結果都會自動配合 rank 更新，不需要手動同步陣列。
3. `formRankOptions` 先嘗試從資料導出選項，若清單還沒載入就退回預設陣列，維持 UI 的可操作性。

**三、effect 把 signal 與表單行為黏在一起**
`effect` 在這支元件裡扮演「資料變動後該做的命令式動作」。我們在 constructor 中把表單同步、快取載入與 effect 連動串好。

```ts
// src/app/heroes/heroes.component.ts
constructor() {
  for (const control of [this.createForm.controls.name, this.editForm.controls.name]) {
    control.addAsyncValidators(this.heroNameTakenValidator());
    control.updateValueAndValidity({ onlySelf: true, emitEvent: false });
  }

  this.editForm.valueChanges
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(value => {
      this.editFormValue.set({
        name: value.name || '',
        rank: value.rank || ''
      });
    });

  this.heroService
    .loadAll()
    .pipe(
      finalize(() => this.heroesLoading.set(false)),
      takeUntilDestroyed(this.destroyRef)
    )
    .subscribe({
      next: () => {
        this.heroesError.set(null);
      },
      error: (err) => {
        this.heroesError.set(String(err ?? 'Unknown error'));
      },
    });

  effect(() => {
    const options = this.formRankOptions();
    const control = this.createForm.controls.rank;
    const current = control.value;
    if (!current) {
      return;
    }
    if (!options.includes(current)) {
      const fallback: HeroRank = options[0] ?? '';
      control.setValue(fallback, { emitEvent: false });
      control.markAsPristine();
      control.markAsUntouched();
    }
  });

  effect(() => {
    const selected = this.selectedHero();
    if (!selected) {
      this.lastEditHeroId = null;
      this.editForm.reset({ name: '', rank: '' }, { emitEvent: false });
      this.editForm.markAsPristine();
      this.editForm.markAsUntouched();
      this.editFormValue.set({ name: '', rank: '' });
      this.saveError.set(null);
      return;
    }

    const desiredValue = {
      name: selected.name,
      rank: (selected.rank as HeroRank) ?? '',
    };
    const isNewSelection = this.lastEditHeroId !== selected.id;
    const currentValue = this.editForm.getRawValue();

    if (!isNewSelection && this.editForm.dirty) {
      return;
    }

    const alreadySynced =
      currentValue.name === desiredValue.name && currentValue.rank === desiredValue.rank;
    if (!isNewSelection && alreadySynced) {
      return;
    }

    this.lastEditHeroId = selected.id;
    const formValue = {
      name: desiredValue.name,
      rank: desiredValue.rank,
    };
    this.editForm.reset(formValue, { emitEvent: false });
    this.editForm.markAsPristine();
    this.editForm.markAsUntouched();
    this.editFormValue.set(formValue);
    this.saveError.set(null);
  });
}
```

**兩個 effect 的工作：**
1. 第一個 effect 監看 `formRankOptions`，當資料變動導致現有選項失效時，自動回填合法值，維持表單穩定。
2. 第二個 effect 監看 `selectedHero`，針對不同情境（切換英雄、新選取、已修改）決定是否要 reset 表單，避免覆蓋使用者尚未送出的異動。

**四、RxJS 搜尋管線如何回寫 signal**
搜尋仍靠 RxJS operator 進行節流、錯誤處理，但最後把結果寫回 `rawSearchResults` signal，讓畫面透過 `computed` 取得一致的資料流。

```ts
// src/app/heroes/heroes.component.ts
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
```

重點是最後的 `.subscribe` 只負責把資料寫回 signal，畫面顯示與篩選都交給前面的 `computed` 處理。這讓我們在 effect、computed 與 RxJS 之間有清楚分工。

**五、Detail 頁面的 effect：路由參數驅動資料載入**
不只列表使用 effect，細節頁也靠 effect 監聽路由輸入，確保切換 `/heroes/:id` 時會自動觸發資料載入。

```ts
// projects/hero-journey/src/app/hero-detail/hero-detail.ts
constructor() {
  effect(() => {
    const curId = Number(this.id());
    this.loading.set(true);
    this.error.set(null);
    this.loadHero(curId);
  });
}
```

這段會在 `id` 改變時重新呼叫 `loadHero`，並搭配服務層的 `getById` 快取，達到「先用快取、再補請求」的體驗。

**今日小結：**
Signals 的三件套並非各自為政，而是構成一條清楚的資料流水線：服務層維護 `signal` 與 `computed`，元件用 `computed` 產生對應視圖，`effect` 則在狀態改變時處理命令式需求。
搭配 RxJS，我們就能寫出既宣告式又容易推理的互動流程。

**參考資料：**
- Signals：
  [https://dev.angular.tw/guide/signals](https://dev.angular.tw/guide/signals)