# SEAT-BOMBER-MIGRATION-002 — 工具升級 + Grid 遷移 + R0 基準重置

> 交付對象:持有舊版 grid.json 的 local agent
> 前提:手上的 grid 基於今日較早版本的 seat-bomber。目標:升級到最新版工具、grid 遷移到最新 schema、以現有佈局重置為 Round 0 基準。
> 原則不變:**改資料不改工具**。工具以 sha256 為準,禁止修改或重新實作其中程式碼。

---

## Step 1 — 更新工具

以最新版 `seat-bomber.html` 覆蓋舊版,覆蓋後驗證:

```bash
sha256sum seat-bomber.html
# 必須為 872b881dd6c327dccaaf69556243fa451fb75a2a7dcd9dd8385e4cc4dbc9d897
```

不符 = 檔案在傳輸中被動過,重新取得(必要時走 base64 通道),不要嘗試修復。

## Step 2 — Grid 遷移(舊 schema → 最新 schema)

寫一支 `migrate.py`,逐項處理:

| # | 舊狀態(若存在) | 遷移動作 |
|---|---|---|
| 1 | seats 有 `w:1.5` 或 col 為小數(浮點 MGR 時代) | 改為 1×1:`w` 欄位刪除、`col` 取整數(取 floor;若撞格則取最近空整數欄)。保留 `seat_type:"MGR"` |
| 2 | 頂層有 `flower_seats:[…]` 陣列(短暫過渡版) | 刪除頂層欄位;對其中每個 seat_id,在對應 seat 上設 `"deco":"flower"` |
| 3 | manager 對應的 member | 確保有 `"grade":"MGR"` 與 `"locked":true` |
| 4 | members 無 `product` | 不需補;product 為選填,缺省只走 sect 軸 |
| 5 | seats 無 `walkway` | 不需補;純視覺選填。若已有則原樣保留 |
| 6 | `barriers` / `raw` / 其他既有欄位 | 原樣保留,不清洗 |

## Step 3 — R0 基準重置

以「現在的佈局」作為全新的 Round 0:**刪除**以下頂層欄位(如存在):

```
baseline_assignment, round, history, banned_seats, banned_moves
```

保留 `assignment` 原樣。工具載入時會以 assignment 為 homeSeat → 已搬人數歸零、
R0 基準分 = 載入佈局的分數,一切從這裡重新起算。

## Step 4 — 驗證(全過才交付)

- [ ] `migrate.py` 可重跑(冪等)
- [ ] 無浮點 col、無 `w` 欄位、無頂層 `flower_seats`
- [ ] member/seat/assignment 三方一致(照 INTEGRATION-001 §3 清單)
- [ ] 瀏覽器載入新工具 + 新 grid:自動渲染、`ROUND 0`、`已搬(vs R0) 0 人`、
      戰隊計分板 sect 數正確、mgr 顯示 👑、小花顯示 🌼、console 無錯
- [ ] 隨手跑一輪(勾自動優化)可產生提案且不含任何 MGR

## 交付物

`migrate.py` + `grid_v2.json` + 一段簡短的遷移紀錄(各項動了幾筆)。
遇到表格外的情況照 INTEGRATION-001 §4 停下產報告,不要自行硬套。
