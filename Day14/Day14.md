---
post_title: 'Day 14｜CRUD 實作：刪除 (DELETE) 與錯誤處理'
author1: '阿蘇'
post_slug: 'day-14-crud-delete-error-handling'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - HttpClient
  - DELETE
  - catchError
  - RxJS
  - Signals
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '擴充 delete$ 與 catchError，提供刪除成功與錯誤提示，補齊 CRUD 流程。'
post_date: '2025-09-28'
---

哈囉，各位邦友們！
昨天我們完成了 HeroService 的新增、更新流程，也在畫面上實現了互動。
今天要補上 CRUD 最後一塊：刪除 (DELETE)，同時透過 `catchError` 建立錯誤處理流程。

## 今天要做什麼？
1. 在 `HeroService` 新增 `delete$()`。
2. 為 Heroes 畫面加入刪除按鈕與確認流程。
3. 使用 `catchError` 與 `EMPTY` 妥善處理錯誤，避免程式整個崩潰。
4. 加上成功/失敗訊息。

## 前置需求
- 已完成 [Day13](https://ithelp.ithome.com.tw/articles/10389320) 的 create$/update$。
- 專案 `ng serve` 可正常啟動。

**一、HeroService：新增 delete$() 並清理 Map 快取**
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

  delete$(id: number) {
    return this.http.delete<void>(`${this.baseUrl}/${id}`).pipe(
      tap(() => {
        this.cache.delete(id);
      })
    );
  }
}
```

**二、HeroesComponent：整合刪除流程與 catchError**
```typescript
// src/app/heroes/heroes.component.ts
import { Component, DestroyRef, effect, inject, signal } from '@angular/core';
import { HeroService, Hero } from '../hero.service';
import { FormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { RouterModule } from '@angular/router';
import { HeroBadge } from '../hero-badge/hero-badge';
import { EMPTY, catchError, finalize } from 'rxjs';

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

  protected readonly deletingId = signal<number | null>(null);
  protected readonly deleteError = signal<string | null>(null);
  protected readonly feedback = signal<string | null>(null);

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
    this.feedback.set(null);

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
    this.feedback.set(null);

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

  removeHero(hero: Hero) {
    const confirmed = confirm(`確定要刪除英雄「${hero.name}」嗎？`);
    if (!confirmed) {
      return;
    }

    this.deletingId.set(hero.id);
    this.deleteError.set(null);
    this.feedback.set(null);

    this.heroService
      .delete$(hero.id)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        catchError((err) => {
          this.deleteError.set(String(err ?? 'Unknown error'));
          return EMPTY;
        }),
        finalize(() => {
          this.deletingId.set(null);
        })
      )
      .subscribe(() => {
        this.heroes.update((list) => list.filter((h) => h.id !== hero.id));
        if (this.selectedHero()?.id === hero.id) {
          this.selectedHero.set(null);
          this.editName.set('');
        }
        this.feedback.set(`已刪除英雄「${hero.name}」。`);
      });
  }
}
```

**三、Heroes 模板：刪除按鈕與狀態提示**
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

  @if (feedback(); as msg) {
    <p class="feedback" aria-live="polite">{{ msg }}</p>
  }

  @if (deleteError(); as err) {
    <p class="error" aria-live="assertive">Delete failed: {{ err }}</p>
  }

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
            <button
              type="button"
              class="danger"
              (click)="removeHero(h); $event.stopPropagation()"
              [disabled]="deletingId() === h.id">
              @if (deletingId() === h.id) { Deleting... } @else { Delete }
            </button>
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

**四、樣式：提示訊息與刪除按鈕**
```scss
/* src/app/heroes/heroes.component.scss */
.error {
  color: #c33;
}

.feedback {
  margin: 12px 0;
  padding: 8px 12px;
  background: #e6f6ec;
  color: #0a5e2a;
  border-radius: 6px;
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

button.danger {
  margin-left: 8px;
  background: #f7eceb;
  color: #a32020;
  border-color: #f1c4c1;
}

button.danger[disabled] {
  opacity: 0.7;
  cursor: progress;
}
```

延續昨天的設定，InMemoryData 使用提供資料，刪除後的結果只影響本次執行中的假後端。

**驗收清單：**
- 按下 Delete 後，對話框確認後才會真正刪除，列表與編輯區同步更新。
![https://ithelp.ithome.com.tw/upload/images/20250928/201592381YRGYKvD2H.png](https://ithelp.ithome.com.tw/upload/images/20250928/201592381YRGYKvD2H.png)![https://ithelp.ithome.com.tw/upload/images/20250928/20159238XmKerN3ds3.png](https://ithelp.ithome.com.tw/upload/images/20250928/20159238XmKerN3ds3.png)
- 刪除成功時會顯示 `已刪除英雄「xxx」`，刪除失敗時顯示 `Delete failed: ...`。
![https://ithelp.ithome.com.tw/upload/images/20250928/20159238T7nO65jyWp.png](https://ithelp.ithome.com.tw/upload/images/20250928/20159238T7nO65jyWp.png)

**常見錯誤與排查：**
- 刪除後列表未更新：檢查 `heroes.update` 是否回傳新的陣列，且別忘了 `return list.filter(...)`。
- 錯誤訊息沒有出現：確認 `catchError` 有回傳 `EMPTY`，讓 Observable 停在錯誤分支。

**今日小結：**
我們將 HeroService 補齊 `delete$()`，完成了 CRUD 的最後一步！

**參考資料：**
- HttpClient DELETE：
  [https://dev.angular.tw/api/common/http/HttpClient#delete](https://dev.angular.tw/api/common/http/HttpClient#delete)
- RxJS catchError：
  [https://rxjs.dev/api/operators/catchError](https://rxjs.dev/api/operators/catchError)
- EMPTY 常數：
  [https://rxjs.dev/api/index/const/EMPTY](https://rxjs.dev/api/index/const/EMPTY)
- finalize 運算子：
  [https://rxjs.dev/api/operators/finalize](https://rxjs.dev/api/operators/finalize)
