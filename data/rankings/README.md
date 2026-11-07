# 排行榜完整資料格式

`*.json` 主檔保留副本資訊、更新時間、`ranking_entries`，以及 `report_shards` 分片清單。

完成排名建置所需的 report/fight/player 脈絡保存在同名的 `*.reports/*.json` 分片目錄中。分片是為了避免單一檔案超過 GitHub 100 MB 上限；分析時請透過 `scripts/fetch_fflogs.py` 的 `讀取排行榜檔案()` 讀取，會自動合併主檔與所有分片。

為控制 repo 容量，新的 report 不再保存 `fflogs_raw`、`master_data` 與 `matched_players`。這些欄位屬於可由 FFLogs API 依 report code 重查的原始資料；本資料夾只保留前端建置與資料追溯必要欄位，例如 `matched_traditional_chinese_servers`、`fights`、`fights[].players`、傷害時間分母與 `fight_hash`。

既有分片若需要移除這些大型 raw 欄位，請先執行 `npm run compact:rankings -- --dry-run` 檢查預估異動，再執行 `npm run compact:rankings`。這個清理流程只壓縮欄位與重分片，不刪除 report、fight 或 player 紀錄。

## 欄位契約

- `schema_version`：目前為 `1`，供未來 schema 演進判斷。
- `encounter`：副本摘要，包含 `key`、`name`、`category`、`zone_id`、`encounter_id`、`difficulty`。
- `ranking_entries`：前端與 `build_user_data.mjs` 優先使用的扁平排行榜列。每位角色在同一副本、同一職業只保留最佳成績，排序規則為 rDPS、通關時間、aDPS。
- `report_shards`：完整 report 分片相對路徑。若存在，主檔通常不直接保存 `reports`。
- `reports`：report 索引，舊格式或讀取分片後才會出現。每個 report 會保留 `matched_traditional_chinese_servers`、`fights` 與完成排行榜建置所需的玩家、時間、傷害統計欄位；舊資料可能仍含 `fflogs_raw`、`master_data` 或 `matched_players`，但新資料不再寫入這些大型 raw 欄位。
- `fights[].data_integrity`：暫時性的 fight 層資料品質檢核結果。它保存規則版本、檢核時間、狀態、原因與必要的「全隊敵方承傷／敵方最大生命池」倍率；M5S～M8S 的 v11 結果另保存逐 NPC GUID 的固定生命值、等效承傷實例數與轉場比例比對證據。`hidden_from_public=true` 代表此 fight 不會出現在排行榜、個人成績、隊伍榜與近期動態，但原始 report/fight/player 一律保留。這不是 `report_hidden`，同一 report 內其他正常 fight 不受影響。
- `fights[].support_metrics_summary`：新收錄 fight 的支援統計計算版本、減傷規則版本、坦／補人數與原始事件筆數摘要。`raw_events_persisted=false` 表示事件只在抓取當下用於計算，沒有寫入 Git。
- `fights[].players[].healing_stats`：補師的治療摘要，包含 `hps`、`pure_healing`、`protection`、`overheal` 與 `overheal_percent`。`hps` 使用 FFLogs Healing table 的 `combatTime`；`pure_healing=totalReduced`，`protection=max(total-totalReduced, 0)`，OH% 則以 `overheal / (pure_healing + overheal)` 計算，護盾不進 OH% 分母。
- `fights[].players[].tank_stats`：坦克的 `damage_taken`、`absorbed_damage`、`unmitigated_damage`、`self_healing`、`personal_protection`、`team_protection` 與 `mitigation_coverage`。承傷只加總 DamageTaken events 的 `type=damage`，不得把同一次命中的 `calculateddamage` 再算一次；個人／團隊防護是 Healing table 中實際被消耗的護盾量，不是理論盾值。
- `tank_stats.mitigation_coverage`：依全副本共用、版本化的玩家 Status ID 規則，把坦克施放的個人 Buff、隊友／團隊 Buff 與敵方降傷 Debuff 還原成時窗。只有實際傷害事件落在時窗內的 activation 才算有效；`effective_activation_percent` 是有效時窗數占全部時窗數，`personal`／`team` 另保存不重複加總的 `covered_unmitigated_damage` 與傷害覆蓋率。這不是「整場有 Buff 的時間百分比」，也不代表單一技能實際減免量，因為多層減傷重疊時 FFLogs 無法把 `mitigated` 精確歸因給其中一招。

上述欄位都是選填的向後相容擴充；舊 fight 沒有支援統計時仍可正常重建。現階段它們保存在來源 report 分片與來源 `ranking_entries`，尚未加入公開排行榜 allowlist。同場另一補不會在 Python 抓取層複製成 UI 專用欄位；未來由 Node.js 建置層在同一 fight 的兩名補師之間建立關聯。

若同名角色有跨伺服器公開紀錄，公開 `ranking_entries` 會以「角色名稱 + 伺服器 + 職業」拆成不同排行榜身分。遊戲允許不同伺服器使用相同角色名稱，因此公開資料不再自動依最新公開紀錄所在伺服器合併，也不再輸出 `original_server` 作為轉服追溯欄位。

## 去重與追溯

- `fight_hash` 用通關時間、傷害時間與全隊玩家成績辨識同一場戰鬥，降低多名隊員重複上傳造成的重複計算。
- `source_reports` 與 `duplicate_count` 會保留重複來源數，讓前端顯示去重後排名時仍可追查資料來源。
- `data/rankings/*.json` 與 `*.reports/*.json` 是 append-only 歷史資產；合併或同步時不可用刪檔或覆蓋方式處理。
