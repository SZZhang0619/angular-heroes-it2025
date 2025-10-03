---
post_title: 'Day 2｜環境設定：Standalone 專案結構'
author1: '阿蘇'
post_slug: 'day-02-setup-standalone-structure'
microsoft_alias: 'SZZhang0619'
featured_image: 'https://angular.io/assets/images/logos/angular/angular.svg'
categories:
  - Angular
tags:
  - Angular CLI
  - Standalone
  - SSR
  - Zoneless
  - Project Structure
  - TypeScript Paths
ai_note: 'This post was assisted by AI (GitHub Copilot).'
summary: '使用 Angular CLI v19 建立 Standalone 專案，導覽核心設定並示範啟用 SSR 與 zoneless。'
post_date: '2025-09-16'
---

哈囉，各位邦友們！
今天我們試著用最新的 Angular CLI 來建立一個 Standalone 專案，並快速認識專案結構。

## 今天要做什麼？
1. 用 Angular CLI 建一個全新專案（Standalone 預設啟用）
2. 啟動開發伺服器，確認首頁正常
3. 讀懂 main.ts、app.component.ts、index.html、angular.json、tsconfig* 的用途

## 前置需求
- Node.js ^20.19.0（建議 20 LTS）

**一、建立專案與啟動**
```sh
npm i -g @angular/cli@latest

ng new hero-journey --style=scss --routing=false

? Do you want to enable Server-Side Rendering (SSR) and Static Site Generation
(SSG/Prerendering)? (y/N) y

? Do you want to create a 'zoneless' application without zone.js? (y/N) y

? Which AI tools do you want to configure with Angular best practices?
https://angular.dev/ai/develop-with-ai (Press <space> to select, <a> to toggle
all, <i> to invert selection, and <enter> to proceed)
❯◉ None
 ◯ Claude         [ https://docs.anthropic.com/en/docs/claude-code/memory
     ]
 ◯ Cursor         [ https://docs.cursor.com/en/context/rules
     ]
 ◯ Gemini         [ https://ai.google.dev/gemini-api/docs
     ]

cd hero-journey
ng serve

Would you like to share pseudonymous usage data about this project with the
Angular Team
at Google under Google's Privacy Policy at https://policies.google.com/privacy.
For more
details and how to change this setting, see https://angular.dev/cli/analytics.

(y/N) N
```

**二、專案結構快覽（你需要先認識的檔案）**
- src/main.ts
  - 以 bootstrapApplication(AppComponent) 啟動應用
  - 未來會在這裡加入 providers（如 provideRouter、provideHttpClient）
  - 範例：
```ts
// src/main.ts
// ...existing code...
bootstrapApplication(AppComponent, {
  providers: [
    // 未來：provideRouter(routes), provideHttpClient(), ...
  ]
});
// ...existing code...
```
- src/main.server.ts
  - SSR 端的 bootstrap 入口。
- src/server.ts
  - Express 整合，提供靜態檔案與將請求交給 Angular SSR。
- src/app/app.config.ts
  - 提供 provideZonelessChangeDetection() 與 provideClientHydration(withEventReplay())。
- src/app/app.config.server.ts + src/app/app.routes.server.ts
  - 伺服器端 providers 與預渲染路由（RenderMode.Prerender）。
- src/index.html
  - 掛載點 <app-root>。
- angular.json
  - Angular 專案的開發/建置設定檔（CLI 會讀取）。
- tsconfig.json / tsconfig.app.json / tsconfig.spec.json
  - 嚴格型別設定；可在此擴充 paths 以優化 import。
- public/
  - 靜態資源的根目錄（由 angular.json 的 assets 指向）。

**三、簡單練習：設定路徑別名**
- 後續若想優化 import，可在 tsconfig.json 增加 paths
```json
{
  "compilerOptions": {
    // ...existing options...
    "paths": {
      "@app/*": ["src/app/*"],
      "@assets/*": ["src/assets/*"]
    }
  }
}
```

**驗收清單：**
1. 可以成功 ng serve，首頁正常顯示
2. 說得出以下檔案的用途：main.ts、app.component.ts、index.html、angular.json、tsconfig.json

**常見錯誤與排查：**
- Node 版本過舊：升級到 20 LTS（^20.19.0）
- 埠被占用：ng serve --port 4300

**今日小結：**
- 我們使用全域 CLI 建好專案並跑起首頁。從今天開始，你的開發體驗就是現代 Angular 的樣子，明確、快速、直覺。

**參考資料：**
- Angular CLI:
  [https://dev.angular.tw/cli](https://dev.angular.tw/cli)
- ng new:
  [https://dev.angular.tw/cli/new](https://dev.angular.tw/cli/new)
- bootstrapApplication:
  [https://dev.angular.tw/api/platform-browser/bootstrapApplication](https://dev.angular.tw/api/platform-browser/bootstrapApplication)
- provideRouter:
  [https://dev.angular.tw/api/router/provideRouter](https://dev.angular.tw/api/router/provideRouter)
- provideHttpClient:
  [https://dev.angular.tw/api/common/http/provideHttpClient](https://dev.angular.tw/api/common/http/provideHttpClient)
- SSR（伺服器端渲染）:
  [https://dev.angular.tw/guide/ssr](https://dev.angular.tw/guide/ssr)
- provideZonelessChangeDetection:
  [https://dev.angular.tw/api/core/provideZonelessChangeDetection](https://dev.angular.tw/api/core/provideZonelessChangeDetection)
- TypeScript paths:
  [https://www.typescriptlang.org/tsconfig#paths](https://www.typescriptlang.org/tsconfig#paths)
