# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾目前**尚未初始化為獨立 git 儲存庫**（2026-09-23 建立，等使用者確認要不要推公開/部署後再比照工作區慣例補建 repo）。

## 這是什麼

**跨境電商健檢總覽儀表板**，單檔前端、無後端。2026-09-23 依使用者要求建置，定位是「跨境電商工具矩陣」（`product-title-generator`／`amazon-listing-generator`／`amazon-cost-calculator`／`amazon-listing-mix-calculator`／`amazon-ppc-advisor`／`product-keyword-cloud`／`amazon-review-reply-generator`／`traffic-rank-estimator`／`amazon-logistics-game`）的收尾控制台——不做任何新的計算或內容生成，純粹把既有工具的結果集中在一頁看。

**資料來源盤點結論（建置前已逐一 grep 確認，非假設）**：這批工具裡，只有 `amazon-listing-mix-calculator` 把完整的多商品清單存進 `localStorage`（`amazonListingMixState`：`{categories:[{id,name,feePct}], products:[{id,name,categoryId,price,priceCurrency,priceTWD,stock,weeklySales,monthlySales}], weeklyThreshold,monthlyThreshold,restockLeadDays,capEnabled,capAmount,usdToTwd,jpyToTwd,netCoefficient}`）。其餘工具（標題產生器、Listing產生器、PPC建議、關鍵字雲、客評回覆、流量排名）的 `STORAGE_KEY`/`STATE_KEY` 存的都只是「最後一次表單草稿」或「跑馬燈快取」，**沒有結構化的歷史產出紀錄可供跨工具彙整**。因此本儀表板的「商品組合健檢」區塊完全基於 `amazon-listing-mix-calculator` 的資料，其餘工具只能提供「序號授權狀態」（讀取各自的序號 localStorage key）與「快速入口」，商品層級的使用進度改用本工具自己的檢查清單手動追蹤。

## 架構

`index.html` 單一 IIFE `<script>`，無建置步驟：

- **資料讀取**（`loadSource()`）— 優先讀同源 `localStorage.getItem('amazonListingMixState')`（`srcMode='live'`）；讀不到時退回本工具自己快取的 `localStorage.getItem('ecomHealthImportedState')`（`srcMode='cached'`，使用者手動貼上過的資料）；都沒有則顯示空狀態（`srcMode='none'`）。**這個「同源」機制依賴本工具部署在跟 `amazon-listing-mix-calculator` 同一個網域下**——兩者目前都預計/已經在 `m255525.github.io`（不同 repo、不同路徑，但同一個使用者站台網域，瀏覽器視為同源，`localStorage` 天然共用，做法沿用 `animal-ecommerce-adventure` 已驗證過的序號自動代入機制，見 [[animal-ecommerce-adventure-project]]）。本機用不同 port 分別跑兩個工具的 `python -m http.server` 時**不是同源**，測試時必須改成單一伺服器同時 serve 整個工作區根目錄（例如 `python -m http.server 8899 --directory "C:\Users\mark_\AI Test"`），用路徑區分兩個工具，才能驗證同源讀取邏輯——2026-09-23 建置時就是用這個方法搭配 Playwright 驗證通過（手動寫入 `amazonListingMixState` 測試資料→切到本工具頁面→確認 KPI/補貨/角色分類數字跟手算完全吻合）。
- **`calculate(state)`／`classifyProductRoles(perProduct)`** — 逐字移植自 `amazon-listing-mix-calculator/index.html` 的核心計算邏輯（毛利＝`priceTWD×銷量×(1-分類抽成%)`、淨利＝毛利×淨利係數、補貨天數分緊急/注意/正常三級、商品角色依月淨利貢獻度ABC分析），**純唯讀重算，不寫回來源工具**。若來源工具的計算公式之後又改版（該工具已經歷五輪公式修正，見其 CLAUDE.md），這裡要記得同步移植，否則兩邊數字會對不上。
- **貼上資料備援**（`buildImportBox()`/`bindImportBox()`）— 使用者在來源工具的瀏覽器 Console 執行 `copy(localStorage.getItem('amazonListingMixState'))` 後貼上，存進本工具自己的 `ecomHealthImportedState` 快取，供非同源環境（本機測試、或萬一之後部署到不同網域）使用。
- **逐商品工作流程檢查清單** — 本工具原創功能，非移植。針對 `amazon-listing-mix-calculator` 清單裡的每個商品，勾選是否已完成「標題／Listing／成本試算／PPC／關鍵字雲／客評回覆」六項（對應 `TOOLS` 陣列裡 `checklist:true` 的工具），存進本工具自己的 `localStorage['ecomHealthChecklist']`（`{[商品id]: {title:bool, listing:bool, ...}}`）。**用商品在來源工具內部的 `id` 對應**，若使用者之後在來源工具用 CSV 重新匯入整批商品導致 id 改變，先前勾選的進度會對不上新商品列——已在 `manual.html` 明確標註此限制，非 bug。
- **序號授權總覽**（`renderLicenseSection()`）— 逐一 grep 過 9 個工具的 `STORAGE_KEY` 定義取得真實 key 名稱（`ptgSerial`／`alGenSerial`／`amazonCostCalcSerial`／`amazonListingMixSerial`／`ppcAdvisorLicenseSerial`／`pkcLicenseSerial`／`arrgLicenseSerial`／`trafficRankSerial`；`amazon-logistics-game` 未列入序號檢查，`TOOLS` 陣列裡 `licenseKey:null`），純讀取該 key 是否有值，**只代表「這台瀏覽器本機是否輸入過序號」，不是即時重新呼叫各工具的 Google Apps Script 後端確認是否仍在有效期限內**——刻意不做即時重驗證，避免對 9 個不同的 Apps Script 端點發送額外請求拖慢載入速度，manual.html 已明確標註此限制。
- **快速入口**（`renderQuickLinks()`）— 9 個工具的絕對網址（已用 `curl` 逐一驗證 HTTP 200 確認真的在線，2026-09-23），非本機相對路徑，因為這 9 個工具各自是獨立 repo 部署在 `m255525.github.io/<repo-name>/`，不是同一個檔案樹底下的相對路徑。

