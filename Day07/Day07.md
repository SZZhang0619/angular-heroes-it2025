---
post_title: 'Day 7｜表單技巧：FormsModule 與 ngModel'
author1: '阿蘇'
post_slug: 'day07-weekly-review-ngmodel'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Angular
  - FormsModule
  - ngModel
  - TemplateDrivenForms
  - Recap
ai_note: 'Assisted by AI and reviewed by the author.'
summary: '在元件內以 ngModel 編輯英雄名稱，。'
post_date: '2025-09-21'
---

哈囉，各位邦友們！
昨天完成用事件綁定與 @for/@if 搭配 scss 來實現選中狀態時的互動。
今天來試著挑戰透過input編輯英雄名稱，並且在畫面上即時更新。


## 今天要做什麼？
1. 在 Standalone 元件中加入 FormsModule。
2. 在 Selected 區塊加入輸入框，使用 ngModel 即時編輯英雄名稱。
3. 用 signal 的 update() 來更新資料。

## 前置需求
- 已完成 [Day05](https://ithelp.ithome.com.tw/articles/10383468) 的 heroes 清單與 [Day06](https://ithelp.ithome.com.tw/articles/10384071) 的 selectedHero 選取邏輯。
- 專案可 ng serve，選中英雄資訊能正常顯示.

**一、在 Standalone 中加入 FormsModule**
```typescript
// src/app/app.ts
// ...existing code...
import { FormsModule } from '@angular/forms';
// ...existing code...

@Component({
  // ...existing code...
  imports: [HeroBadge, FormsModule],
  // ...existing code...
})
export class App {
  // ...existing code...

  protected readonly heroes = signal<Hero[]>([
    { id: 11, name: 'Dr Nice', rank: 'B' },
    { id: 12, name: 'Narco', rank: 'A' },
    { id: 13, name: 'Bombasto' },
    { id: 14, name: 'Celeritas', rank: 'S' },
  ]);

  protected readonly selectedHero = signal<Hero | null>(null);

  // 新增：同步更新 selectedHero 與 heroes 清單
  updateName(name: string) {
    const selected = this.selectedHero();
    if (!selected) {
      return;
    }

    // 1. 建立更新後的英雄物件
    const updatedHero = { ...selected, name };

    // 2. 更新英雄列表 (heroes signal)
    this.heroes.update(list =>
      list.map(hero => (hero.id === updatedHero.id ? updatedHero : hero))
    );

    // 3. 更新當前選取的英雄 (selectedHero signal)
    this.selectedHero.set(updatedHero);
  }
}
```

**二、用 ngModel 編輯選中英雄名稱並驗證**
```html
<!-- src/app/app.html -->
<!-- ...existing code... -->
<section>
  <!-- ...existing code... -->
  @if (selectedHero(); as s) {
    <aside class="panel">
      <h3>Selected</h3>
      <p>
        #{{ s.id }} - {{ s.name }}
        @if (s.rank) { <span class="rank">[{{ s.rank }}]</span> }
      </p>

      <label for="hero-name">Edit name：</label>
      <input
        id="hero-name"
        name="hero-name"
        type="text"
        placeholder="enter new name"
        required
        minlength="3"
        #nameCtrl="ngModel"
        [ngModel]="s.name"
        (ngModelChange)="updateName($event)"
        [attr.aria-invalid]="nameCtrl.invalid && nameCtrl.touched"
        aria-describedby="hero-name-errors" />

      @if (nameCtrl.invalid && nameCtrl.touched) {
        <ul id="hero-name-errors" class="errors">
          @if (nameCtrl.hasError('required')) {
            <li>請輸入名稱</li>
          }
          @if (nameCtrl.hasError('minlength')) {
            <li>至少 3 個字</li>
          }
        </ul>
      }
    </aside>
  }
</section>
<!-- ...existing code... -->
```

**三、新增樣式**

```scss
// src/app/app.scss
/* ...existing code... */
.errors {
  color: #c33;
}
```

**驗收清單：**
- 點擊清單中的英雄後，Selected 區塊出現英雄資訊與輸入框。
![https://ithelp.ithome.com.tw/upload/images/20250921/20159238mOtLUqC99f.png](https://ithelp.ithome.com.tw/upload/images/20250921/20159238mOtLUqC99f.png)
- 在輸入框輸入文字時，會即時更新 Selected 區塊與清單上的英雄名稱。
![https://ithelp.ithome.com.tw/upload/images/20250921/20159238xwdKloo6FF.png](https://ithelp.ithome.com.tw/upload/images/20250921/20159238xwdKloo6FF.png)
- 再次點擊同一英雄可取消選中，Selected 區塊隱藏。
![https://ithelp.ithome.com.tw/upload/images/20250921/20159238DTbZYZU34d.png](https://ithelp.ithome.com.tw/upload/images/20250921/20159238DTbZYZU34d.png)
- 清單為空時顯示 @empty 的 No heroes。
![https://ithelp.ithome.com.tw/upload/images/20250921/201592383e3nRO5tga.png](https://ithelp.ithome.com.tw/upload/images/20250921/201592383e3nRO5tga.png)

**常見錯誤與排查：**
- ngModel 無效：確認 FormsModule 已加入當前元件的 imports。
- 畫面未更新：請使用 selectedHero.update()，避免只改 s.name。
- 表單出現警告：確保 input 含有 name 屬性。
- 綁定錯誤：檢查 (ngModelChange) 與方法名稱是否一致。

**今日小結：**
今天在 Selected 區塊中加入 FormsModule 與 ngModel，使選取中的英雄可以編輯，並讓清單與選取畫面實現同步更新。
明天會將建立 HeroService，透過 inject() 依賴注入，學習Angular中服務的使用方法，讓專案更符合實際的架構設計。

**參考資料：**
- Forms（Angular 中的表單）:
  [https://dev.angular.tw/guide/forms](https://dev.angular.tw/guide/forms)
- FormsModule（API）:
  [https://dev.angular.tw/api/forms/FormsModule](https://dev.angular.tw/api/forms/FormsModule)
- Signals（訊號）:
  [https://dev.angular.tw/guide/signals](https://dev.angular.tw/guide/signals)