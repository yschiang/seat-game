# SEAT-OPT Handoff Package

座位重組優化專案交付包。版本:2026-07-05。

## 內容

| 檔案 | 用途 | 給誰 |
|---|---|---|
| `seat-bomber.html` | 互動優化工具(單檔、離線可用,瀏覽器直接開) | 排座位的人 |
| `SEAT-OPT-TASK-001.md` | 任務規格 v0.2:目標優先序、四項評分公式、λ>w4 不變量、四階段流程 | 所有人(SSOT) |
| `SEAT-BOMBER-INTEGRATION-001.md` | grid.json 轉換交辦:目標 schema、轉換規則、驗證清單、升級討論條件 | 持有原始資料的 agent |
| `grid_sample.json` | 目標 schema 的完整示範資料(兩大區 80 人、12 sect、2×8 pod) | 轉換 agent 對照用 |

## 快速上手(排座位的人)

1. 瀏覽器開 `seat-bomber.html`(建議電腦;開啟即自動載入示範樓面)。
2. 拖入你的 grid.json → 🔧 地形模式修正隔板/走道 → **匯出現況**存一份正確地形版。
3. 💣 鎖定不能動的人(或戰隊卡「鎖隊」)→ 調 λ → ▶ 跑一輪 → 逐條裁決,或勾 ⚡ 自動優化。
4. 每輪開始前自動進存檔槽,隨時可 revert;遊戲內「📖 規則」有完整手冊。
5. 滿意後「匯出 moves」→ `move_list.csv`(含 raw 欄)就是最終搬遷審閱表。

## 快速上手(轉換 agent)

讀 `SEAT-BOMBER-INTEGRATION-001.md`,把手上的資料轉成 `grid_sample.json` 的格式。
原則:寫 convert 腳本、不改工具;驗證清單全過才交付;套不進來就照 §4 產報告回來討論。

## 關鍵語義備忘

- 優先序:sect 內聚 > 少搬人 > dept 靠攏(以 λ > w4 保證)。
- 位移人數與分數永遠對比 Round 0(匯出檔含 baseline_assignment,跨檔續玩不失基準)。
- 2×8 pod:pod 內兩排相鄰(小走道不建格,`walkway` 僅視覺);pod 間 `barriers` 隔板;只有主走道建 AISLE。
- raw string 跟人走,審閱時以「新座位 × raw」對照。
