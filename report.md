# CI/CD 作業實作報告

## 1. CI Pipeline 說明

### 主要 Workflow 內容

以下為 `.github/workflows/ci_314551133.yaml` 的核心設計：

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - "**"

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

permissions:
  contents: read
  checks: write
  pull-requests: write
  statuses: write

jobs:
  # 1. TypeScript typecheck
  typecheck:
    runs-on: ubuntu-latest
    name: TypeScript Type Check
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npm run typecheck

  # 2. Prettier check
  format-check:
    runs-on: ubuntu-latest
    name: Prettier Format Check
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npm run format:check

  # 3. Unit tests
  test:
    runs-on: ubuntu-latest
    name: Unit Tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: npm }
      - run: npm ci
      - run: npm run test
      - name: Upload test results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: Test Results
          path: test-results.xml
          reporter: jest-junit
          fail-on-error: true
          fail-on-empty: true
```

### 實作方式、工具與策略

本專案採用 GitHub Actions 構建 CI Pipeline。整個系統圍繞三個自動化驗證機制建構：

1. **觸發與環境設定**：
   - 設定 `on: push: branches '**'` 、確保在代碼推送到任何分支時自動啟動檢查。
   - 使用 `actions/setup-node@v4` 確保 Node.js 環境版本一致性（設定為 `22`）並引入 `npm ci` 快取策略。
   - 調整 Action 權限 (`permissions`)，以便開放 Github Checks API 允許回傳並渲染測試報告。

2. **Job 並行化策略**：
   - 將 Pipeline 拆分為 `typecheck` (TypeScript), `format-check` (Prettier), `test` (Vitest) 三個獨立的 Job。
   - 三個任務同時進行能顯著降低整體驗證所需時間。當多個錯誤同時發生時，開發者也能在同一輪 CI 執行中一次掌握所有錯誤資訊。

3. **測試報告視覺化（Test Reporter）**：
   - 於專案內調整 `vitest.config.ts` 的 `reporters` 屬性，在測試完畢後除了標準終端機輸出外，額外生成 `test-results.xml` (JUnit 格式)。
   - 在 `test` Job 後續利用第三方 Action `dorny/test-reporter@v1` 上傳與解析產出的 `.xml`。即便測試環節出錯（設定 `if: always()`），依然能強迫解析測試報告輸出到 GitHub 的 Check Run 頁面，便於後續除錯定位。

---

## 2. CI 執行結果截圖 (成功案例)

> [!NOTE] 
> 📝 **[請在此處插入截圖]** 
> 1. GitHub Actions 工作流程全數通過的畫面。
> 2. GitHub 頁面上渲染出的 "Test Results" 詳細測試報告截圖。

---

## 3. 失敗案例說明

> [!NOTE] 
> 📝 **[請在此處插入截圖]** 
> *請提供包含「紅色叉叉」指出某個 Job 失敗的 GitHub Actions 頁面截圖，並呈現錯誤日誌畫面。*

**錯誤情境製造與說明：**

*   **情境**：故意造成 Prettier 格式錯誤。
*   **觸發原因**：在 ci_314551133.yaml 中，使用""(雙引號)，而非''(單引號)。在開發者將程式碼強制推送到 GitHub 時，觸發了 `npm run format:check`。
*   **觀察到的結果**：由於 `npm run format:check` 反應到代碼格式不符，行程回傳了非零的結束代碼 (`exit code 1`)，使得 GitHub Actions 終止該 Pipeline 步驟並在面板上顯示 `format-check` 該項失敗。
*   **修正方式**：在本地開發環境執行 `npm run format`（底層執行 `prettier --write .`），讓工具自動且強制修正在本地的所有format，確認無誤後再次進行 commit 與 push 解決此問題。
