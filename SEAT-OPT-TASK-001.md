# SEAT-OPT-TASK-001 — 樓面座位重組優化任務規格

> 交付對象:local agent(Claude Code / 同等 agent)
> 執行模式:分階段執行,每階段結束產出 artifact 並停下等人工確認(hard gate),不得跨階段自行推進。

---

## 0. 任務目標與不變量

**目標**:重組部門座位,依組織結構 dept → sect → member 聚合,優先序:
1. 同 sect 的 members 儘量相鄰成一整塊(可直接轉頭討論)——最高優先。
2. 換位子的人數儘量少。
3. 同 dept 的各 sect 儘量彼此相鄰,聚成部門大區——最低優先,只做 tiebreak。

**優先序的參數保證**:必須維持 **λ > w4**(單人搬動成本 > dept 單筆最大增益),確保演算法絕不會純粹為了 dept 靠攏而搬人;dept 項只在移動已因 sect 內聚划算時發揮引導方向的作用。調參時此不等式為硬性約束。

**不變量(Invariants,任何階段不得違反)**:
- INV-1:原始 Excel 為唯一輸入,全程唯讀。所有產出寫入獨立檔案。
- INV-2:牆與走道格永遠不可被指派為座位。
- INV-3:座位總數與人員總數在優化前後必須一致(不憑空增減)。
- INV-4:被使用者標記為 `locked` 的人員不得移動。
- INV-5:每一版方案必須附完整移動清單,可逆(能還原到現況)。

---

## 1. 輸入

- `input/seating.xlsx`:樓面座位表。每個有人格子含:座位編碼、部門編碼、課編碼。另有走道、牆(可能以空白、底色、合併儲存格或文字表示,不保證格式一致)。

---

## 2. 工作流程(4 Phases,每個 Phase 結束為 hard gate)

### Phase 1 — 讀檔與牆/走道推測
1. 讀取 xlsx,含儲存格值、底色、邊框、合併範圍。
2. 依啟發式推測每格類型:`SEAT` / `WALL` / `AISLE` / `EMPTY_SEAT` / `UNKNOWN`。
3. 產出:
   - `artifacts/phase1_grid_draft.json`(見 §4 schema)
   - `artifacts/phase1_annotated.html`:視覺化標注圖(色塊 + 圖例),`UNKNOWN` 格高亮。
4. **GATE-1**:人工確認/修正標注。修正以覆寫 `phase1_grid_draft.json` 中 `cell_type` 或提供修正清單方式進行。

### Phase 2 — 座標模型與鄰接關係
1. 以確認後的 grid 建立座標系 `(row, col)`。
2. 建立鄰接規則(預設,可被使用者覆寫):
   - 上下左右相鄰且中間無 WALL → adjacent。
   - AISLE 隔開 → **不視為 adjacent**(內聚計算上算斷開),但距離計算仍可跨走道(距離 +2 懲罰)。
3. 產出:`artifacts/phase2_grid_model.json`(grid + 鄰接表 + 人員清單)。
4. **GATE-2**:人工確認鄰接規則(特別是走道是否隔斷、斜角是否算相鄰)。

### Phase 3 — 評分函數與現況基準
1. 評分函數(預設權重,可調):

   ```
   score = Σ_sect cohesion(sect) + Σ_dept dept_bonus(dept) − λ × moved_count

   cohesion(sect) = n_sect × [ w1 × 連通性 + w2 × 緊密度 + w3 × 距離項 ]
     連通性 = 1 − (連通區塊數−1)/n_sect     (n×連通性 = n − 區塊數 + 1)
     緊密度 = 成員四鄰中同 sect 的平均比例
     距離項 = 1 − 課內平均曼哈頓距離/樓面對角距離

   dept_bonus(dept) = w4 × (n_dept − dept連通區塊數 + 1)

   預設 w1=0.5, w2=0.35, w3=0.15, w4=0.15, λ=0.5/人
   ```

   **量級校準(關鍵)**:內聚分必須人數加權,否則正規化到 0~1 的分數會讓
   任何單筆移動的增益(~0.02)遠低於 λ,greedy 永遠找不到正分移動。
   加權後的量級直覺:sect 合併一次區塊 = +w1;dept 合併一次 = +w4;
   λ > w4 的優先序不變量在此量級下依然成立。

   設計理由:「容易互相討論」本質是連通與緊密,不是直線距離——
   w1 保證 sect 不被切塊(走道/牆/隔板都算切斷),
   w2 懲罰「連通但拉成長蛇形」的排法,
   w3 只做 tiebreak,
   w4 讓同 dept 的 sect 自然靠攏成部門大區。
