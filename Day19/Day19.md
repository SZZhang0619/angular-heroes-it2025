---
post_title: 'Day 19｜進階表單：Reactive Forms 與 FormBuilder'
author1: '阿蘇'
post_slug: 'day-19-typed-reactive-forms'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - ReactiveForms
  - FormBuilder
  - TypeScript
  - Validation
  - Signals
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '將 Heroes 頁面的新增與編輯流程重構為強型別 Reactive Forms，整合驗證與訊號狀態。'
post_date: '2025-10-03'
---

哈囉，各位邦友們！
前面一路從 Standalone、Signals、HTTP，到昨天的 `NgOptimizedImage`，可以說核心功能都穩定下來了。
今天就來開始挑戰 `Reactive Forms`，過去我們在 Heroes 頁面用 signal 暫存輸入值，對於簡單互動很方便。但一旦需求變複雜（例如欄位較多、動態驗證、串接後端錯誤），Template-driven Forms 就會顯得吃力，而 Reactive Forms 提供明確的資料結構，是企業專案最常見的做法，我目前接觸到的所有專案也都是如此處理。

## 今天要做什麼？
1. 引入 `ReactiveFormsModule` 與 `FormBuilder.nonNullable()`，建立表單並改寫新增流程。
3. 重構編輯流程與範本，讓 signals 與表單狀態同步。