## 刻意不做的部分（比照 `mandala-thinking`／`scamper-thinking-generator` 等 manual-first 純工具的定位）

- **無序號授權**：本工具不呼叫 AI、不曝露任何需要保護的付費邏輯，純粹是讀取使用者自己其他工具的本機資料，判斷不需要鎖。
- **無跑馬燈／PWA／訪客計數器／浮水印匯出**：這些是「公開對外教學/推廣工具」的標準配件（見 [[marquee-rollout-coverage]]／[[pwa-install-rollout]]／[[visitor-counter-rollout]]），本工具定位是使用者自己的內部管理控制台，是否要補齊這些配件待使用者確認是否要公開部署後再處理，比照 [[pref-confirm-before-deploy-new-experimental-tool]] 的既有慣例不自動加上。
- **尚未 `git init`／未推 GitHub repo／未部署 GitHub Pages**：同上，等使用者確認要不要公開後再處理。**這一步很關鍵**——本工具的核心價值（自動讀取商品健檢資料）要等它跟 `amazon-listing-mix-calculator` 一樣部署在 `m255525.github.io` 底下才會生效，只用 `file://` 或本機單獨 port 開啟時只能靠「手動貼上資料」備援。

## localStorage

- `ecomHealthImportedState` — 手動貼上資料的快取（僅在讀不到同源即時資料時使用）。
- `ecomHealthChecklist` — 逐商品工作流程勾選進度。
- 讀取（不寫入）：`amazonListingMixState` 及 9 個工具各自的序號 key（見上）。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）即可看空狀態＋貼上資料備援；要測試同源自動讀取，需跟 `amazon-listing-mix-calculator` 部署在同一個網域，或本機用單一伺服器同時 serve 整個工作區根目錄後用路徑區分兩者。

預覽伺服器：port `8821`（工作區根目錄 `.claude/launch.json` 的 `ecommerce-health-dashboard` 項目）。**注意**：用 launch.json 個別啟動本工具（單獨 port）時讀不到 `amazon-listing-mix-calculator` 的同源資料（不同 port＝不同 origin），這種情況下只能用「手動貼上資料」測試；要驗證真正的同源自動讀取，需改用 `python -m http.server <port> --directory "C:\Users\mark_\AI Test"` 同時 serve 整個工作區根目錄。
