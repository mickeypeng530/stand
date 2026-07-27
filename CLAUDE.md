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

- 2026-07-18 **review findings 12 條全修完** + B 批次結帳 + **C 方案**(寵物單調計數對)+ **LWW 時鐘校正**。同步邏輯的已知結構性問題全數收斂;桌寵 `cloud.js` 已同步(Pet repo `c40d790`/`deb0b38`)。**部署後注意:全裝置 Ctrl+F5 + 重啟桌寵**(混版窗口內舊 code 可能用殘留 petDays 多種一隻)。
- 2026-07-17 「69 水滴滴」事件已修(3 commits,見 DECISION_LOG)。資料已清理;備份在 `../Pet/.backup-health-tracker-*.json`。

## 3. 架構速覽(同步機制 = 一切 bug 的核心)

**Schema v2.5**(`_schemaVersion:2`),節點下命名空間:
```
health-tracker/{uid}/
  state/*      今日計數/session/lastDate/updatedAt … LWW(cloudUA > localUA 才整批採用)
  pets/*       合併規則按欄位語意分:
               - 生涯單調計數(只增不減)→ Math.max 天然正確:
                 *GoalDays/*Graduated/sitGoalRevoked(C 方案,寵物)、
                 *HarvestCount/*PlantedCount(B 結帳,種子)
               - stage 欄位(PET_LWW_KEYS:*StageIdx/*StageXP)→ LWW(歸零可傳播,#8)
               - species 字串 → LWW('original'=未孵化蛋;孵化必寫明確種)
               - *PetDays = 凍結遺物,僅遷移種子時讀(seedPetCounters)
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
- **lastDate 格式** `new Date().toDateString()`;rollover(三端都做):歸檔 history、重置 today*、算 streak、零食寵 GoalDays+1。
- 寵物(C 方案):目標達成 → `*GoalDays+1`(三端,無 cap);其餘全衍生 —— `effective=GoalDays−Revoked`、本輪=`%8`、待畢業=`floor(/8)−Graduated`。孵化 `>=3 && species==='original'`(index);畢業由 `settlePetType` 種下+`Graduated+1`(index,進 runSettlement 批次)。**沒有歸零、沒有 cap、沒有同日 guard** —— 單調計數對數學即 dedupe。
- **LWW 時鐘**:updatedAt/editedAt/墓碑一律 `cloudNow()`(= `Date.now()+serverTimeOffset`);桌寵用 `.sv` sentinel。garden `at` 維持本機鐘(identity 用途)。
- **批次結帳(B)**:mini 完成種子只 `*HarvestCount+1`;index 在「auth 後成功 pull 完」「listener merge 後」「visibilitychange 補拉成功後」跑 `runSettlement()` —— 種下 `Harvest−Planted` 差額(cap 50/次)+ 寵物 ≥8 消化,合出**單一**摘要慶祝(`_celebrateBatch` 緩衝 celebrate/beep)。結算**絕不在 pull 失敗或 auth 前**跑(stale 計數會重複種)。

## 4. 常見坑(root-cause 家族)

- **🔴 Math.max 合併無法表達「歸零/遞減」**(歷史教訓,家族已全數收斂):曾爆 69 水滴滴、種子 XP(#8→LWW)、歷史下修(#6→editedAt 全勝)、undo(#7→Revoked 計數)、petDays 每日免費畢業未爆彈(→C 方案)。**正確 idiom,新欄位設計時照選**:會 reset 的量 → LWW;「做過幾次 vs 消化幾次」→ 單調計數對相減(B 收成、C 寵物)。**永遠不要再造「會歸零又走 Math.max」的欄位。**
- **🔴 `delete S[key]` / 歸零旗標推不上雲**:cloudSave 只遍歷存在的 key。已爆:`_pendingSnackLevelCheck` 轟炸;刪除歷史日(#3,已用 tombstone+editedAt 修)。**任何「消化後清掉」的旗標都是地雷** —— 用單調計數對或 tombstone,不要用會清除的欄位。
- **mini 是功能子集**:index 在 rollover/goal 之外做的事 mini 沒有 → mini 先 rollover 就漏。已補:streak+零食寵(#4)、種子收成記帳(#9→B)。新增 rollover/goal 副作用時記得三端(index/mini/桌寵)一起看。
- **精確門檻**:孵化用 `===3` —— 計數跳過去就永遠不觸發(畢業已改 `>=8`,孵化還是 `===`)。
- **桌寵是第三寫入端**:schema/rollover/合併規則任何變動,**必須同步改 `../Pet/cloud.js`**(它複刻 mini 的 rollover 與 goal 邏輯;不碰 stage/種子/garden/deletedDays,寵物只 GoalDays+1)。
- **garden 的 `at` 是 identity**:union by at + 雲端路徑 key。同 ms 多筆要用遞增序號(種子結算與寵物結算都已處理),否則被靜默去重。
- **LWW 時間戳一律用 `cloudNow()`**(桌寵用 `.sv`),別再用裸 `Date.now()` 寫 updatedAt/editedAt/墓碑 —— 裝置時鐘漂移會穩贏 LWW,症狀像隨機回檔。

## 5. Code review findings(2026-07-17 開列;**全數已修**)

| # | 狀態 | 問題 → 修法 |
|---|---|---|
| 3 | ✅ `4ab383a` | 刪除歷史日復活 → deletedDays tombstone + editedAt + null 推送 |
| 4 | ✅ `ae8ea14` | mini rollover 缺 streak/零食寵 → 比照 index/桌寵補上 |
| 5 | ✅ `0dafa91` | index cloudPull 裸 get() 無 timeout → getWithTimeout + 結算 gate 在 pull 成功 |
| 6 | ✅ `0325175` | 歷史數字改小被推回 → mergeDayMax 改「editedAt 新者整筆全勝,同戳才逐欄 max」;rollover 內聯合併同步(編輯過的 entry 不被自動歸檔資料 max) |
| 8 | ✅ `1df720f` | 種子 stage Math.max 復活 → PET_LWW_KEYS 改 LWW |
| 9 | ✅ `b429f77` | mini 收成丟失 → 單調計數對批次結帳(**未用**原提案的 _pendingHarvest 旗標,旗標會復活) |
| 10 | ✅ `953b701` | init 期歸檔不完整花園 → `hadCloudLogin()` 旗標:曾登入跳過 init 歸檔,交給 pull 後 definitive check;登出分支就地補跑 |
| 11 | ✅ `29944a3` | mini 跨夜直接按記到昨天 → `rolloverGuard()` 掛 5 個動作進入點 |
| 12 | ✅ `d237aad` | 跨夜後 UI 停 session 模式 → rolled 分支補 `applySessionUI()`(含 stopTick) |
| 7 | ✅ `87793c6` | undo 的 petDays 遞減被推回 → C 方案 sitGoalRevoked 單調撤銷計數,跨畢業邊界 clamp 0 |

**2026-07-18 兩項架構級收尾(同日完成)**:
- **C 方案**(`87793c6` + Pet `c40d790`):petDays → GoalDays/Graduated 單調計數對,衍生本輪/待畢業;cap 8、同日 guard、`===3` 孵化門檻全拆;「mini 卡 8 每日免費畢業」未爆彈經測試重放確認消滅;遷移 seedPetCounters 亂序安全,舊欄位凍結不刪。
- **LWW 時鐘校正**(`e0a682c` + Pet `deb0b38`):updatedAt/editedAt/墓碑改 `cloudNow()`(serverTimeOffset);桌寵 `.sv` sentinel。三端時間基準一致。

## 6. 接手者 cheatsheet

```powershell
cd C:\Users\彭嗣翔\Claude_Work\stand
# 部署 = push main → GitHub Pages 自動(1-2 分鐘);各裝置要強制刷新(Ctrl+F5)
git push origin main
```
- 雲端資料檢視:用 `../Pet/secrets.json` 機器帳號 + REST,參考 `../Pet/cloud.js` 的 auth 寫法;API key 有 referrer 白名單,請求帶 `Referer: https://dashsean.web.app/`
  - ⚠ 該機器帳號(`2ct8XD…`)的實際權限**不只 health-tracker**:還有 `scheduler` 整棵(含 OPD caselist PHI)+ `hub/todos` 唯讀。憑證明文躺在每台裝 DeskPet 的 PC → 那台被拿到 = 這些全可讀。權限清單以 `../whole/database.rules.json` 為準
  - 2026-07-27 起 `health-tracker` / `health-tracker-backups` 的 `$uid` 已從 `=== auth.uid` 改成硬編 owner UID(原本任何登入者都能自建子樹寫任意資料)。代理讀寫 owner 資料的條款保留 → 機器帳號不受影響;但**用非 owner 的 Google 帳號登入 stand 會被拒**(index/mini 已補 `notePermDenied` 明示提醒)
- 資料備份:`../Pet/.backup-health-tracker-*.json`(69 事件前全量)
- 結帳查帳:index console `s=JSON.parse(localStorage.health_v6);({種子H:s.sitHarvestCount,種子P:s.sitPlantedCount,寵G:s.sitGoalDays,寵畢:s.sitGraduated,撤:s.sitGoalRevoked})` — 穩態:種子差值 0;寵物 `floor((G−撤)/8)===畢`
- sync 事件:localStorage `sync_log_v1`(PUSH/PULL/MERGE/SETTLE/TOMBSTONE_PUSH/ROLLOVER…)
- 改動守則:單檔 vanilla JS,**不重構架構**;動合併/rollover/schema → 同步 `../Pet/cloud.js` 並告知
