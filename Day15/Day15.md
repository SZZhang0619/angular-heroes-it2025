---
post_title: 'Day 15｜UI/UX 優化：Loading 與 Empty 狀態'
author1: '阿蘇'
post_slug: 'day-15-ui-ux-loading-empty-states'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - UI
  - UX
  - Spinner
  - EmptyState
  - HttpClient
  - Signals
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '建立共用 LoadingSpinner 元件，調整 Heroes 模板與樣式，提供載入與空清單的視覺回饋，確保 CRUD 體驗更順手。'
post_date: '2025-09-29'
---

哈囉，各位邦友們！
昨天我們完成了 HeroService 的刪除流程，並在畫面上加上錯誤提示，避免失敗操作沒人知道。
但通常實戰時還會需要考慮到「等待」與「沒有資料」時的畫面處理。
所以今天來接著把 Loading 與 Empty 狀態補齊，讓整個 CRUD 像真正的產品。

## 今天要做什麼？
1. 建立 `LoadingSpinnerComponent`，提供重複使用的載入指示器。
2. 在 Heroes 畫面匯入 Spinner，統一取代暫時顯示的純文字。
3. 強化 `@if` / `@empty` 區塊，讓空清單時有明確指引與行動。

## 前置需求
- 已完成 [Day14](https://ithelp.ithome.com.tw/articles/10390010)（CRUD 刪除與錯誤處理），HeroService 擁有 `delete$()` 與錯誤回饋。
- 專案可以 `ng serve`，並正常顯示畫面。

**一、LoadingSpinnerComponent：打造共用載入指示器**
之前都使用一行 `Loading heroes...` 來表示載入中，今天改成可重複利用的元件，之後任何頁面要顯示載入狀態都能直接套用。

```sh
ng g component ui/loading-spinner
```

```typescript
// src/app/ui/loading-spinner/loading-spinner.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-loading-spinner',
  imports: [],
  templateUrl: './loading-spinner.html',
  styleUrl: './loading-spinner.scss',
})
export class LoadingSpinner {
  @Input() label = 'Loading...';
}
```

```html
<!-- src/app/ui/loading-spinner/loading-spinner.html -->
<div class="spinner" role="status" aria-live="polite">
  <span class="spinner__dot" aria-hidden="true"></span>
  <span class="spinner__dot" aria-hidden="true"></span>
  <span class="spinner__dot" aria-hidden="true"></span>
  <span class="spinner__text">{{ label }}</span>
</div>
```

```scss
/* src/app/ui/loading-spinner/loading-spinner.scss */
.spinner {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  color: #4a6078;
  font-weight: 600;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.spinner__dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: currentColor;
  animation: spinner-bounce 0.8s ease-in-out infinite;
}

.spinner__dot:nth-child(2) {
  animation-delay: 0.12s;
}

.spinner__dot:nth-child(3) {
  animation-delay: 0.24s;
}

.spinner__text {
  font-size: 0.85rem;
  opacity: 0.8;
}

@keyframes spinner-bounce {
  0%,
  80%,
  100% {
    transform: scale(0.6);
    opacity: 0.4;
  }
  40% {
    transform: scale(1);
    opacity: 1;
  }
}

@media (prefers-reduced-motion: reduce) {
  .spinner__dot {
    animation: none;
  }
}
```

**二、HeroesComponent：匯入 Spinner**
匯入元件後，把 HeroesComponent 的載入狀態改用 `<app-loading-spinner>` 呈現即可，其餘程式碼不更動。

```typescript
// src/app/heroes/heroes.component.ts
import { LoadingSpinner } from '../ui/loading-spinner/loading-spinner';
@Component({
  selector: 'app-heroes',
  imports: [HeroBadge, FormsModule, RouterModule, LoadingSpinnerComponent],
  templateUrl: './heroes.component.html',
  styleUrl: './heroes.component.scss',
})
export class HeroesComponent {
  // ...其餘程式碼維持不變
}
```

**三、Heroes 模板：用 `@if` / `@empty` 呈現狀態**
更新模板，把原本的純文字 Loading 改為元件，並 @empty 時顯示一個指引提示使用者新增英雄。

```html
<!-- src/app/heroes/heroes.component.html -->
@if (heroesLoading()) {
  <app-loading-spinner label="Loading heroes..."></app-loading-spinner>
} @else if (heroesError(); as e) {
  <p class="error">Load failed: {{ e }}</p>
} @else {
  <section class="create" id="create">
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
        <li class="empty-state" aria-live="polite">
          <h3>目前還沒有英雄</h3>
          <p>點擊上方的 Add 建立第一位夥伴，或在 in-memory API 填入假資料。</p>
        </li>
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

**四、樣式：幫 Loading 與 Empty 狀態加上樣式**
載入時需要少量動態樣式，而沒有資料時則可以顯眼地提醒使用者該做什麼。
這段加在 `heroes.component.scss` 最下面就可以了。

```scss
/* src/app/heroes/heroes.component.scss */
.empty-state {
  margin: 24px 0;
  padding: 32px 24px;
  display: grid;
  gap: 12px;
  justify-items: center;
  text-align: center;
  border: 1px dashed #a5b4d0;
  border-radius: 12px;
  background: #f4f7ff;
  color: #44516c;
}

.empty-state h3 {
  margin: 0;
  font-size: 1.05rem;
  font-weight: 600;
}

.empty-state p {
  margin: 0;
  max-width: 320px;
  line-height: 1.5;
  color: #5b6b8c;
}

.empty-state__cta {
  display: inline-block;
  padding: 8px 14px;
  border-radius: 999px;
  border: 1px solid #5c7aff;
  background: white;
  color: #3b4ecb;
  font-weight: 600;
  text-decoration: none;
  transition: background 0.2s ease, color 0.2s ease;
}

.empty-state__cta:hover {
  background: #5c7aff;
  color: white;
}

.empty-state__cta:focus-visible {
  outline: 3px solid #feeaa3;
  outline-offset: 3px;
}
```

**驗收清單：**
- 重新整理頁面或在程式碼中加上dalay()模擬慢速網路，可看到 Spinner 的 `Loading heroes...` 顯示。
![https://ithelp.ithome.com.tw/upload/images/20250929/20159238OuMNpnD6BB.png](https://ithelp.ithome.com.tw/upload/images/20250929/20159238OuMNpnD6BB.png)
- 刪除所有英雄（或把 in-memory 資料改成空陣列）時，列表顯示無資料提示。
![https://ithelp.ithome.com.tw/upload/images/20250929/20159238Iu2Qp2tzGr.png](https://ithelp.ithome.com.tw/upload/images/20250929/20159238Iu2Qp2tzGr.png)
- 新增一位英雄後，無資料提示消失、列表與選取區恢復原狀。
![https://ithelp.ithome.com.tw/upload/images/20250929/20159238ECTtbqad3K.png](https://ithelp.ithome.com.tw/upload/images/20250929/20159238ECTtbqad3K.png)

**常見錯誤與排查：**
- 沒看到 Spinner：確認 `LoadingSpinnerComponent` 已加入 `imports` 並且 `heroesLoading()` 初始值為 `true`。
- 無資料提示沒有顯示：檢查 `heroes()` 是否回傳空陣列，使用 `@empty` 只能在 `@for` 內運作。

**今日小結：**
我們把專案補上 Loading 與 Empty UI，讓使用者在等待或資料為空時的使用者體驗更好。
這些細節不會改變程式邏輯，卻能提升操作體驗，也是正式專案不可或缺的一環。