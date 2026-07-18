# DECISION_LOG — stand

> append-only。每筆:日期+決策 / 沒選的方案與原因 / 代價 / 狀態。

---

**2026-07-17|「69 水滴滴」事件:選逐點補丁,不做合併層重構**
桌寵接入後 user 久違開 index,花園被重複灌入水寵成體(峰值 232 筆)。根因鏈:mini 無上限累加 `pets/waterPetDays`(到 12)→ index auto-recovery 畢業無防重入 → 歸零被 Math.max 合併復活 → 每次開頁/合併再畢業。
**修法(3 commits)**:①checkPetLevelUp 同日防重入 ②auto-recovery/listener 消化後立即 saveLocal+cloudSave ③listener 消化移到 garden merge 之後 ④mini 計數 cap 8。資料清理:花園復原 81 筆、計數歸零(備份 `../Pet/.backup-health-tracker-*.json`)。
**沒選**:把 pets/stage 欄位改 LWW/tombstone(根治 Math.max 家族)—— 影響面大,留給專門 session(見 CLAUDE.md §5 findings)。
代價:同族 bug(#3 #6 #7 #8)仍在。狀態:active。

**2026-07-17|桌寵(第三寫入端)定位 = mini 同級記錄器**
桌寵 `../Pet/cloud.js` 一度自行做孵化/畢業 —— 在 Math.max 世界裡歸零會被復活,同樣會刷花園。改為:只 +1 計數(cap 8)、不碰 garden、升級一律交給 index。
**沒選**:桌寵完整複刻 index 升級邏輯 —— 三端各自升級 = 三倍競態面。
代價:只用桌寵+mini 的日子,孵化/畢業慶祝延到下次開 index。狀態:active。

**2026-07-18|pets 合併規則按欄位語意拆分:stage→LWW,petDays 留 Math.max(#8)**
種子 stage(idx/XP)是 set-then-reset 語意,Math.max 讓歸零復活(69 同機制、種子一天可完成多次故同日 guard 無效)→ 改 LWW(`PET_LWW_KEYS`)。
**沒選**:pets 全改 LWW —— petDays 是 increment 語意,LWW 會把「已被同日 guard 罩住的罕見復活」換成「沒有 guard 的 lost-update」,負優化。**沒選**:generation-tag(gen 大者全勝)—— 最正確但動 schema + 桌寵,個人 app 過度設計。
代價:petDays 歸零復活仍在(guard 限一天一次)。狀態:active。

**2026-07-18|開頁批次結帳:單調計數對,不用 pending 旗標(#9 收編)**
mini 完成種子只歸零不種花,收成靜默丟失(約 2-4 天一棵)且多事件慶祝互蓋。改:`*HarvestCount`(完成數)/`*PlantedCount`(已種數)兩個只增不減欄位,待結帳=差值,index 在 auth 後/listener 後結算+單一摘要。
**沒選**:findings #9 原提案的 `_pendingHarvest` 旗標 —— 「消化後清掉」= 歸零 = Math.max 復活 = 無限重種,和 69 同機制。**連帶**:移除 init 期(auth 前、未合併)的舊 auto-recovery —— stale 計數結算會重複種;測試抓到 per-type `now()` 撞號被 mergeGarden 靜默去重 → `at` 改跨類型遞增序號。
代價:歷史已丟收成救不回(從 0/0 起算);兩 index 分頁同時結算有亞秒重複窗。狀態:active。

**2026-07-18|#5 提前修(getWithTimeout)+ 結算 gate 在 pull 成功**
B 把結算掛在 `await cloudPull(true)` 之後 → 裸 get() 卡死從「同步死」升級成「listener+結算全死」,#5 從待辦變必修。cloudPull 回傳成敗,結算只在成功後跑(失敗時用 stale 計數會重複種);visibilitychange 補拉成功也結算(同裝置 backlog 時 listener 會因 cloudUA<=localUA 早退)。狀態:active。

**2026-07-18|刪除歷史日:tombstone + editedAt 排序(#3)**
`deletedDays/{day}=ts` 只增不減 → union/max 天然正確;死亡判定 `deletedAt > entry.editedAt`,重新編輯蓋新戳即復活 —— 不需要「移除 tombstone」(那會把 delete-不-propagate 問題遞迴上移一層)。
**沒選**:tombstone 放 settings(整批 LWW 會互相蓋)、只推 null 不留 tombstone(stale 裝置 SANITY_PUSH 推回就復活)。`editedAt` 同時為 #6 鋪路(mergeDayMax 已傳遞戳,差「新者全勝」規則)。狀態:active。

**2026-07-18|#6/#10/#11/#12 收尾批 + 發現 petDays 未爆彈**
#6:mergeDayMax 改「editedAt 新者整筆全勝、同戳才逐欄 max」—— 沒選逐欄 LWW(粒度過細無需求);rollover 內聯合併對已編輯 entry 整筆保留(手動編輯 > 自動歸檔)。#10:init 歸檔用 `hadCloudLogin()` localStorage 旗標當「登入將至」的同步 proxy —— 沒選等 auth resolve 再歸檔(要重排 init 流程,面大)。#11:mini 五個動作進入點掛 rolloverGuard。#12:一行 applySessionUI。
**未爆彈(多日尺度推演發現)**:寵物畢業歸零後 mini Math.max 卡 8 → 次日起每日免費畢業一隻;桌寵免疫(無本機狀態)。7/17 清零 → 首爆約 7/25。選項 A(petDays 入 LWW 止血)vs B(直接做 C),**待 user 決定**。狀態:active。

**2026-07-17|_pendingSnackLevelCheck 旗標棄用(桌寵端)**
stand 用 `delete` 清旗標,但差異推送永遠推不了「刪除」→ 雲端旗標卡死 → 每次開頁轟炸提示。桌寵 rollover 改為就地把零食寵 +1(cap 8),不寫旗標。index 端的旗標機制未動(它本機 set→本機 delete,自身閉環沒事)。
教訓:**任何用 delete 清理、又會被推上雲的欄位都是地雷**(→ findings #3)。狀態:active。