2. 計算現況基準分,輸出各課碎片化報告(連通區塊數、平均距離、最碎的前 5 個課)。
3. 產出:`artifacts/phase3_baseline_report.md` + `phase3_scores.json`。
4. **GATE-3**:人工確認權重與 λ(搬動成本敏感度)。

### Phase 4 — 迭代優化(循環直到使用者滿意)
1. 演算法建議:以現況為初始解,做局部搜尋(pairwise swap / 小群搬遷),只接受 Δscore > 0 的移動;或模擬退火但限制總移動人數上限(預設 ≤ 總人數 20%,可調)。
2. 每輪產出:
   - `artifacts/round_N_plan.html`:前後對比座位圖(左現況、右方案,課別上色)。
   - `artifacts/round_N_moves.csv`:移動清單(人員、座位編碼 from → to)。
   - 分數對比:基準分 vs 方案分,各課 cohesion 變化。
3. 接受使用者追加約束(如「某人 locked」「某課必須靠窗」「兩課不可相鄰」),寫入 `constraints.json` 後重跑。
4. **GATE-4**:使用者滿意 → 輸出 `final/seating_new.xlsx`(格式與原檔一致)+ `final/move_list.csv`。

---

## 3. Guard(異常處理)

- 讀檔後若 `UNKNOWN` 格比例 > 15%,停止並回報,不得猜測後直接進 Phase 2。
- 若同一座位編碼重複、或人員格缺部門/課編碼,列入 `artifacts/data_issues.md` 並在 GATE-1 提出。
- 優化過程中若任何不變量被違反,立即中止該輪並回報。

---

## 4. Grid JSON Schema(中間 artifact 契約)

人與座位分離建模(member 有穩定 ID,搬動、鎖定、否決約束都掛在 member 上):

```json
{
  "meta": { "source": "seating.xlsx", "rows": 0, "cols": 0 },
  "members": [
    { "id": "M001", "name": "…", "dept": "D1", "sect": "S11", "locked": false }
  ],
  "seats": [
    { "seat_id": "A001", "row": 3, "col": 5, "barriers": ["E"] }
  ],
  "assignment": { "M001": "A001" },
  "cells": [
    { "row": 0, "col": 0, "cell_type": "WALL | AISLE" }
  ],
  "adjacency_rules": { "diagonal": false, "aisle_blocks_adjacency": true, "aisle_distance_penalty": 2 }
}
```

屏障語義(兩層):
- **厚牆** = `cell_type: "WALL"`,佔一格、不可坐、鄰接圖上無此節點。
- **隔板** = `barriers: ["N","E","S","W"]` 掛在 seat 上,不佔格,只移除該邊的鄰接。Loader 必須做鏡像補齊(A 格標 E ≡ 鄰格標 W),確保鄰接圖對稱。
- **走道** = `cell_type: "AISLE"`,佔一格、不可坐、隔斷 sect 連通,距離計算加懲罰。

---

## 5. 給 agent 的執行提示

- 使用 openpyxl 讀取(需含 fill/border 樣式);視覺化用單檔 HTML(inline CSS),不依賴外部服務。
- 課別配色需在同部門內可區分(同部門用同色系不同深淺)。
- 所有 artifact 檔名帶版本/輪次,不覆寫歷史。
- 每個 GATE 停下時,用一段話總結「這一步做了什麼、需要你確認什麼、預設值是什麼」。
