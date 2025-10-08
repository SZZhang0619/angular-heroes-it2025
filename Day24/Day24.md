---
post_title: 'Day 24｜架構演進：從 NgModule 到 Standalone'
author1: '阿蘇'
post_slug: 'day-24-ngmodule-to-standalone'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Standalone
  - NgModule
  - Architecture
  - Migration
  - BestPractices
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '回顧 NgModule 架構的歷史脈絡，解析 Standalone 元件帶來的改變，並整理 Hero Journey 專案實作 Standalone 的經驗與檢查清單。'
post_date: '2025-10-08'
---

哈囉，各位邦友們！
昨天我們借助 `resource()` API 整頓了 Heroes 專案的非同步狀態，感受 Signals 與 RxJS 攜手合作的威力。
今天換個角度，回到「專案骨架」本身，聊聊 Angular 從 `NgModule` 演進到 Standalone 的歷史，也複習我們專案一開始便採用的 Standalone 架構與實作細節。

## 今天要做什麼？
1. 回顧 `NgModule` 時代的設計思維與常見痛點。
2. 拆解 Standalone 架構的核心能力，對照我們在 Day02~Day23 逐步採用的技巧。
3. 彙整 Hero Journey Standalone 實作的檢查清單，確認我們的架構維持在最佳狀態。

## 前置需求
- 已完成前一篇 `resource()` 重構，對 `bootstrapApplication` 組態與 `AppConfig` 提供者不陌生。
- 熟悉 [Day02](https://ithelp.ithome.com.tw/articles/10381532) 的 Standalone 專案初始化與 [Day10](https://ithelp.ithome.com.tw/articles/10386810) 的 `provideRouter` 實作。
- 了解 Day12~Day20 建立的 HeroService、HTTP 與表單重構，便於觀察提供者轉移的影響。

---

### 一、`NgModule` 時代：集中註冊帶來的限制
過去 Angular 依賴 `NgModule` 進行組態：所有元件、指令、管線必須先被宣告 (declarations)，再由 `imports` 共享給其他模組。典型啟動流程如下：

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import { HeroesModule } from './heroes/heroes.module';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HeroesModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

這套模式在大型應用很受歡迎，但也帶來幾個實務痛點：
- 宣告、引入與導出的分工繁瑣，新人常因漏掉 `declarations` 或 `exports` 而卡關。
- 動態載入 (lazy load) 需要額外切模組，掌握成本高。
- 測試或 Storybook 想單獨載入元件，必須額外寫一個測試模組才行。

### 二、Standalone：以元件為中心的應用骨架
為了降低學習門檻，Angular 在 v14~v15 正式釋出 Standalone 元件，讓每個元件可以自帶 `imports`，應用入口也不再需要 `NgModule`。

Hero Journey 專案自一開始就採用 CLI 的 Standalone 模板，`main.ts` 透過 `bootstrapApplication()` 啟動應用：

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { App } from './app/app';

bootstrapApplication(App, appConfig).catch((err) => console.error(err));
```

`AppConfig` 則集中提供路由、HTTP、SSR 與 zoneless 設定：

```ts
import { ApplicationConfig, importProvidersFrom, provideBrowserGlobalErrorListeners, provideZonelessChangeDetection } from '@angular/core';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { HttpClientInMemoryWebApiModule } from 'angular-in-memory-web-api';
import { routes } from './app.routes';
import { InMemoryData } from './in-memory-data';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideBrowserGlobalErrorListeners(),
    provideZonelessChangeDetection(),
    provideHttpClient(withFetch()),
    importProvidersFrom(HttpClientInMemoryWebApiModule.forRoot(InMemoryData, {
      dataEncapsulation: false,
      delay: 300,
      post204: false,
      put204: false,
    })),
    provideClientHydration(withEventReplay()),
  ],
};
```

在 Standalone 世界裡：
- 每個元件都能直接透過 `imports` 使用其他 Standalone 元件或路由、表單等模組，一次完成紀錄與分享。
- `provideRouter`、`provideHttpClient` 等函式化 API，讓 [Day10](https://ithelp.ithome.com.tw/articles/10386810)、[Day12](https://ithelp.ithome.com.tw/articles/10388716) 的設定改為純函式呼叫，更好拆測試與重構。
- CLI 從 v17 起預設生成 Standalone 應用，Day03~Day07 的範例因此能專注在元件、Signals 與控制流本身。

### 三、Hero Journey 的 Standalone 實作複習
因為我們從一開始就已經從 Standalone 架構出發，今天就當成一次複習，確認專案的每個重要環節。

1. **入口啟動**：`main.ts` 透過 `bootstrapApplication()` 直接引導根元件，省去 `NgModule`。根元件 `App` 宣告 `standalone: true` 並明確列出依賴：

```ts
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterLink, RouterLinkActive, RouterOutlet],
  templateUrl: './app.html',
  styleUrl: './app.scss',
})
export class App {
  protected readonly title = signal('hero-journey');
}
```
2. **提供者集中管理**：`app.config.ts` 以 `ApplicationConfig` 包裝路由、HTTP、SSR 與 zoneless 設定，透過 `provideRouter()`、`provideHttpClient()`、`importProvidersFrom()` 等函式化 API 統一調整。我們在 Day12~Day23 完成的服務、表單與資源重構，皆使用這套提供者模型。
3. **功能模組拆散為獨立元件**：Hero Journey 的 UI 與表單元件幾乎都帶著 `standalone: true` 與專屬 `imports`，需要時直接在使用端引入，降低測試與 Storybook 建置的心智負擔。
4. **測試與開發體驗**：由於設定集中在 `ApplicationConfig`，`TestBed` 或 Storybook 的元件測試只需要匯入同一份設定即可重建環境，與傳統 `NgModule` 相比更容易維護。

保持這些原則，就能確保專案持續受惠於 Standalone 架構的模組化與清晰依賴關係。

## 驗收清單
- `main.ts` 改以 `bootstrapApplication()` 啟動，`AppModule` 不再參與流程。
- 所有元件均透過 `imports` 引入依賴，不再依附於 `declarations`。
- 路由、HTTP、SSR、Zoneless 等設定已集中在 `ApplicationConfig`，測試環境能以相同函式組態重建。

## 今日小結
Angular 從 `NgModule` 走到 Standalone，最大的價值在把焦點從模組搬回元件與功能本身。
下一篇我們會延伸到 `@defer`，繼續探索現代 Angular 的效能優化能力。

## 參考資料
- `bootstrapApplication`：
  [https://dev.angular.tw/api/platform-browser/bootstrapApplication](https://dev.angular.tw/api/platform-browser/bootstrapApplication)