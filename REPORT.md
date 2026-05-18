# CI/CD 作業實作報告

## 一、作業檔案位置

- 實作的 workflow 檔案：[.github/workflows/ci_r14228008.yaml](.github/workflows/ci_r14228008.yaml)
- 實作報告：[REPORT.md](REPORT.md)（本檔）

---

## 二、評分對應一覽

| 評分項目                | 比重 | 對應實作                                                                            |
| ----------------------- | ---- | ----------------------------------------------------------------------------------- |
| 自動觸發 (push)         | 20%  | `on: push` + `on: pull_request`，任何分支 push 皆會觸發                             |
| 核心檢查 (typecheck / prettier / test) | 20% | 三個獨立 step：`npm run typecheck`、`npm run format:check`、`npm test` |
| 錯誤阻斷機制            | 20%  | 每個 step 用非零 exit code 自動讓 Job 失敗；不額外吞錯誤                            |
| 測試結果呈現            | 20%  | (a) Vitest JUnit XML → `dorny/test-reporter@v2` 顯示成 Check；(b) Upload artifact；(c) `$GITHUB_STEP_SUMMARY` 統計摘要 |
| 實作報告                | 20%  | 本文件                                                                              |

---

## 三、實作方式說明

### 3.1 Workflow 整體結構

`.github/workflows/ci_r14228008.yaml` 採用 **單一 job + 多個 step** 的設計，使用 Ubuntu Runner 與 Node.js 22（對應 `package.json` 的 `"engines": { "node": ">=22 <25" }`）。

```yaml
on:
  push:
  pull_request:
```

- `on: push` 滿足「在 push 時自動執行」的要求，沒有指定 `branches` 表示**所有 branch push 都會觸發**。
- 額外加上 `pull_request`，使 PR 也能取得品質檢查結果，符合一般 CI 慣例。

並加入 `concurrency` 設定，當同一 ref 連續多次 push 時，前一次未完成的 run 會被取消，避免浪費 Runner minutes：

```yaml
concurrency:
  group: ci-assignment-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### 3.2 核心三項檢查

在「Install dependencies (`npm ci`)」之後依序執行：

| Step                          | 指令                  | 來源                                    |
| ----------------------------- | --------------------- | --------------------------------------- |
| TypeScript typecheck          | `npm run typecheck`   | `package.json` → `tsc --noEmit`         |
| Prettier check                | `npm run format:check`| `package.json` → `prettier --check .`   |
| Run tests (with JUnit output) | `npm test -- --reporter=default --reporter=junit --outputFile.junit=reports/vitest-junit.xml` | Vitest 內建多 reporter |

為了讓使用者在**一次 run 中看到所有失敗**，prettier、test 等後續 step 使用 `if: ${{ !cancelled() }}`，意思是「只要工作流程沒被取消就繼續跑」。這樣即使 typecheck 失敗，後續 prettier check 與 test 仍會執行，但 Job 整體最終仍會因為 typecheck 的非零 exit code 而被標記為 **failed**。

### 3.3 錯誤阻斷機制

GitHub Actions 預設行為：**任何 `run:` step 回傳非 0 exit code 即視為失敗，並讓整個 job/workflow 失敗**。本實作完全採用此預設行為，沒有使用 `continue-on-error: true`，也沒有用 `|| true` 等手法吞掉錯誤。

驗證方式：

1. **故意打壞 typecheck**：在 `src/app.ts` 加入 `const x: number = 'hello';` → push → workflow 顯示 ❌ failed。
2. **故意打壞 prettier**：把任一 `.ts` 檔案的 quote 改成不一致的格式 → push → workflow 顯示 ❌ failed。
3. **故意打壞 test**：把 `test/app.test.ts` 中的 `expect(response.statusCode).toBe(200)` 改成 `toBe(999)` → push → workflow 顯示 ❌ failed，且 Vitest Report 中可看到失敗的測項。

### 3.4 測試結果呈現

採用 **三層展示策略**，確保結果在 GitHub Actions 結果頁面充分可見：

#### (a) Vitest 產出 JUnit XML

Vitest 4 內建 JUnit reporter。指令中同時掛上 `default` 與 `junit` 兩個 reporter，所以：

- `default` reporter：仍在 step log 上顯示彩色測試輸出，方便 debug
- `junit` reporter：把結果寫到 `reports/vitest-junit.xml`，供下游 actions 解析

#### (b) `dorny/test-reporter@v2` 將結果發成 GitHub Check

這是市面上最受歡迎的 JUnit → GitHub Check 對接 action：

```yaml
- name: Publish test results to Actions UI
  if: ${{ !cancelled() }}
  uses: dorny/test-reporter@v2
  with:
    name: Vitest Report
    path: reports/vitest-junit.xml
    reporter: jest-junit
    fail-on-error: false
