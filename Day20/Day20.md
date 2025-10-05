---
post_title: 'Day 20｜進階表單：Validators 與自訂驗證'
author1: '阿蘇'
post_slug: 'day-20-reactive-forms-validation'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - ReactiveForms
  - Validators
  - FormBuilder
  - CustomValidator
  - UI
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '在 Heroes Reactive Forms 中建立錯誤訊息機制、加入同步與非同步驗證，確保英雄名的品質與體驗。'
post_date: '2025-10-04'
---

哈囉，各位邦友們！
昨天我們把 Heroes 頁面的新增與編輯流程全面重構為 Reactive Forms。
今天則要把自訂驗證訊息放進來，透過 Angular 內建的 `Validators` 與自訂檢查，在驗證輸入時給使用者更多指引。

## 今天要做什麼？
1. 建立共用的錯誤訊息對照與 helper，整齊地在範本上顯示驗證結果。
2. 擴充現有 Reactive Forms，加入 `Validators.maxLength`、`Validators.pattern` 等同步檢查。
3. 新增自訂驗證：阻擋保留字，並以非同步方式比對既有英雄名，避免重複。

## 前置需求
- 完成 [Day19](https://ithelp.ithome.com.tw/articles/10393022) 的重構，HeroesComponent 已使用 Reactive Forms。
- 專案可 `mg serve`，並能呼叫 in-memory API。
- 具備基本的 RxJS 與 Angular 表單觀念，知道 `Validators` 與 FormControl 狀態 (`touched`, `pending` 等)。

**一、集中管理錯誤訊息與保留字**
我們先補上需要的匯入、常數與 helper，避免在模板裡到處硬寫字串。

```ts
// projects/hero-journey/src/app/heroes/heroes.component.ts
import {
  AbstractControl,
  AsyncValidatorFn,
  FormBuilder,
  FormControl,
  FormGroup,
  ReactiveFormsModule,
  ValidationErrors,
  ValidatorFn,
  Validators,
} from '@angular/forms';
// ...existing code...
import {
  EMPTY,
  Subject,
  catchError,
  debounceTime,
  distinctUntilChanged,
  filter,
  finalize,
  map,
  of,
  switchMap,
  tap,
  timer,
} from 'rxjs';

const HERO_NAME_PATTERN = /^[A-Za-z][A-Za-z0-9\s'-]{2,23}$/;
const RESERVED_HERO_NAMES = ['admin', 'root', 'unknown'] as const;

function heroNameReservedValidator(names: readonly string[]): ValidatorFn {
  const normalized = names.map((name) => name.trim().toLowerCase());
  return (control) => {
    const value = (control.value ?? '').trim().toLowerCase();
    if (!value) {
      return null;
    }
    return normalized.includes(value) ? { reserved: true } : null;
  };
}
```

在類別裡加入錯誤訊息對照與 helper，讓範本單純呼叫 `controlError()`。

```ts
export class HeroesComponent {
  // ...existing code...
  private lastEditHeroId: number | null = null;
  private readonly validationMessages: Record<string, string> = {
    required: '名稱必填，不能空白。',
    minlength: '請至少輸入 3 個字。',
    maxlength: '名稱不可超過 24 個字。',
    pattern: '僅允許英文、數字、空白與 -′ 字元。',
    reserved: '這個名稱被列為保留字，請換一個。',
    duplicated: '已有英雄使用這個名稱。',
  };

  protected controlError(control: AbstractControl | null): string | null {
    if (!control || control.disabled || !control.invalid || !control.touched) {
      return null;
    }
    const errors = control.errors as ValidationErrors | null;
    if (!errors) {
      return null;
    }
    for (const key of Object.keys(errors)) {
      const message = this.validationMessages[key];
      if (message) {
        return message;
      }
    }
    return '輸入格式不正確，請再試一次。';
  }

  // ...existing code...
}
```

**二、擴充同步驗證規則**
把新增與編輯表單的 `name` 控制項改成物件寫法，加入更多同步驗證並設定 `updateOn: 'blur'`，避免每個字都觸發非同步檢查。

```ts
export class HeroesComponent {
  // ...existing code...
  protected readonly createForm: HeroFormGroup = this.fb.nonNullable.group({
    name: this.fb.nonNullable.control('', {
      validators: [
        Validators.required,
        Validators.minLength(3),
        Validators.maxLength(24),
        Validators.pattern(HERO_NAME_PATTERN),
        heroNameReservedValidator(RESERVED_HERO_NAMES),
      ],
      updateOn: 'blur',
    }),
    rank: this.fb.nonNullable.control<HeroRank>(''),
  });

  protected readonly editForm: HeroFormGroup = this.fb.nonNullable.group({
    name: this.fb.nonNullable.control('', {
      validators: [
        Validators.required,
        Validators.minLength(3),
        Validators.maxLength(24),
        Validators.pattern(HERO_NAME_PATTERN),
        heroNameReservedValidator(RESERVED_HERO_NAMES),
      ],
      updateOn: 'blur',
    }),
    rank: this.fb.nonNullable.control<HeroRank>(''),
  });
  // ...existing code...
}
```

**三、加入自訂重複名稱驗證**
接著建立一個 `AsyncValidatorFn`，先比對目前記憶體內的英雄，再透過 `HeroService.search$` 與伺服器確認是否重複。
為了降低負擔，我們用 `timer(300)` 做簡單 debounce。

```ts
export class HeroesComponent {
  // ...existing code...
  private heroNameTakenValidator(): AsyncValidatorFn {
    return (control) => {
      const raw = (control.value ?? '').trim();
      if (!raw) {
        return of(null);
      }

      const value = raw.toLowerCase();
      const currentId = this.selectedId();
      const selected = this.selectedHero();
      if (
        selected &&
        selected.id === currentId &&
        selected.name.trim().toLowerCase() === value
      ) {
        return of(null);
      }
      const existsLocally = this.heroes().some(
        (hero) => hero.name.toLowerCase() === value && hero.id !== currentId
      );
      if (existsLocally) {
        return of({ duplicated: true });
      }

      return timer(300).pipe(
        switchMap(() => this.heroService.search$(raw)),
        map((heroes) => {
          const taken = heroes.some(
            (hero) => hero.name.toLowerCase() === value && hero.id !== currentId
          );
          return taken ? { duplicated: true } : null;
        }),
        catchError(() => of(null))
      );
    };
  }

  constructor() {
    // ...existing constructor code...

    for (const control of [this.createForm.controls.name, this.editForm.controls.name]) {
      control.addAsyncValidators(this.heroNameTakenValidator());
      control.updateValueAndValidity({ onlySelf: true, emitEvent: false });
    }

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
  // ...existing code...
}
```

**四、在模板顯示錯誤與 pending 狀態**
最後更新範本，改為呼叫 `controlError()`，並在非同步驗證進行時顯示提示。

```html
<!-- projects/hero-journey/src/app/heroes/heroes.component.html -->
<input
  id="new-hero"
  placeholder="enter new hero"
  formControlName="name"
  type="text" />
@if (controlError(createForm.controls.name); as err) {
  <small class="error">{{ err }}</small>
} @else if (createForm.controls.name.pending) {
  <small class="hint">檢查名稱中...</small>
}

<!-- ...existing code... -->

<button type="submit" [disabled]="creating() || createForm.invalid || createForm.pending">
  @if (creating()) { Saving... } @else { Add }
</button>
```

同樣套用在編輯面板：

```html
<input id="hero-name" type="text" formControlName="name" />
@if (controlError(editForm.controls.name); as err) {
  <small class="error">{{ err }}</small>
} @else if (editForm.controls.name.pending) {
  <small class="hint">檢查名稱中...</small>
}

<!-- ...existing code... -->

<button
  type="button"
  (click)="saveSelected()"
  [disabled]="saving() || editForm.invalid || editForm.pending || !dirtyCompared()">
  @if (saving()) { Saving... } @else { Save }
</button>
```

**五、調整樣式**
幫 `hint` 與  `error` 新增樣式，確保顏色區隔開來。

```scss
.error {
  color: #d92d20;
  font-size: 0.85rem;
}

.hint {
  color: #475467;
  font-size: 0.85rem;
}
```

**驗收清單：**
- 在新增英雄欄位輸入 `ad` 並離開輸入框會看到最少字數提示；輸入 `admin` 則顯示保留字訊息。
![https://ithelp.ithome.com.tw/upload/images/20251004/20159238rXuJOyoiOu.png](https://ithelp.ithome.com.tw/upload/images/20251004/20159238rXuJOyoiOu.png)
- 若輸入已存在的英雄名（例如列表中的某位），離開輸入框會觸發「英雄名已存在」錯誤，提交按鈕保持 disabled。
![https://ithelp.ithome.com.tw/upload/images/20251004/201592386l1VFURVdJ.png](https://ithelp.ithome.com.tw/upload/images/20251004/201592386l1VFURVdJ.png)
- 非同步驗證進行時，表單會顯示「檢查名稱中…」，並在完成後自動更新狀態。
![https://ithelp.ithome.com.tw/upload/images/20251004/20159238D8KFj7dvZ4.png](https://ithelp.ithome.com.tw/upload/images/20251004/20159238D8KFj7dvZ4.png)
- 編輯面板保留原本英雄名稱時不會報錯，只有改成其他人的名稱才會阻擋儲存。
![https://ithelp.ithome.com.tw/upload/images/20251004/20159238Ya04fhEIp8.png](https://ithelp.ithome.com.tw/upload/images/20251004/20159238Ya04fhEIp8.png)

**今日小結：**
我們把 Reactive Forms 用內建 Validators、寫出自己的檢查邏輯，在面對多欄位、多規則的表單時，就能用一致的方式管理品質。

**參考資料：**
- Reactive Forms 驗證 & 自訂 Validator：
  [https://dev.angular.tw/guide/forms/form-validation](https://dev.angular.tw/guide/forms/form-validation)