## 前置需求
- 已完成 [Day18](https://ithelp.ithome.com.tw/articles/10392570) 的內容，專案可 `ng serve`。
- HeroService 已具備 `create()`、`update()`、`delete()` 等 CRUD 能力。
- 了解 [Day07](https://ithelp.ithome.com.tw/articles/10384956) 介紹的 Template-driven Forms 說明，方便比較差異。

**一、引入 Reactive Forms 並定義 HeroFormGroup**
HeroesComponent 要從 Template-driven 轉到 Reactive Forms，首先補上必要的import與型別定義，並用 `FormBuilder.nonNullable()` 建立FormControl。

```ts
// src/app/heroes/heroes.component.ts
import { FormBuilder, FormControl, FormGroup, ReactiveFormsModule, Validators } from '@angular/forms';
// ...existing code...
type HeroRank = '' | 'S' | 'A' | 'B' | 'C';
type HeroFormGroup = FormGroup<{
  name: FormControl<string>;
  rank: FormControl<HeroRank>;
}>;

@Component({
  selector: 'app-heroes',
  imports: [
    ReactiveFormsModule,
    RouterModule,
    LoadingSpinner,
    MessageBanner,
    HeroListItem,
  ],
  templateUrl: './heroes.component.html',
  styleUrl: './heroes.component.scss',
})
export class HeroesComponent {
  // ...existing code...
  private readonly fb = inject(FormBuilder);

  // ...existing code...

  protected readonly createForm: HeroFormGroup = this.fb.nonNullable.group({
    name: ['', [Validators.required, Validators.minLength(3)]],
    rank: this.fb.nonNullable.control<HeroRank>(''),
  });
  protected readonly editForm: HeroFormGroup = this.fb.nonNullable.group({
    name: ['', [Validators.required, Validators.minLength(3)]],
    rank: this.fb.nonNullable.control<HeroRank>(''),
  });
  protected readonly editFormValue = signal<{ name: string; rank: HeroRank }>({ name: '', rank: '' });
  // ...existing code...
}
```

**二、重構新增英雄流程**
新增功能改成讀取 `createForm` 的值，並在驗證失敗時直接標記欄位。

```ts
// src/app/heroes/heroes.component.ts
export class HeroesComponent {
  // ...existing code...
  protected addHero() {
    if (this.createForm.invalid) {
      this.createForm.markAllAsTouched();
      return;
    }
    const { name, rank } = this.createForm.getRawValue();
    const payload: Pick<Hero, 'name' | 'rank'> = {
      name: name.trim(),
      rank: rank || undefined,
    };
    // ...existing code...

    this.heroService
      .create(payload)
      .pipe(
        finalize(() => this.creating.set(false)),
        takeUntilDestroyed(this.destroyRef)
      )
      .subscribe({
        next: (created) => {
          this.feedback.set('新增英雄成功！');
          this.createForm.reset({ name: '', rank: '' });
          this.selectedId.set(created.id);
        },
        error: (err) => {
          // ...existing code...
        },
      });
  }
  // ...existing code...
}
```

**三、編輯區塊調整**
選取英雄時用 `effect` 重設 `editForm`，並以 `computed` 判斷是否有改動，避免送出空操作。

```ts
// projects/hero-journey/src/app/heroes/heroes.component.ts
export class HeroesComponent {
  // ...existing code...
  protected readonly dirtyCompared = computed(() => {
    const selected = this.selectedHero();
    if (!selected) {
      return false;
    }
    const value = this.editFormValue();
    const isDirty = (
      value.name.trim() !== selected.name ||
      value.rank !== ((selected.rank as HeroRank) ?? '')
    );

    return isDirty;
  });

  constructor() {
    this.editForm.valueChanges
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((value) => {
        this.editFormValue.set({
          name: value.name || '',
          rank: (value.rank as HeroRank) ?? '',
        });
      });

    // ...existing code...

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
        this.editForm.reset({ name: '', rank: '' });
        this.editForm.markAsPristine();
        this.editForm.markAsUntouched();
        this.editFormValue.set({ name: '', rank: '' });
        this.saveError.set(null);
        return;
      }
      const formValue = {
        name: selected.name,
        rank: (selected.rank as HeroRank) ?? '',
      };
      this.editForm.reset(formValue);
      this.editForm.markAsPristine();
      this.editForm.markAsUntouched();
      this.editFormValue.set(formValue);
      this.saveError.set(null);
    });
  }

  protected saveSelected() {
    const hero = this.selectedHero();
    if (!hero || this.editForm.invalid || !this.dirtyCompared()) {
      return;
    }
    const { name, rank } = this.editForm.getRawValue();
    // ...existing code...
  }

    // ...existing code...

    this.heroService
    .update(hero.id, { name: name.trim(), rank: rank || undefined })
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
```

**四、更新範本**

```html
<!-- src/app/heroes/heroes.component.html -->
<section class="create" id="create">
  <form [formGroup]="createForm" (ngSubmit)="addHero()">
    <label for="new-hero">Name：</label>
    <input
      id="new-hero"
      placeholder="enter new hero"
      formControlName="name"
      type="text" />
    @if (createForm.controls.name.touched && createForm.controls.name.invalid) {
      <small class="error">請輸入至少 3 個字</small>
    }

    <label for="new-hero-rank">Rank：</label>
    <select id="new-hero-rank" formControlName="rank">
      <option [ngValue]="''">未指定</option>
      @for (rank of formRankOptions(); track rank) {
        <option [ngValue]="rank">{{ rankLabel(rank) }}</option>
      }
    </select>

    <button type="submit" [disabled]="creating() || createForm.invalid">
      @if (creating()) { Saving... } @else { Add }
    </button>
  </form>

  @if (createError(); as err) {
    <app-message-banner type="error">Create failed: {{ err }}</app-message-banner>
  }
</section>

<!-- ...existing code... -->

@if (selectedHero(); as hero) {
  <aside class="panel" [formGroup]="editForm">
    <h3>Edit</h3>
    <p>
      #{{ hero.id }} - {{ hero.name }}
      @if (hero.rank) { <span class="rank">[{{ hero.rank }}]</span> }
    </p>

    <label for="hero-name">Name：</label>
    <input id="hero-name" type="text" formControlName="name" />
    @if (editForm.controls.name.touched && editForm.controls.name.invalid) {
      <small class="error">請輸入至少 3 個字</small>
    }

    <label for="hero-rank">Rank：</label>
    <select id="hero-rank" formControlName="rank">
      <option [ngValue]="''">未指定</option>
      @for (rank of formRankOptions(); track rank) {
        <option [ngValue]="rank">{{ rankLabel(rank) }}</option>
      }
    </select>

    <button
      type="button"
      (click)="saveSelected()"
      [disabled]="saving() || editForm.invalid || !dirtyCompared()">
      @if (saving()) { Saving... } @else { Save }
    </button>

    @if (saveError(); as err) {
      <app-message-banner type="error">Save failed: {{ err }}</app-message-banner>
    }
  </aside>
}
```

**說明：**
- `FormGroup` 自帶 `dirty`, `pristine`, `touched` 等旗標，後續要串續後端錯誤、highlight 欄位都更簡單。

**驗收清單：**
- 新增英雄表單改用 Reactive Forms，未輸入 3 個字時會顯示錯誤訊息且無法送出。
![https://ithelp.ithome.com.tw/upload/images/20251003/20159238CWMj18TbdB.png](https://ithelp.ithome.com.tw/upload/images/20251003/20159238CWMj18TbdB.png)
![https://ithelp.ithome.com.tw/upload/images/20251003/20159238br27LPT8nr.png](https://ithelp.ithome.com.tw/upload/images/20251003/20159238br27LPT8nr.png)
- 編輯英雄面板會依選取項目自動重設表單；無變更或表單無效時 Save 按鈕保持 disabled。
![https://ithelp.ithome.com.tw/upload/images/20251003/20159238FbCaTa5wzR.png](https://ithelp.ithome.com.tw/upload/images/20251003/20159238FbCaTa5wzR.png)
![https://ithelp.ithome.com.tw/upload/images/20251003/20159238ji0KQYzWaD.png](https://ithelp.ithome.com.tw/upload/images/20251003/20159238ji0KQYzWaD.png)
![https://ithelp.ithome.com.tw/upload/images/20251003/20159238dI2eJhKTyb.png](https://ithelp.ithome.com.tw/upload/images/20251003/20159238dI2eJhKTyb.png)

**今日小結：**
今天我們把新增與編輯流程變成透過 Reactive Forms 去實現。
明天會接著處理更複雜的 UI 整合，把這套表單真正運用在真實的使用情境內。

**參考資料：**
- 回應式表單: 
  [https://dev.angular.tw/guide/forms/reactive-forms](https://dev.angular.tw/guide/forms/reactive-forms)
- 型別化表單: 
  [https://dev.angular.tw/guide/forms/typed-forms](https://dev.angular.tw/guide/forms/typed-forms)
- 驗證器： 
  [https://dev.angular.tw/api/forms/Validators](https://dev.angular.tw/api/forms/Validators)