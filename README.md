# FFXIV 繁中服排行榜資料快照

此 repo 只保存 [Final-Fantasy-XIV-Ranking-for-TC](https://github.com/Kantai235/Final-Fantasy-XIV-Ranking-for-TC) 資料管線產生的最新可追溯快照。

- `data/`：FFLogs report、fight、player、掃描狀態與歷史游標等權威來源。
- `public/data/`：主站共用的靜態 JSON；個別玩家成績與明細由 Users repo 承載。
- `snapshot-manifest.json`：本快照的檔案清單、大小與 SHA-256。

每次更新都建立沒有 parent 的 root commit，並以 `force-with-lease` 更新 `main`。舊快照不屬於可追溯歷史；歷史來源由目前快照內的 append-only report 與 state 保存。

請勿直接人工修改資料。所有寫入都必須先通過主專案的資料契約與守恆驗證。
