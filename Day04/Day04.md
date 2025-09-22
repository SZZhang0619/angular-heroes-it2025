---
post_title: 'Day 4｜模板語法：資料綁定與 @if 控制流'
author1: '阿蘇'
post_slug: 'day-04-template-binding-if'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Template Syntax
  - Control Flow
  - Property Binding
  - Attribute Binding
  - Class Binding
  - Signals
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '在範本中展示資料綁定與 @if/@else 控制，涵蓋 [prop]、[attr.xxx]、[class.xxx] 與 signals。'
post_date: '2025-09-18'
---

哈囉，各位邦友們！
昨天我們完成了第一個子元件並理解了元件的裝飾器結構。
今天來進一步把資料綁到畫面上，並用 @if/@else 做條件顯示。

## 今天要做什麼？
1. 在 App 建立 hero 狀態（signal）
2. 使用繫結動態文字與屬性/類別綁定顯示資料
3. 用 @if/@else 控制「有資料/無資料」畫面

## 前置需求
- 專案可 ng serve，[Day03](https://ithelp.ithome.com.tw/articles/10382288) 的`<app-hero-badge>`可正常渲染

**一、在 App 建立 hero 狀態**
```ts
// src/app/app.ts
import { Component, signal } from '@angular/core';
import { HeroBadge } from './hero-badge/hero-badge';

type Hero = { id: number; name: string; rank?: string };

@Component({
  selector: 'app-root',
  imports: [HeroBadge],
  templateUrl: './app.html',
  styleUrl: './app.scss'
})
export class App {
  protected readonly title = signal('hero-journey');

  // 單一英雄狀態
  protected readonly hero = signal<Hero | null>({ id: 1, name: 'Narco', rank: 'A' });
  // protected readonly hero = signal<Hero | null>(null);
}
```

**二、範本使用繫結動態文字/屬性/類別綁定 + @if/@else**
```html
<!-- src/app/app.html -->
<h1>Hello, {{ title() }}</h1>

@if (hero(); as h) {
  <section
    [title]="'Hero #' + h.id"
    [class.is-a]="h.rank === 'A'"
    [attr.aria-label]="'hero-' + h.name">
    <h2>{{ h.name }}</h2>
    <p>Rank: {{ h.rank ?? 'N/A' }}</p>
  </section>
} @else {
  <p class="muted">No hero selected.</p>
}

<!-- 保留 Day03 範例 -->
<app-hero-badge></app-hero-badge>
```

**重點：**
- 繫結動態文字 {{ }}：適合直接顯示字串/數字。
- 屬性綁定 [prop]：綁定 DOM property（例：[title]）。
- 屬性(attribute) 綁定 [attr.xxx]：綁定真正的 HTML attribute（例：[attr.aria-label]）。
- 類別綁定 [class.xxx]：依狀態切換樣式。
- @if (expr; as x) { ... } @else { ... }：以別名 x 在區塊內使用計算結果。

**三、新增樣式**
```scss
/* src/app/app.scss */
/* ...existing code... */
.is-a { 
    color: #225;
    background: #eef;
    padding: 4px 8px;
    border-radius: 4px;
}

.muted {
    color: #888;
}
```

**補充：**
- 繫結動態文字只會讀取，不會改變 DOM 屬性值。
- [prop] 綁定 property；[attr.xxx] 才是 attribute 綁定。
- 用 @if 的 as 別名存取區塊內資料。

**驗收清單：**
- 畫面顯示 Hello 與英雄資訊（Name、Rank）。
- rank 為 'A' 時 section 具 is-a 類別。
![https://ithelp.ithome.com.tw/upload/images/20250918/20159238Y86kpPhDSs.jpg](https://ithelp.ithome.com.tw/upload/images/20250918/20159238Y86kpPhDSs.jpg)
- 將 hero 設為 null 後，@else 顯示「No hero selected.」。
![https://ithelp.ithome.com.tw/upload/images/20250918/20159238tsfJhmTqc0.jpg](https://ithelp.ithome.com.tw/upload/images/20250918/20159238tsfJhmTqc0.jpg)

**常見錯誤與排查：**
- 範本錯誤：檢查 @if/@else 區塊的大括號位置。
- 綁定無效：確認使用對應語法（[prop] vs [attr.xxx]）。
- 樣式未生效：確認類別名稱與選擇器一致。

**今日小結：**
- 你現在已經會使用繫結動態文字與屬性/類別綁定等方式顯示資料了，並且用 @if/@else 控制「有資料/無資料」畫面。
- 明天我們把單一 hero 擴充成一個列表，練習 @for 與 track。

**參考資料：**
- 範本語法:
  [https://dev.angular.tw/guide/templates](https://dev.angular.tw/guide/templates)
- Properties And Attributes Binding:
  [https://dev.angular.tw/guide/templates/binding](https://dev.angular.tw/guide/templates/binding)
- Control Flow:
  [https://dev.angular.tw/guide/templates/control-flow](https://dev.angular.tw/guide/templates/control-flow)
- Signals（訊號）:
  [https://dev.angular.tw/guide/signals](https://dev.angular.tw/guide/signals)