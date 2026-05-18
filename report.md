# CI/CD Lab 實作報告

## 專案概述

本報告詳細說明了為 Fastify 應用構建的 GitHub Actions CI/CD Pipeline 的實作方式、使用工具與策略。

---

## 1. Pipeline 自動執行 (20%)

### 實作方式

- **觸發事件**：在 `main` 和 `release/*` 分支上進行 `push` 時自動執行
- **工作流程文件**：`.github/workflows/ci_314551133.yaml`

### 配置詳情

```yaml
on:
  push:
    branches:
      - main
      - feat/cicd-homework
```

**說明**：

- 每當開發者向指定分支推送代碼時，GitHub Actions 會立即觸發 Pipeline
- 支援多個分支模式，便於在 `main` 主分支與發佈分支上同時運行驗證

---

## 2. Pipeline 檢查項 (20%)

### 包含的檢查

#### 2.1 TypeScript 類型檢查

- **命令**：`npm run typecheck`
- **工具**：TypeScript 5.9.3
- **目的**：確保代碼無類型錯誤，提高代碼品質
- **失敗條件**：任何類型不匹配或引用錯誤

#### 2.2 Prettier 格式檢查

- **命令**：`npm run format:check`
- **工具**：Prettier 3.8.3
- **目的**：驗證代碼格式符合統一的風格規範
- **失敗條件**：代碼格式不符合規範

#### 2.3 單元測試

- **命令**：`npm run test`
- **工具**：Vitest 4.1.5
- **目的**：執行所有單元測試，確保功能正確性
- **失敗條件**：任何測試用例失敗

### 執行順序

所有檢查並行執行以提高效率。

---

## 3. 檢查失敗處理 (20%)

### 實作策略

使用 GitHub Actions 的 `jobs.<job_id>.if` 條件和 `continue-on-error: false` 來確保失敗時 Pipeline 顯示為失敗。

**機制**：

- 每個步驟（step）的失敗會自動導致該工作（job）失敗
- GitHub Actions 預設會停止後續步驟的執行
- 最終結果在 GitHub UI 中顯示為 ❌ 失敗

---

## 4. 測試結果顯示 (20%)

### 實作方式

#### 4.1 方案：使用 dorny/test-reporter

- **GitHub Action**：[dorny/test-reporter@v1](https://github.com/dorny/test-reporter)
- **功能**：
  - 自動解析 Vitest 測試結果（JSON 格式）
  - 在 GitHub Actions 結果頁面顯示詳細的測試報告
  - 支援失敗測試的快速定位和診斷

#### 4.2 配置流程

1. **生成測試結果**：在 Vitest 執行時產生 JSON 報告
2. **上傳報告**：使用 test-reporter action 解析並上傳
3. **顯示結果**：在 Pull Request 檢查和 Actions 頁面上展示

#### 4.3 輸出內容

- 測試總數、通過數、失敗數
- 每個失敗測試的詳細信息
- 測試執行時間

---

## 5. 實作工具與策略 (20%)

### 開發工具

| 工具           | 版本     | 用途             |
| -------------- | -------- | ---------------- |
| Node.js        | >=22 <25 | 運行時環境       |
| TypeScript     | 5.9.3    | 類型檢查和編譯   |
| Prettier       | 3.8.3    | 代碼格式化檢查   |
| Vitest         | 4.1.5    | 測試框架和執行器 |
| GitHub Actions | 內置     | CI/CD 自動化平台 |

### 策略

#### 5.1 環境準備

- **Node.js 版本**：使用 `actions/setup-node@v4` 確保一致的運行環境
- **依賴安裝**：使用 `npm ci` 而不是 `npm install`，保證鎖定版本

#### 5.2 並行執行優化

- **多個 Job 並行運行**：typecheck、format-check、test 在不同 job 中並行執行
- **好處**：
  - 任何一個檢查的失敗不會阻止其他檢查的運行
  - 快速反饋（用戶可以看到所有問題）
  - 整體 Pipeline 執行時間更短

#### 5.3 測試報告集成

- **Vitest 配置**：產生 `vitest.json` JSON 格式報告
- **Test Reporter**：自動將結果轉換為可視化報告
- **用戶體驗**：無需進入 logs，直接在結果頁面查看測試情況

---

## 6. CI Pipeline 工作流程圖

```
┌─────────────────────────────────────┐
│      代碼 push 至 GitHub             │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   GitHub Actions Pipeline 觸發      │
└──────────────┬──────────────────────┘
               │
        ┌──────┴──────────┬─────────────┐
        ▼                 ▼             ▼
   ┌────────┐        ┌────────┐   ┌─────────┐
   │ Check: │        │ Check: │   │ Check:  │
   │TypeScript│      │Prettier│   │  Test   │
   │Typecheck │      │  Check  │   │         │
   └────┬────┘       └────┬───┘   └────┬────┘
        │                 │            │
        └─────────┬───────┴────────────┘
                  │
          ┌───────▼────────┐
          │  所有檢查完成  │
          └─────────────┬──┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
         ✅ 全部通過         ❌ 有失敗
         (綠色通過)         (紅色失敗)
              │                   │
              └───┬───────────────┘
                  │
                  ▼
        ┌──────────────────────┐
        │ 上傳測試結果報告     │
        │(test-reporter)      │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │ GitHub Results 頁面  │
        │ 顯示詳細報告信息     │
        └──────────────────────┘
```

---

## 7. 使用指南

### 本地測試 CI Pipeline

使用 `act` 工具在本地模擬 GitHub Actions：

```bash
# 安裝 act (如果未安裝)
# macOS: brew install act
# Linux: curl https://raw.githubusercontent.com/nektos/act/master/install.sh | bash

# 在本地運行 CI Pipeline
act push -b

# 查看詳細日誌
act push -b -v
```

### 觀察 Pipeline 執行

1. **Push 代碼**：`git push origin <branch>`
2. **訪問 GitHub**：進入 repo 的 **Actions** 標籤
3. **查看結果**：點擊最新的 workflow run
4. **檢查詳情**：
   - 各個 job 的執行狀態
   - 測試報告和覆蓋率
   - 日誌輸出

---

## 8. 故障排除

| 問題                       | 原因         | 解決方案                       |
| -------------------------- | ------------ | ------------------------------ |
| TypeScript 檢查失敗        | 類型錯誤     | 檢查錯誤信息，修正代碼類型     |
| Prettier 檢查失敗          | 格式不符     | 運行 `npm run format` 自動修正 |
| 測試失敗                   | 業務邏輯錯誤 | 檢查測試日誌，修正對應代碼     |
| test-reporter 無法找到報告 | 報告路徑錯誤 | 確認 Vitest 產生的報告文件位置 |

---

## 9. 未來改進

- [ ] 添加代碼覆蓋率檢查
- [ ] 添加安全掃描（依賴檢查）
- [ ] 集成自動部署步驟
- [ ] 添加性能基準測試
- [ ] 支援預提交 Hook（pre-commit）

---

## 總結

本 CI Pipeline 通過自動化的方式確保了代碼質量和可靠性：

- ✅ 在每次 push 時自動執行完整檢查
- ✅ 及時反饋所有潛在問題
- ✅ 提供可視化的測試報告
- ✅ 提高開發效率和代碼質量
