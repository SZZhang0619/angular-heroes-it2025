---
post_title: 'Day 18｜英雄頭像：NgOptimizedImage 圖片最佳化'
author1: '阿蘇'
post_slug: 'day-18-week2-refactor-ngoptimizedimage'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Refactor
  - Signals
  - NgOptimizedImage
  - RxJS
  - UI
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '全面檢視這週完成的 CRUD 與 UI 功能，整理 HeroService 與 Heroes 程式碼，並在細節頁導入 NgOptimizedImage，作為第二個學習檢查點與 Day17 挑戰的解答。'
post_date: '2025-10-02'
---

哈囉，各位邦友們！
昨天實作了不少東西，感覺一不小心塞太多了。
今天只要在細節頁加入英雄頭像，並使用 `NgOptimizedImage` 優化就好。

## 今天要做什麼？
1. 新增英雄頭像，並在 `HeroDetail` 導入 `NgOptimizedImage`，讓頭像載入更有效率。

## 前置需求
- 完成 [Day17](https://ithelp.ithome.com.tw/articles/10391966) 的內容，或至少閱讀過該篇的需求說明。
- 開發環境能 `npm run start`，確認 in-memory API 與 zoneless 設定皆正常。

**一、導入 `NgOptimizedImage`**
新版 Angular 內建的 `NgOptimizedImage` 能自動加上 `loading="lazy"`、預先載入策略與寬高屬性。
我們把它放進 `HeroDetail` 並利用 DiceBear 的圖像 API：
   ```
   https://api.dicebear.com/7.x/bottts-neutral/png?seed=<HeroName>&size=320&background=%23eef3ff
   ```
這樣就不需要額外放圖檔，也能依英雄名稱產生一致的頭像。

```ts
// projects/hero-journey/src/app/hero-detail/hero-detail.ts
import { Component, DestroyRef, computed, effect, inject, input, signal } from '@angular/core';
// ...existing code...
import { RouterModule } from '@angular/router';
import { NgOptimizedImage } from '@angular/common';

@Component({
  // ...existing code...
  imports: [RouterModule, NgOptimizedImage],
  // ...existing code...
})
export class HeroDetail {
  // ...existing code...
  readonly avatarUrl = computed(() => {
    const hero = this.hero();
    if (!hero) {
      return null;
    }
    const seed = encodeURIComponent(hero.name);
    return `https://api.dicebear.com/7.x/bottts-neutral/png?seed=${seed}&size=320&background=%23eef3ff`;
  });

  // ...existing code...
}
```

```html
<!-- projects/hero-journey/src/app/hero-detail/hero-detail.html -->
@if (loading()) {
  <!-- ...existing code... -->
} @else if (error(); as e) {
  <!-- ...existing code... -->
} @else if (hero(); as h) {
  <section class="detail">
    @if (avatarUrl(); as src) {
      <figure class="detail__media">
        <img
          [ngSrc]="src"
          width="320"
          height="320"
          [attr.fetchpriority]="h.rank === 'S' ? 'high' : null"
          alt="{{ h.name }} portrait" />
      </figure>
    }
    <!-- ...existing code... -->
  </section>
} @else {
  <!-- ...existing code... -->
}
```

```scss
// projects/hero-journey/src/app/hero-detail/hero-detail.scss
// ...existing code...

.detail__media {
  margin: 0 0 16px;
  display: flex;
  justify-content: center;
}

.detail__media img {
  border-radius: 16px;
  box-shadow: 0 12px 24px rgba(27, 54, 93, 0.18);
}
```

## 驗收清單
- `HeroDetail` 顯示的頭像具有 `ngSrc`、`loading="lazy"`，另外因為 Rank S 英雄有設定`[attr.fetchpriority]=high`的圖片優先載入。
![https://ithelp.ithome.com.tw/upload/images/20251002/20159238A9cbMACiaE.png](https://ithelp.ithome.com.tw/upload/images/20251002/20159238A9cbMACiaE.png)
![https://ithelp.ithome.com.tw/upload/images/20251002/20159238UPONR8J0TX.png](https://ithelp.ithome.com.tw/upload/images/20251002/20159238UPONR8J0TX.png)

## 今日小結
今天在英雄細節頁使用 DiceBear 的 API 產生圖像，並透過 `NgOptimizedImage` 這種現代化特性讓圖片最佳化。
下一階段會開始處理更進階的體驗與部署，今天稍微放鬆一下！

## 參考資料
-  NgOptimizedImage: 
  [https://dev.angular.tw/guide/image-optimization](https://dev.angular.tw/guide/image-optimization)