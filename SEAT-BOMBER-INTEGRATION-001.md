# SEAT-BOMBER-INTEGRATION-001 — grid.json 套入交辦

> 交付對象:持有現成 grid.json 的 agent
> 目標:把你手上的 grid.json 轉換成 seat-bomber.html 可直接載入的格式。
> 原則:**寫轉換腳本,不要修改 seat-bomber.html 本身**。工具是穩定資產,資料遷就工具。

---

## 1. 目標 Schema(工具唯一接受的格式)

```json
{
  "meta": { "source": "..." },
  "members": [
    { "id": "M001", "name": "王小明", "dept": "D1", "sect": "S11", "locked": false,
      "raw": "原始 Excel 格子字串(原封不動)" }
  ],
  "seats": [
    { "seat_id": "A001", "row": 3, "col": 5, "barriers": ["E"], "walkway": ["S"] }
  ],
  "assignment": { "M001": "A001" },
  "cells": [
    { "row": 0, "col": 0, "cell_type": "WALL" },
    { "row": 0, "col": 1, "cell_type": "AISLE" }
  ]
}
```

欄位規則:
- `members[].id`:唯一、穩定。`locked` 可省略(預設 false)。
- `seats[].barriers`:隔板,值限 `N/E/S/W`,可省略。只標一側即可,loader 會自動鏡像補齊。
- `assignment`:member id → seat_id。**每個 member 必須有一個存在的座位**;空位 = 沒被任何人指到的 seat。
- `cells`:只放 WALL / AISLE。**座位不要放進 cells**(座位由 seats 陣列定義)。
- 座標:row/col 從 0 起算的整數格點,同一 (row,col) 不得同時是 seat 又是 cell。

## 2. 常見來源格式的轉換規則

若你的 grid.json 是 **cell-based 舊格式**(每格帶 cell_type=SEAT + dept/section/seat_id):
1. `cell_type ∈ {WALL, AISLE}` → 原樣進 `cells`。
2. `cell_type ∈ {SEAT, EMPTY_SEAT}` → 進 `seats`(帶 seat_id/row/col/barriers)。
3. 有 dept+section 的 SEAT → 生成一個 member(`id` 用人員編號;沒有就用 `seat_id` 當初始 member id,並在報告中註明),寫入 `assignment`。
4. 欄位名對映:`section`→`sect`;其他名稱差異一律在轉換腳本中處理。
5. **raw string 跟人走**:原始格子的完整字串放進 `members[].raw`,原封不動、不解析、不清洗。它屬於 member 不屬於座位——人搬到哪,raw 跟到哪,最終審閱時用「新座位 × raw」對照確認。座位本身不保留 raw。
6. **2×8 pod 的建模**:pod 內兩排之間的窄走道**不建格**——兩排直接建成相鄰的兩列(背對背轉身可討論 = 相鄰,語義正確)。窄走道千萬不要建成 AISLE cell,否則會錯誤地隔斷 pod 內連通。要在畫面上呈現這條窄走道,在 pod 上排的 seat 標 `"walkway": ["S"]`(**純視覺欄位**,不影響任何鄰接或計分)。pod 與 pod 之間的隔板用 `barriers` 建在邊界列上。只有「真正走人的主走道」才建 AISLE cell。

## 3. 驗證清單(轉換後必跑,全過才交付)

- [ ] member id 無重複;seat_id 無重複;(row,col) 無重複
- [ ] assignment 中每個 member 存在於 members、每個座位存在於 seats
- [ ] 沒有兩個 member 指到同一座位
- [ ] barriers 值只含 N/E/S/W
- [ ] WALL/AISLE 座標不與任何 seat 重疊
- [ ] 統計輸出:人數、座位數、空位數、sect 清單與各 sect 人數、dept 清單 —— 與原始資料一致
- [ ] 實際用瀏覽器載入 seat-bomber.html 拖入產出檔,確認:樓面渲染正常、人數與計分板 sect 數正確、無 console error

## 4. 套不進來時 — 升級討論(不要自行硬套)

以下任一情況,**停止轉換**,產出報告回來討論,不要猜測語義:

| 情況 | 報告需附 |
|---|---|
| 有 member 沒有座位,或座位數 < 人數 | 缺口清單 |
| 一個格子有多人(共用座位?輪班?) | 該格原始資料樣本 |
| 座標不是規則格點(自由平面座標、非整數) | 座標分布摘要 |
| 有目標 schema 表達不了的元素(柱子、門、高度差、跨樓層) | 元素類型與數量 |
| dept/sect 層級對不上(如中間多一層 team,或一人多 sect) | 組織欄位原始樣本 |
| 隔板無法對映到格子邊(斜的、跨多格的) | 隔板原始表示法 |

報告格式:`INTEGRATION-REPORT.md`,含:卡住的規則編號、原始資料 5–10 筆樣本(去識別化)、你建議的 2–3 個處理選項與各自代價。

## 5. 交付物

1. `convert.py`(或同等):原格式 → 目標格式,可重跑。
2. `grid_converted.json`:轉換結果。
3. `CONVERSION-NOTES.md`:對映決策、驗證清單勾選結果、統計輸出。
