---
post_title: 'Day 25｜延遲載入：@defer'
author1: '阿蘇'
post_slug: 'day-25-defer-lazy-load'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Defer
  - Performance
  - LazyLoad
  - ControlFlow
  - UX
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '聚焦 @defer 的觸發條件與區塊使用（placeholder/loading/error/prefetch），以實作範例展示如何延遲載入非關鍵視圖、改善 LCP 與互動體驗。'
post_date: '2025-10-09'
---

哈囉，各位邦友們！
今天的主角是 Angular v17 引入的「可延遲視圖」：`@defer`。
它能讓你把「非關鍵、可晚點出現」的區塊推遲載入，改善初始載入與互動速度。
我們會用實戰範例示範「何時載、怎麼載、出錯怎麼辦」。

## 今天要做什麼？
1. 認識 `@defer` 與可用的觸發條件。
2. 實作 `on viewport` 與 `on interaction` 的典型情境。
3. 加上 `@placeholder`、`@loading`、`@error` 與 `prefetch`。
4. 蒐集最佳實務與常見地雷，建立你的延遲載入口袋名單。

## 前置需求
- 熟悉 Standalone 與控制流。
- 專案可 `ng serve`，具備 `LoadingSpinner`、`MessageBanner` 等共用元件（[Day15](https://ithelp.ithome.com.tw/articles/10390073)、[Day17](https://ithelp.ithome.com.tw/articles/10391966)）。

---

## 一、什麼是 `@defer`？
一句話：把不需要「立刻」出現的畫面延後載入。
`@defer` 區塊會等到特定條件達成時才載入其內容（含依賴的元件／模組），
在此之前可以顯示 `@placeholder` 骨架、載入中狀態 `@loading`，或在錯誤時顯示 `@error` 區塊。

常見觸發條件：
- on viewport：區塊進入可視區時載入（常用於頁面下方區塊）
- on interaction(target)：使用者互動後載入（點擊、focus、input…）
- on hover(target)：滑過目標後載入
- on idle：瀏覽器閒置時載入（不影響關鍵互動）
- on timer(ms)：延遲指定時間後載入
- when expr：當條件為真時載入（可搭配 signal/表單/路由）

可搭配 prefetch：
- prefetch on idle / on viewport / on interaction(...)：先行抓資源但不渲染，
  真正需要時能秒開，視覺更流暢。

## 二、在 `Dashboard` 頁面下方元件用 `on viewport` 延遲
在 Dashboard 下方新增一個TopHeroesChart元件，
讓它進入視窗才載入；在此之前顯示骨架或 Loading。

```sh
ng g c top-heroes-chart
```

```ts
// src/app/top-heroes-chart/top-heroes-chart.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-top-heroes-chart',
  imports: [],
  templateUrl: './top-heroes-chart.html',
  styleUrl: './top-heroes-chart.scss'
})
export class TopHeroesChart {

}
```

```html
<!-- src/app/top-heroes-chart/top-heroes-chart.html -->
<section class="chart">
  <h3>Top Heroes</h3>
  <!-- 這裡可放實際圖表或列表 -->
  <ul>
    <li>Hero A</li>
    <li>Hero B</li>
    <li>Hero C</li>
  </ul>
</section>
```

```ts
// src/app/dashboard/dashboard.component.ts
import { Component } from '@angular/core';
import { TopHeroesChart } from '../top-heroes-chart/top-heroes-chart';
import { LoadingSpinner } from '../ui/loading-spinner/loading-spinner';
import { MessageBanner } from '../ui/message-banner/message-banner';

@Component({
  selector: 'app-dashboard',
  imports: [TopHeroesChart, LoadingSpinner, MessageBanner],
  templateUrl: './dashboard.component.html',
  styleUrl: './dashboard.component.scss',
})
export class DashboardComponent {}
```

```html
<!-- src/app/dashboard/dashboard.component.html -->
<section class="dashboard">
  <h2>Dashboard</h2>
  <p class="muted">以下排行榜為次要內容，進入視窗再載入。</p>

  @defer (on viewport; prefetch on idle) {
    <app-top-heroes-chart />
  } @placeholder (minimum 300ms) {
    <div class="skeleton skeleton--chart" aria-hidden="true"></div>
  } @loading (after 100ms; minimum 400ms) {
    <app-loading-spinner label="載入排行榜..." />
  } @error {
    <app-message-banner type="error">排行榜載入失敗，稍後再試。</app-message-banner>
  }
</section>
```

```scss
/* src/app/dashboard/dashboard.component.scss */
.skeleton--chart {
  height: 260px;
  border-radius: 12px;
  background: linear-gradient(
    90deg,
    #eef1f7 25%,
    #f6f8fc 37%,
    #eef1f7 63%
  );
  background-size: 400% 100%;
  animation: shimmer 1.25s ease-in-out infinite;
}

@keyframes shimmer {
  0% { background-position: 100% 0; }
  100% { background-position: 0 0; }
}
```
**說明：**
- `on viewport`：捲動到可視範圍才載入，避免影響 LCP。
- `prefetch on idle`：瀏覽器空檔先抓檔案，真的進入視窗時就不會卡。
- `@placeholder` 與 `@loading` 可設定延遲與最短顯示時間，避免「閃一下」。

## 三、`@placeholder`、`@loading`、`@error`、`prefetch` 心法
- placeholder：空間占位、骨架畫面，讓版面穩定不跳動（避免 CLS）。
- loading：顯示「正在載入」；常搭配 after/minimum 參數避免閃爍。
- error：請用清楚的錯誤訊息與「重試」行為（可提供按鈕觸發狀態重新變化）。
- prefetch：提早抓檔但不渲染，建議搭配 on idle/on hover，平衡效能與體驗。

**常用參數（區塊標頭內）：**
- `after 100ms`：延後顯示（避免瞬間切換造成閃爍）
- `minimum 400ms`：至少顯示多久（避免一閃而過）
- 多個觸發可並列：`(on viewport; prefetch on idle)`

---

## 四、最佳實務與常見地雷
- 不要延遲「關鍵內容」：會影響 LCP/SEO/可用性。
- 底部區塊優先用 `on viewport`；互動展開的內容用 `on interaction`。
- 搭配 `prefetch` 在「使用者可能要點」的元素上（hover/focus）。
- SSR/水合：`@defer` 與 hydration 能協作，但不要延遲首屏主要內容。
- 量測再優化：配合 DevTools/ Lighthouse 檢視 LCP/CLS/INP 是否改善。
- 區塊內的依賴越重，`@defer` 的收益越大（巨大圖表、第三方套件）。

**常見錯誤與排查：**
- 內容「總是」提前出現：檢查是否誤用 `when true` 或在父層就已載入依賴。
- 骨架一閃而過：補上 `minimum` 或增加 `after`。
- 互動目標無效：確認 `#ref` 與 `on interaction(refName)` 一致。
- 效果不明顯：你延遲的區塊太小；選擇真正重的依賴再試。

**今日小結：**
`@defer` 讓我們能以「使用者情境」決定載入時機，
替初始速度與互動體驗找到平衡。把非關鍵、較重的區塊
移到「看得到/需要時」再載入，往往能帶來明顯的體感提升。

**參考資料：**
- 使用 @defer 延遲載入）：
  [https://dev.angular.tw/guide/templates/defer](https://dev.angular.tw/guide/templates/defer)