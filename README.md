[README.md](https://github.com/user-attachments/files/27149004/README.md)
# 站起來了 — Health Tracker

個人健康追蹤工具。單一 HTML 檔，可部署到 GitHub Pages。

**部署網址**：https://mickeypeng530.github.io/stand/
**Mini 工具**：https://mickeypeng530.github.io/stand/mini.html

---

## 檔案結構

| 檔案 | 用途 |
|------|------|
| `index.html` | 主應用（CSS + HTML + JS 全在一起） |
| `mini.html` | 簡易加水/站起小工具（Chrome PWA 用，AHK 釘住） |
| `sprite.svg` | 14 個寵物/裝飾的 SVG symbols |
| `stand-mini.ahk` | Windows 視窗釘小工具腳本 |

---

## 核心概念

### 一日節奏
1. **按「開始工作」** → session 啟動，計時+久坐偵測
2. **久坐 N 分鐘**（門檻 5/15/30/45 mins） → 提醒
3. **按「站起來了」** → 累積站起、種子 XP
4. **按「喝了一次」** → 累積喝水（每次 0.5 杯）
5. **按「+1 零食」** → 零食計數
6. **跨午夜 12 點** → rollover：今日資料歸檔到 history、reset

### 三個寵物系統
| 寵物 | 達標條件 | 養成天數 |
|------|---------|---------|
| **久坐寵物** | 每天站起 ≥ 8 次 | 8 天 → adult |
| **喝水寵物** | 每天喝水 ≥ 6 杯 | 8 天 → adult |
| **零食寵物** | 每天零食 ≤ 3 次 | 8 天 → adult |

每隻寵物：
- Day 0-2：蛋
- Day 3：孵化（隨機種類，例如皮卡丘 or 馬力歐）
- Day 8：進化 adult，**永久進花園**走動 + 重置成新蛋

---

## State 結構（`S` 物件）

### 計數欄位
```js
todayStand    // 今日站起次數
todayWater    // 今日喝水杯數（半杯為單位）
todaySnack    // 今日零食次數
totalStands   // 累積站起總數
totalWater    // 累積喝水總數
totalSnacks   // 累積零食總數
sessionsToday // 今日 session 數
sessionSecTotal // 今日工作秒數
longestSit    // 今日最長久坐秒數
```

### 寵物欄位
```js
sitPetDays / waterPetDays / snackPetDays      // 養成天數 (0-8)
sitPetSpecies / waterPetSpecies / snackPetSpecies  // 種類（例如 'pikachu'）
sitGoalMetToday / waterGoalMetToday / snackGoalMetToday  // 今日是否達標
```

### Session 欄位
```js
inSession     // 是否在工作中
sitStartAt    // 這次坐下時間
sessionStartAt // 這次 session 開始時間
sessionAccum  // 今日累積秒數（恢復用）
lastWaterRemindAt // 上次喝水時間
alerted       // 是否已超過久坐門檻
```

### 種子/花園
```js
sitStageIdx / sitStageXP    // 久坐種子階段+XP
waterStageIdx / waterStageXP // 喝水種子階段+XP
garden        // [{type, svg, source, x, y, at}, ...]  花園所有 items
gardenArchive // {month: {plantedAt, archivedAt, items, summary}}  歷月歸檔
```

### 設定欄位
```js
thresh        // 久坐門檻（分鐘）：5/15/30/45
restToday     // 今天設成休息日
snackLimit    // 零食每日上限
theme         // 主題：default/kawaii/pixel/tama/neon/paper/pokemon/mario
bgmEnabled / bgmTrack / bgmPlaylistMode  // BGM 設定
soundEnabled / notifEnabled  // 音效/通知
histEditEnabled  // 進階：是否可編輯歷史
```

### 系統欄位
```js
lastDate      // 上次更新的日期（toDateString）
history       // 歷史紀錄 {date: {stands, water, ...}}
updatedAt     // 最後 saveSync 時間（cloud sync 衝突解決用）
firstUseAt    // 第一次使用時間（用來顯示總天數）
```

---

## 同步機制

### Firebase Realtime Database
- 路徑：`health-tracker/{uid}/`
- 認證：Google Login (deer530530@gmail.com)
- 每次 saveSync = saveLocal + cloudSave

### 三層防護（避免資料覆蓋）

#### Layer 1: 自動跨日 rollover
- `setInterval(checkDayRollover, 5 * 60 * 1000)` 每 5 分鐘檢查
- `visibilitychange` 切回前景也檢查
- 解決：tab 跨午夜開著，避免本機過時資料覆蓋雲端新資料

#### Layer 5: history 獨立路徑寫入
- 偵測到 history 變動時，**per-day path** 寫入：`history/{date}: {...}`
- **不再整體覆蓋整個 history**
- 解決：多裝置同時寫雲時 history 互相覆蓋

#### Layer 3: 自動備份
- 每 50 次 saveSync 自動 backup 到 `health-tracker-backups/{uid}/{ts}`
- 跨日 rollover 前也 backup（保留昨日完整 state）
- 保留最近 14 個（舊的自動清掉）
- 救援：可從 Firebase Console 找時間戳取回過去版本

---

## 重要函數對照

### Sync
- `loadL()` — 從 localStorage 載入 + 跑 maybeRollover
- `saveLocal()` — 寫 localStorage
- `cloudSave()` — 寫 Firebase（diff-only，per-day history）
- `cloudPull()` — 從 Firebase 讀
- `maybeRollover(d)` — 純函數，跨日歸檔 history、reset 今日
- `checkDayRollover()` — 主動偵測 + 觸發 rollover + 寫雲

### 渲染
- `renderAll()` — 全頁渲染
- `renderTimer()` — 每秒更新（session 時）
- `renderGarden()` — 花園 scene（白天/夜晚、星星、寵物 walk）
- `renderHistory()` — 歷史頁
- `updateWaterCountdown()` — 喝水倒數副字（每 30s 一次）

### 動作
- `startWork()` / `handleStandup()` — 工作 session 控制
- `doWater()` / `doSnack()` — 加水 / 加零食
- `addSeedXP(type, xp)` — 種子加 XP（觸發階段升級）
- `checkPetGoals()` — 檢查今日達標
- `checkPetLevelUp(type)` — 檢查 pet days 是否到 adult、進花園

### Edit
- `openEditModal()` — 編輯最近一次站起紀錄
- `openEditTodayModal()` — 編輯今日總計
- `openEditHistoryModal(date)` — 編輯歷史某天（進階開關）

---

## 設計決定

### 為何 8 天才 adult？
平衡達成感：太短沒意義，太長放棄。實測 8 天約 1 週多一點，符合「養成」感。

### 為何半杯為單位？
以前整杯 = 250ml。現在 0.5 杯 = 75ml（按一次 = 一口水），更符合實際喝水節奏。

### 為何 W_INT = 2 小時？
以前每 2 小時跳 toast 提醒。後來改成只顯示倒數。2 小時是文獻建議的補水間隔。

### 為何花園不全清空？
寵物 adult 是稀有成就（≥8 天達標），跨月清空殘忍。改為**寵物每年一次清空**（元旦），樹/花則跨月清。

### 為何 mini.html 沒做完整 levelup？
mini 設計就是「快速操作」，不該載入 sprite/garden 邏輯。靠主網頁 init 自動補救（檢測 `petDays >= 8` 觸發 levelup）。

---

## 已知限制

### 1. mini.html 達標時不直接進化
- mini 寫 +1 day 後，主網頁打開才會自動補 levelup
- 設計選擇：mini 保持輕量

### 2. 花園 SVG 動畫只用 CSS animation
- 寵物只能在原位置 ±30px 走動（無法跨 zone）
- 設計選擇：避免 JS 計時器太多

### 3. 跨裝置同時編輯同一天 history 仍可能衝突
- 兩台同時編輯時，後寫的勝
- 緩解：第 3 層 backup 救得回來

---

## 開發備忘

- `node -e "..."` 語法檢查 module script
- `cairosvg` 用來預覽 sprite（dev 時）
- 部署：GitHub Web 上傳，**不用 iOS Safari**（智慧引號會搞壞）
- AHK 檔必須 UTF-8 BOM
