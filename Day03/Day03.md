---
post_title: 'Day 3｜核心概念：Component 詳解'
author1: '阿蘇'
post_slug: 'day-03-component-basics'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Component
  - Standalone
  - Imports
  - Decorator
  - Angular CLI
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '拆解 @Component 裝飾器，建立 HeroBadge 子元件並透過 imports 組合畫面。'
post_date: '2025-09-17'
---

哈囉，各位邦友們！
昨天我們把專案建好，今天正式走進 Angular 的核心：Component。先抓住 @Component 的骨架，再做一個小小的子元件，最後以 imports 串起父子關係，為明天的範本語法與 @if 鋪路。

## 今天要做什麼？
1. 讀懂 @Component：selector、templateUrl/styleUrl、imports
2. 用 CLI 建立 HeroBadge元件
3. 在 App 元件透過 imports 使用子元件

## 前置需求
- 已完成 [Day 2](https://ithelp.ithome.com.tw/articles/10381532) 專案初始化並可以成功 ng serve，首頁正常顯示

**一、@Component**
- selector：HTML 會用到的標籤名
- templateUrl：外部範本
- styleUrl：外部樣式
- imports：此元件需要用到的其他元件/指令/管道
```ts
// src/app/app.ts
import { Component, signal } from '@angular/core';
// ...existing code...

@Component({
  selector: 'app-root',
  imports: [
    // 之後放入 HeroBadgeComponent 或常用功能
  ],
  templateUrl: './app.html',
  styleUrl: './app.scss'
})
export class App {
  protected readonly title = signal('hero-journey');
}

// src/app/app.html
<h1>Hello, {{ title() }}</h1>
<!-- 之後會放入 <app-hero-badge /> -->

// src/app/app.scss
:host {
    display: block;
    padding: 16px; 
}

h1 {
    margin: 0 0 12px; 
}
```

**二、用 CLI 產生子元件 HeroBadge元件**
```sh
ng g component hero-badge
```
產生後，打開檔案並確認：
```ts
// src/app/hero-badge/hero-badge.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-hero-badge',
  imports: [],
  templateUrl: './hero-badge.html',
  styleUrl: './hero-badge.scss'
})
export class HeroBadge {}

// src/app/hero-badge/hero-badge.html
<span class="badge">New Hero</span>

// src/app/hero-badge/hero-badge.scss
:host {
  display: inline-block;
}

.badge {
  padding: 4px 8px;
  border-radius: 4px;
  background: #eef;
  font-weight: 600;
}
```

**三、在 App 使用子元件（透過 imports）**
```ts
// src/app/app.ts
// ...existing code...
import { HeroBadge } from './hero-badge/hero-badge';

@Component({
  selector: 'app-root',
  imports: [HeroBadge],
  templateUrl: './app.html',
  styleUrl: './app.scss'
})
export class App {
  protected readonly title = signal('hero-journey');
}

// src/app/app.html
<h1>Hello, {{ title() }}</h1>
<app-hero-badge></app-hero-badge>
```

**說明：**
- 在元件內，若父元件想用子元件，必須把子元件加進 imports。

**簡短 QA：**
- Q：imports 要放什麼？
  - A：這個元件會用到的東西才放。

**驗收清單：**
- 畫面顯示 <h1>Hello, {{ title() }}</h1> 與一個 “New Hero” 樣式標章
  ![New Hero badge screenshot](https://ithelp.ithome.com.tw/upload/images/20250917/20159238iTMfevcRgS.jpg)
- 能解釋 component 的基本概念

**常見錯誤與排查：**
- 看不到 <app-hero-badge>：確認子元件已加入 App.imports
- 標籤拼錯：selector 與範本標籤需一致（預設 app-hero-badge）
- 樣式沒生效：請把樣式寫在子元件內

**今日小結：**
- 你已經掌握 Standalone 元件的核心用法，並能以 imports 串起父子元件。

**參考資料：**
- Components（元件剖析）:
  [https://dev.angular.tw/guide/components](https://dev.angular.tw/guide/components)
- @Component 裝飾器:
  [https://dev.angular.tw/api/core/Component](https://dev.angular.tw/api/core/Component)
- Angular CLI generate:
  [https://dev.angular.tw/cli/generate](https://dev.angular.tw/cli/generate)
