# CLAUDE.md — stand(久坐/喝水追蹤 + 花園養成)

> 最後更新:2026-07-18。決策史在 `DECISION_LOG.md`。

## 1. 這專案在做什麼

彭嗣翔醫師自用的健康追蹤 app:久坐提醒、喝水/零食記錄、花園養成(種子 XP 長植物、目標天數養寵物)。GitHub Pages 部署(push main 自動更新),資料在 Firebase RTDB `case-scheduler-f752b` 的 `health-tracker/{uid}`。

**三個寫入端共用同一節點**:
| 端 | 檔案 | 角色 |
|---|---|---|
| index.html(~5300 行) | 主 app | 完整功能:session/花園/寵物升級/歸檔/批次結帳 |
| mini.html(~950 行) | 輕量記錄器 | 只記數(水/站/零食)+ 記錄種子完成,**無花園邏輯**,user 日常主力 |
| 桌寵 DeskPet | `../Pet/cloud.js` | Electron 桌面寵物,機器帳號(UID `2ct8XD…`)寫入,行為比照 mini |

⚠ **`../OPD/stand.html` 是 6 月前的舊版(3534 行),勿當參考** —— 線上版 v2.5-0602 schema 已大改。

## 2. 現在進度

- 2026-07-18 **review findings 主力批已修**(4 commits:#4 `ae8ea14`、#8 `1df720f`、B 結帳 `b429f77`+#9 收編、#5 `0dafa91`、#3 `4ab383a`):mini rollover 補 streak/零食寵、種子 stage 改 LWW、開頁批次結帳(單調計數對)、cloudPull timeout、刪除歷史日 tombstone。**剩 #6 #7 #10 #11 #12**(見 §5)。
- 2026-07-17 「69 水滴滴」事件已修(3 commits,見 DECISION_LOG)。資料已清理;備份在 `../Pet/.backup-health-tracker-*.json`。

## 3. 架構速覽(同步機制 = 一切 bug 的核心)

**Schema v2.5**(`_schemaVersion:2`),節點下命名空間:
```
health-tracker/{uid}/
  state/*      今日計數/session/lastDate/updatedAt … LWW(cloudUA > localUA 才整批採用)
  pets/*       ⚠ 三種語意混居,合併規則按欄位分:
               - *PetDays 計數 → Math.max(+同日畢業 guard 罩復活)
               - stage 欄位(PET_LWW_KEYS:*StageIdx/*StageXP)→ LWW(歸零可傳播,#8)
               - *HarvestCount/*PlantedCount 單調計數對 → Math.max(天然正確)
               - species 字串 → LWW
  meta/*       totalStands/totalWater/totalSnacks … Math.max(only-grow,安全)
  settings/*   門檻/主題/_gardenPeriodKey …
  history/{day}   per-day union + 欄位 Math.max(mergeDayMax);死日過濾見 deletedDays
  deletedDays/{day}=刪除時間戳   歷史日 tombstone(#3):只增不減、per-key max union;
               死亡判定 deletedAt > entry.editedAt(重新編輯蓋新 editedAt 即合法復活)
  garden/{月key}/{at}   植物/寵物,union by at,永不去重(⚠ at 是 identity,撞號會被靜默去重)
  gardenArchive/{月key}  跨月歸檔快照
  (頂層還殘留 v1 同名欄位 = 遺物,v2 client 不讀)
```
- **同步流程**:`cloudSave()` 差異推送(對 `lastCloudSnap` diff,只推有變的 key;死日推顯式 null)→ 別的裝置 `onValue` listener 收到 → `normalizeCloudSnap()` → 依上表規則合併 → `maybeRollover(S)` → render。index cloudPull 有 12s `getWithTimeout`(#5)。
- **lastDate 格式** `new Date().toDateString()`;rollover(三端都做):歸檔 history、重置 today*、算 streak、零食寵 +1(cap 8)。
- 寵物:目標達成(8站/6杯/零食≤上限)→ `*PetDays+1`(mini/桌寵 cap 8);`===3` 孵化定種、`>=8` 畢業進花園(index 才做;同日防重入 guard)。
- **批次結帳(B)**:mini 完成種子只 `*HarvestCount+1`;index 在「auth 後成功 pull 完」「listener merge 後」「visibilitychange 補拉成功後」跑 `runSettlement()` —— 種下 `Harvest−Planted` 差額(cap 50/次)+ 寵物 ≥8 消化,合出**單一**摘要慶祝(`_celebrateBatch` 緩衝 celebrate/beep)。結算**絕不在 pull 失敗或 auth 前**跑(stale 計數會重複種)。

## 4. 常見坑(root-cause 家族)

- **🔴 Math.max 合併無法表達「歸零/遞減」**:stale client 舊高值會復活歸零後的計數。已爆:寵物畢業(69 水滴滴)、種子 XP(#8,已改 LWW 修)。**還在的同族**:undo 站起(#7)、歷史數字下修(#6);petDays 歸零復活靠同日 guard 罩(改 LWW 會換來 lost-update,故意不改,見 DECISION_LOG)。**正確 idiom**:會 reset 的量 → LWW;「做過幾次 vs 消化幾次」→ 單調計數對相減(B 結帳)。
- **🔴 `delete S[key]` / 歸零旗標推不上雲**:cloudSave 只遍歷存在的 key。已爆:`_pendingSnackLevelCheck` 轟炸;刪除歷史日(#3,已用 tombstone+editedAt 修)。**任何「消化後清掉」的旗標都是地雷** —— 用單調計數對或 tombstone,不要用會清除的欄位。
- **mini 是功能子集**:index 在 rollover/goal 之外做的事 mini 沒有 → mini 先 rollover 就漏。已補:streak+零食寵(#4)、種子收成記帳(#9→B)。新增 rollover/goal 副作用時記得三端(index/mini/桌寵)一起看。
- **精確門檻**:孵化用 `===3` —— 計數跳過去就永遠不觸發(畢業已改 `>=8`,孵化還是 `===`)。
- **桌寵是第三寫入端**:schema/rollover/合併規則任何變動,**必須同步改 `../Pet/cloud.js`**(它複刻 mini 的 rollover 與 goal 邏輯;不碰 stage/種子/garden/deletedDays)。
- **garden 的 `at` 是 identity**:union by at + 雲端路徑 key。同 ms 多筆要用遞增序號(結算已處理),否則被靜默去重。
- **LWW 時鐘 = 裝置牆鐘 `Date.now()`**:桌寵 24h 常駐,時鐘漂移會穩贏 LWW(潛在隱患,未修;修法 = `.info/serverTimeOffset` 校正三端,見 DECISION_LOG)。

## 5. Code review findings(2026-07-17 開列;#1-#5、#8、#9 已修)

| # | 嚴重度 | 狀態 | 問題(位置行號為開列當時) |
|---|---|---|---|
| 3 | 🔴 | ✅ `4ab383a` | 刪除歷史日復活 → deletedDays tombstone + editedAt + null 推送 |
| 4 | 🔴 | ✅ `ae8ea14` | mini rollover 缺 streak/零食寵 → 比照 index/桌寵補上 |
| 5 | 🟡 | ✅ `0dafa91` | index cloudPull 裸 get() 無 timeout → getWithTimeout + 結算 gate 在 pull 成功 |
| 8 | 🟡 | ✅ `1df720f` | 種子 stage Math.max 復活 → PET_LWW_KEYS 改 LWW |
| 9 | 🟡 | ✅ `b429f77` | mini 收成丟失 → 單調計數對批次結帳(**未用**原提案的 _pendingHarvest 旗標,旗標會復活) |
| 6 | 🟡 | ⏳ | 歷史數字改小被 mergeDayMax 推回 → entry 已有 editedAt 戳(#3 鋪路),差 mergeDayMax 改「editedAt 新者全勝」 |
| 7 | 🟡 | ⏳ | undo 站起的 sitPetDays 遞減被 Math.max 推回 |
| 10 | 🟡 | ⏳ | 首次 cloudPull 前就歸檔 → stale 裝置月初先開會歸檔不完整花園(結算已改 auth 後,歸檔的 init 呼叫還在) |
| 11 | 🟢 | ⏳ | mini 掛著跨午夜不 rollover → doWater/handleStandup 開頭補 `if(maybeRollover(S))saveSync()` |
| 12 | 🟢 | ⏳ | index 跨午夜後 UI 停在 session 模式 → rolled 分支補 `applySessionUI()` |

長期方向(未排程):寵物 petDays 也改單調計數對(`GoalDays`/`Graduated`,cap 8 可移除、#7 一併消失);LWW 時鐘 server-offset 校正。

## 6. 接手者 cheatsheet

```powershell
cd C:\Users\彭嗣翔\Claude_Work\stand
# 部署 = push main → GitHub Pages 自動(1-2 分鐘);各裝置要強制刷新(Ctrl+F5)
git push origin main
```
- 雲端資料檢視:用 `../Pet/secrets.json` 機器帳號(僅 health-tracker 權限)+ REST,參考 `../Pet/cloud.js` 的 auth 寫法;API key 有 referrer 白名單,請求帶 `Referer: https://dashsean.web.app/`
- 資料備份:`../Pet/.backup-health-tracker-*.json`(69 事件前全量)
- 結帳查帳:index console `s=JSON.parse(localStorage.health_v6);({H:s.sitHarvestCount,P:s.sitPlantedCount})` — 穩態差值應為 0
- sync 事件:localStorage `sync_log_v1`(PUSH/PULL/MERGE/SETTLE/TOMBSTONE_PUSH/ROLLOVER…)
- 改動守則:單檔 vanilla JS,**不重構架構**;動合併/rollover/schema → 同步 `../Pet/cloud.js` 並告知
