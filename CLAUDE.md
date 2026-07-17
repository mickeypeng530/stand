# CLAUDE.md — stand(久坐/喝水追蹤 + 花園養成)

> 最後更新:2026-07-17。決策史在 `DECISION_LOG.md`。

## 1. 這專案在做什麼

彭嗣翔醫師自用的健康追蹤 app:久坐提醒、喝水/零食記錄、花園養成(種子 XP 長植物、目標天數養寵物)。GitHub Pages 部署(push main 自動更新),資料在 Firebase RTDB `case-scheduler-f752b` 的 `health-tracker/{uid}`。

**三個寫入端共用同一節點**:
| 端 | 檔案 | 角色 |
|---|---|---|
| index.html(~5100 行) | 主 app | 完整功能:session/花園/寵物升級/歸檔 |
| mini.html(~900 行) | 輕量記錄器 | 只記數(水/站/零食),**無升級/花園邏輯**,user 日常主力 |
| 桌寵 DeskPet | `../Pet/cloud.js` | Electron 桌面寵物,機器帳號(UID `2ct8XD…`)寫入,行為比照 mini |

⚠ **`../OPD/stand.html` 是 6 月前的舊版(3534 行),勿當參考** —— 線上版 v2.5-0602 schema 已大改。

## 2. 現在進度

- 2026-07-17 **「69 水滴滴」事件**已修(3 commits,見 DECISION_LOG):寵物重複畢業迴圈 —— 根因 = 計數用 Math.max 合併 → 歸零無法傳播 + 畢業無防重入 + mini 無上限累加。資料已清理(花園 81 筆、計數歸零;備份在 `../Pet/.backup-health-tracker-*.json`)。
- **code review 12 條 findings 待處理**(見 §4 後的表),優先序:#4 > #8 > #3 > 其餘。

## 3. 架構速覽(同步機制 = 一切 bug 的核心)

**Schema v2.5**(`_schemaVersion:2`),節點下命名空間:
```
health-tracker/{uid}/
  state/*      今日計數/session/lastDate/updatedAt … LWW(cloudUA > localUA 才整批採用)
  pets/*       寵物天數/種類/種子 stage … 數字一律 Math.max 合併(⚠ 歸零傳不出去)
  meta/*       totalStands/totalWater/totalSnacks … Math.max(only-grow,安全)
  settings/*   門檻/主題/_gardenPeriodKey …
  history/{day}   per-day union + 欄位 Math.max(mergeDayMax)
  garden/{月key}/{at}   植物/寵物,union by at,永不去重
  gardenArchive/{月key}  跨月歸檔快照
  (頂層還殘留 v1 同名欄位 = 遺物,v2 client 不讀)
```
- **同步流程**:`cloudSave()` 差異推送(對 `lastCloudSnap` diff,只推有變的 key)→ 別的裝置 `onValue` listener 收到 → `normalizeCloudSnap()` → 依上表規則合併 → `maybeRollover(S)` → render。
- **lastDate 格式** `new Date().toDateString()`;rollover:歸檔 history、重置 today*、算 streak、零食寵 +1。
- 寵物:目標達成(8站/6杯/零食≤上限)→ `*PetDays+1`;`===3` 孵化定種、`>=8` 畢業進花園(index 才會做;已加同日防重入 guard)。

## 4. 常見坑(root-cause 家族)

- **🔴 Math.max 合併無法表達「歸零/遞減」**:任何 stale client 的舊高值都會復活歸零後的計數。已爆:寵物畢業(69 水滴滴);同族未爆:種子 XP(#8)、undo 站起(#7)、歷史下修(#6)。根治方向 = 這些欄位改 LWW 或 per-field tombstone。
- **🔴 `delete S[key]` 推不上雲**:cloudSave 只遍歷「存在的 key」→ 刪除永遠不同步 → 旗標/歷史日在雲端卡死復活。已爆:`_pendingSnackLevelCheck` 轟炸;同族:刪除歷史日(#3)。
- **mini 是功能子集**:凡 index 在 rollover/goal 之外做的事(streak、零食寵、種子收成、花園),mini 沒有 → mini 先跑 rollover 就整天漏掉(#4、#9)。
- **精確門檻**:孵化用 `===3` —— 計數跳過去就永遠不觸發(畢業已改 `>=8`,孵化還是 `===`)。
- **桌寵是第三寫入端**:schema/rollover/合併規則任何變動,**必須同步改 `../Pet/cloud.js`**(它複刻了 mini 的 rollover 與 goal 邏輯)。

## 5. Code review findings(2026-07-17,#1 #2 已修)

| # | 嚴重度 | 位置(行號為當時) | 問題 |
|---|---|---|---|
| 3 | 🔴 | index deleteEditHistory ~L4880 | 刪除歷史日只 delete 本機,雲端 union 加回來;SANITY_PUSH 也會推回 → 需 null 推送 + tombstone |
| 4 | 🔴 | mini maybeRollover L186-230 | mini 的 rollover 沒有 streak 與零食寵邏輯;mini 先開就整天漏 → streak 永遠不動。修法:mini 補上這兩段,或 index 偵測「lastDate 被別端前進」時補算(`_rolledOver` 是沒人讀的 dead flag 可利用) |
| 5 | 🟡 | index cloudPull ~L2317 | 裸 `get()` 無 timeout,卡住則 `_syncInFlight` 永久鎖死 → 移植 mini 的 `getWithTimeout` |
| 6 | 🟡 | index saveEditHistory ~L4856 | 歷史數字改小會被別端 mergeDayMax 推回 → day entry 加 `editedAt` 走 LWW |
| 7 | 🟡 | index rollbackLastStand ~L4686 | undo 的 sitPetDays 遞減被 Math.max 推回 |
| 8 | 🟡 | PETS_KEYS 內 stage 欄位 | **種子 XP 同款復活 bug**:完成歸零後 stale client 推舊 idx/XP → 重複收成。與 69 水滴滴同機制,同日 dedupe 擋不了(種子一天可多次完成)→ stage 欄位改 LWW |
| 9 | 🟡 | mini addSeedXP L705-709 | mini 種子完成只歸零不種花,收成憑 index 殘留狀態 → 設 `_pendingHarvest` 旗標(入 STATE_KEYS,勿用 delete 清) |
| 10 | 🟡 | index checkMonthlyArchive ~L5003 | 首次 cloudPull 前就歸檔 → stale 裝置月初先開會歸檔不完整花園 → dbRef 可期待時跳過 init 端歸檔 |
| 11 | 🟢 | mini | 掛著跨午夜不 rollover → doWater/handleStandup 開頭補 `if(maybeRollover(S))saveSync()` |
| 12 | 🟢 | index checkDayRollover ~L5065 | 跨午夜後 UI 停在 session 模式 → rolled 分支補 `applySessionUI()` |

## 6. 接手者 cheatsheet

```powershell
cd C:\Users\彭嗣翔\Claude_Work\stand
# 部署 = push main → GitHub Pages 自動(1-2 分鐘);各裝置要強制刷新(Ctrl+F5)
git push origin main
```
- 雲端資料檢視:用 `../Pet/secrets.json` 機器帳號(僅 health-tracker 權限)+ REST,參考 `../Pet/cloud.js` 的 auth 寫法;API key 有 referrer 白名單,請求帶 `Referer: https://dashsean.web.app/`
- 資料備份:`../Pet/.backup-health-tracker-*.json`(69 事件前全量)
- 改動守則:單檔 vanilla JS,**不重構架構**;動合併/rollover/schema → 同步 `../Pet/cloud.js` 並告知
