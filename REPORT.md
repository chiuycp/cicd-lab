# CI 作業實作報告

## 實作內容

本次新增 `.github/workflows/ci_學號.yaml` 作為 GitHub Actions pipeline。Workflow 設定在每次 `push` 到任意 branch 時自動執行，並使用 Ubuntu runner 與 Node.js 22 安裝專案相依套件。

Pipeline 依序執行三項檢查：

1. `npm run typecheck`：執行 TypeScript `tsc --noEmit`，確認型別正確且不輸出編譯檔。
2. `npm run format:check`：執行 Prettier check，確認程式碼與設定檔符合格式規範。
3. `npm test`：執行 Vitest 測試，並同時產生 JUnit XML 測試報告。

## 工具與策略

使用 `actions/checkout` 取得 repository 內容，使用 `actions/setup-node` 安裝 Node.js 22 並啟用 npm cache，加速重複執行的 workflow。相依套件安裝採用 `npm ci`，確保 GitHub Actions 使用 `package-lock.json` 中固定的版本，降低本機與 CI 環境不一致的風險。

每個檢查都是獨立 step，任何一個指令回傳非 0 exit code 時，GitHub Actions 會直接將該 workflow run 標示為失敗。測試 step 使用 `set -o pipefail` 搭配 `tee`，確保即使測試輸出被寫入檔案，只要 Vitest 失敗，pipeline 仍會正確失敗。

## 測試結果展示方式

Vitest 測試執行時同時啟用 `default` 與 `junit` reporter，產生 `reports/vitest-junit.xml`。Workflow 後續使用 Node.js 解析 JUnit XML，將測試總數、通過數、失敗數、略過數、執行時間與每個 test case 的結果寫入 `$GITHUB_STEP_SUMMARY`。

因此測試 summary 會直接出現在 GitHub Actions workflow 的結果頁面，不需要另外進入 artifact 或 test 頁面才能看到。Workflow 仍保留 `actions/upload-artifact` 上傳完整 `reports/` 目錄，方便需要時下載原始 JUnit XML 與 Vitest 輸出。

## 格式設定調整

為了避免 Windows 本機 CRLF 與 GitHub Actions Linux LF 換行差異造成 Prettier check 誤判，`.prettierrc` 新增 `endOfLine: "auto"`。這讓 Prettier 保留既有換行風格，同時仍檢查縮排、引號、分號、逗號等格式規則。
