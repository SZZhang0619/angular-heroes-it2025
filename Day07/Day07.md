---
post_title: 'Day 7｜表單技巧：FormsModule 與 ngModel'
author1: 'Frank Zhang'
post_slug: 'day07-weekly-review-ngmodel'
microsoft_alias: 'frankz'
featured_image: 'https://via.placeholder.com/1200x630?text=Angular+Day+07'
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
### 今天要做什麼？
1. 在 Standalone 元件中加入 FormsModule。
2. 在 Selected 區塊加入輸入框，使用 ngModel 即時編輯英雄名稱。
3. 用 signal 的 update() 來更新資料。

#### 前置需求
* 已完成 Day05 的 heroes 清單與 Day06 的 selectedHero 選取邏輯。
* 專案可 ng serve，選中英雄資訊能正常顯示.

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
* 點擊清單中的英雄後，Selected 區塊出現英雄資訊與輸入框。
* 在輸入框輸入文字時，會即時更新 Selected 區塊與清單上的英雄名稱。
* 再次點擊同一英雄可取消選中，Selected 區塊隱藏。
* 清單為空時顯示 @empty 的 No heroes。

**常見錯誤與排查：**
* ngModel 無效：確認 FormsModule 已加入當前元件的 imports。
* 畫面未更新：請使用 selectedHero.update()，避免只改 s.name。
* 表單出現警告：確保 input 含有 name 屬性。
* 綁定錯誤：檢查 (ngModelChange) 與方法名稱是否一致。

**今日小結：**
今天在 Selected 區塊中加入 FormsModule 與 ngModel，使選取中的英雄可以編輯，並讓清單與選取畫面實現同步更新。
明天會將建立 HeroService，透過 inject() 依賴注入，學習Angular中服務的使用方法，讓專案更符合實際的架構設計。

**參考資料：**
* Template-driven forms（ngModel）  
  [https://dev.angular.tw/guide/forms](https://dev.angular.tw/guide/forms)
* FormsModule（API）  
  [https://dev.angular.tw/api/forms/FormsModule](https://dev.angular.tw/api/forms/FormsModule)
* Signals（概念與 API）  
  [https://dev.angular.tw/guide/signals](https://dev.angular.tw/guide/signals)