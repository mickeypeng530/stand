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

**2026-07-17|_pendingSnackLevelCheck 旗標棄用(桌寵端)**
stand 用 `delete` 清旗標,但差異推送永遠推不了「刪除」→ 雲端旗標卡死 → 每次開頁轟炸提示。桌寵 rollover 改為就地把零食寵 +1(cap 8),不寫旗標。index 端的旗標機制未動(它本機 set→本機 delete,自身閉環沒事)。
教訓:**任何用 delete 清理、又會被推上雲的欄位都是地雷**(→ findings #3)。狀態:active。