```

執行後會在該次 commit / PR 上產生名為 **「Vitest Report」** 的獨立 Check，內含每個測試的通過/失敗結果。`fail-on-error: false` 是因為錯誤訊號已經由 `npm test` 自己 propagate，這個 action 只負責**呈現**。

需要 `permissions: checks: write` 權限才能寫 check，已在 workflow 最上層設定。

#### (c) Artifact 上傳

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: vitest-junit-report
    path: reports/
```

完整的 JUnit XML 會被打包成 artifact，使用者可在 run 頁面下載原檔做更深入分析。

#### (d) `$GITHUB_STEP_SUMMARY` Markdown 摘要

最後一個 step 解析 JUnit XML，把總測試數、失敗數、錯誤數、跳過數寫到 `$GITHUB_STEP_SUMMARY`，這會以 Markdown 表格直接顯示在 Actions run 的最上方摘要區，**不需點進 step log 就能一眼看到結果**。

### 3.5 對 source code / config 的修改

依作業說明「可修改專案中的 source code / config」，本次修改僅針對 prettier check 失敗的 3 個原始檔案做格式化：

```
docker-compose.yml
snippets/01_hello.yaml
snippets/02_run-test.yaml
```

修改方式：執行 `npx prettier --write` 讓它們符合專案根目錄 `.prettierrc` 的設定（`singleQuote`、`printWidth: 100` 等）。**未修改任何邏輯碼**，僅是空白/引號樣式的調整，不影響任何 lab 行為。

未修改 `tsconfig.json`、`vitest.config.ts`、`package.json`，保留原專案結構。

---

## 四、本地驗證流程

在 push 前，於本機跑過一次：

```bash
npm ci
npm run typecheck      # ✅ pass
npm run format:check   # ✅ pass（修復格式後）
npm test               # ✅ 1 file, 2 tests passed
```

也可使用 `act`（專案已內建 `.actrc`）在本地模擬：

```bash
act push -W .github/workflows/ci_r14228008.yaml
```

---

## 五、使用工具與設計策略

### 工具清單

| 工具                            | 用途                                                |
| ------------------------------- | --------------------------------------------------- |
| `actions/checkout@v4`           | Checkout 程式碼                                     |
| `actions/setup-node@v4`         | 安裝 Node 22 並啟用 npm cache                       |
| Vitest 4 內建 `--reporter=junit`| 產出 JUnit XML 測試報告                             |
| `dorny/test-reporter@v2`        | 把 JUnit XML 轉成 GitHub Check，於 Actions 頁面顯示 |
| `actions/upload-artifact@v4`    | 上傳測試報告原檔做為 artifact                       |
| `$GITHUB_STEP_SUMMARY`          | 在 run 摘要區直接顯示 Markdown 統計                 |

### 設計策略

1. **單一 Job + 線性 Steps**：作業範圍小且檢查項彼此邏輯相關，沒有必要切多個 job 帶來 checkout/install 的重複成本。

2. **`if: ${{ !cancelled() }}` 一次回報所有失敗**：相較於前一步失敗就中斷，把所有檢查跑完能讓使用者**在一次 push 內看到所有需要修的問題**，減少 push → 失敗 → 修一個 → 再 push 的循環。

3. **錯誤阻斷靠每個 step 的 native exit code**：不在 workflow 內額外吞錯或包 try/catch，相信 npm / tsc / prettier / vitest 各自的 exit code。

4. **三層測試結果呈現**：
   - 表面層：`$GITHUB_STEP_SUMMARY`（不點 step 就看得到）
   - 結構層：`dorny/test-reporter` 產生 Check（per-test 結果）
   - 原檔層：Artifact 下載 JUnit XML（可餵到其他系統）

5. **Concurrency 取消重複 run**：減少 runner 資源浪費；對同一分支來說，永遠以最新一次 push 的結果為準。

6. **Node 版本鎖在 22**：對應 `package.json` 的 `engines` 條件，與本地開發環境一致，避免 CI 與本機行為不一致。

---

## 六、CI Pipeline 主要內容（貼上 workflow 全文）

下方為 `.github/workflows/ci_r14228008.yaml` 的完整內容：

```yaml
name: CI - Quality Checks (Assignment)

on:
  push:
  pull_request:

permissions:
  contents: read
  checks: write
  pull-requests: write

concurrency:
  group: ci-assignment-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  quality:
    name: TypeScript / Prettier / Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: TypeScript typecheck
        run: npm run typecheck

      - name: Prettier check
        if: ${{ !cancelled() }}
        run: npm run format:check

      - name: Run tests (with JUnit output)
        if: ${{ !cancelled() }}
        run: |
          mkdir -p reports
          npm test -- \
            --reporter=default \
            --reporter=junit \
            --outputFile.junit=reports/vitest-junit.xml

      - name: Publish test results to Actions UI
        if: ${{ !cancelled() }}
        uses: dorny/test-reporter@v2
        with:
          name: Vitest Report
          path: reports/vitest-junit.xml
          reporter: jest-junit
          fail-on-error: false

      - name: Upload JUnit report artifact
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: vitest-junit-report
          path: reports/
          if-no-files-found: warn

      - name: Append summary to GitHub Step Summary
        if: ${{ !cancelled() }}
        run: |
          # 解析 JUnit XML 並寫入 $GITHUB_STEP_SUMMARY（內容略，見原檔）
```

---

## 七、CI 執行結果截圖（成功案例）

> 請將下列三張截圖插入此處：
>
> ![成功 run 概覽](screenshots/success_overview.png)
>
> ![Step 列表全綠](screenshots/success_steps.png)
>
> ![Summary 區 Vitest 統計表](screenshots/success_summary.png)

成功 run 的特徵：

- 所有 9 個 step 顯示 ✅
- `Vitest Report` check 顯示 2/2 tests passed
- Summary 區顯示 Total tests: 2 / Failures: 0 / Errors: 0 / Skipped: 0
- Artifact `vitest-junit-report` 可下載

---

## 八、失敗案例說明

### 8.1 故意製造的錯誤

修改 [test/app.test.ts](test/app.test.ts)，把：

```ts
expect(response.statusCode).toBe(200);
```

改成：

```ts
expect(response.statusCode).toBe(999);
```

Push 後 CI 立刻變成 ❌ failed。

### 8.2 失敗截圖

> 請將下列截圖插入此處：
>
> ![失敗 run 概覽](screenshots/fail_overview.png)
>
> ![失敗 step log](screenshots/fail_step_log.png)
>
> ![Vitest Report 顯示失敗測項](screenshots/fail_vitest_report.png)

### 8.3 錯誤原因

`GET /health` 實際回傳 HTTP `200`，但測試斷言期望 `999`。Vitest 拋出 assertion error，`npm test` 以非零 exit code 結束，連帶讓 GitHub Actions 的 `Run tests (with JUnit output)` step 變成失敗，整個 workflow run 標記為 ❌ failed。

這同時驗證了「**錯誤阻斷機制**」要求：任何一個檢查失敗都會讓 pipeline 失敗，不會被誤判成通過。

### 8.4 修正方式

將測試斷言改回實際的 HTTP 狀態碼 `200`：

```ts
expect(response.statusCode).toBe(200);
```

重新 push 後 CI 回到 ✅ 綠燈。

---

## 九、檔案結構摘要

```
.github/workflows/ci_r14228008.yaml   ← 本次作業實作
REPORT.md                              ← 本報告
docker-compose.yml                     ← 格式化修正
snippets/01_hello.yaml                 ← 格式化修正
snippets/02_run-test.yaml              ← 格式化修正
```

其餘 source code（`src/`、`test/`、`package.json`、`tsconfig.json`、`vitest.config.ts` 等）均未變動。
