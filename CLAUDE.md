# 車上點歌台（car-karaoke）

單一檔案 `index.html` 的網頁 App：在 Tesla 車上用 iPad 當 K 歌提詞機，乘客用手機掃 QR Code 點歌。

## 使用情境
- iPad 用 Safari 開這個網頁，透過藍牙把聲音送到 Tesla 喇叭。
- 為什麼不用 YouTube Music App：iOS 不讓第三方 App 讀取 YouTube Music 的播放進度，所以改用網頁內嵌 YouTube 播放器（IFrame API），自己掌握播放時間來同步歌詞。
- 部署在 GitHub Pages（需要 https 網址；YouTube 內嵌需要正常的 Referer，不能用 file:// 開）。

## 架構
- 同一個檔案兩種模式：網址沒有 hash → 主機（iPad）；`#g=<點歌碼>&k=<API 金鑰>` → 乘客點歌頁。
- 播放：YouTube IFrame API，`onError` 遇到禁止嵌入的影片自動跳過。
- 歌詞：LRCLIB（`lrclib.net/api/search`、`/api/get/{id}`）抓 LRC 同步歌詞。`queriesFor()` 會把 YouTube 標題清理成多組查詢（處理【】《》[] 括號、Official MV 等雜字、`- Topic` 頻道），再挑長度最接近影片的版本。可手動調時間偏移（每首記住）、手動換歌詞版本（每首記住）。
- 點歌傳輸：公開的 ntfy.sh 中繼，沒有自己的後端。
  - `<topic>-req`：乘客 → 主機，訊息 `{type:'add', rid, by, items:[{id,title,ch}]}`，主機用 rid 去重（已處理的 rid 存在 localStorage `seenRids`，最多 300 筆）。
    - 主機把最後收到的訊息時間存在 `reqCursor`，重開頁面或重連時用 `since=<時間-1>` 補收（最多回溯 3 小時），避免 iOS 砍掉分頁時漏歌。
  - `<topic>-st`：主機 → 乘客，廣播目前播放與待播清單；乘客用 `since=latest` 取最新狀態。
    - ntfy.sh 匿名每 IP 每天上限 250 則：廣播會合併、間隔至少 4 秒，失敗時退避重試（429 退 30 秒起、最多 5 分鐘）。
  - 單則訊息需小於約 4KB（中文字 UTF-8 佔 3 bytes），程式有分批與截斷；每批最多 `MAX_BATCH`（25）首，乘客端和主機端共用。
  - `ntfySub()` 在回到前景（背景超過 20 秒會強制重連）、網路恢復、連線變 CLOSED 時會自動重新訂閱。
  - 點歌碼格式 `kdesk-` + 12 碼隨機小寫英數。
- 搜尋：YouTube Data API v3 `search.list`（每次扣 100 單位，每天預設約 100 次）。沒有金鑰時只能貼連結；貼連結用 noembed.com 取得影片標題。
- 我的最愛：存在各自裝置的 localStorage（key 前綴 `kdesk:`），可勾選多首點播、隨機點 5 首、刪除、備份／匯入（JSON 或多個連結）。
- 其他：Screen Wake Lock 保持螢幕常亮；QR Code 用 cdnjs 的 qrcodejs 1.0.0。

## 已知限制 / 隱私
- API 金鑰存在 iPad 的 localStorage，但會放進 QR Code 連結給乘客搜尋用；要在 Google Cloud 限制網址（HTTP referrer）與 API 範圍。
- ntfy.sh 是公開服務，知道點歌碼就能讀寫；可在設定「換一組點歌碼」。
- Safari 對沒常開的網站可能在 7 天後清 localStorage；建議加入主畫面並定期備份最愛。

## 尚未完成 / 可能的下一步
- 還沒在實機 iPad + Tesla 上測試過。
- 可考慮改成「乘客送關鍵字、由 iPad 代為搜尋」，讓金鑰不離開 iPad。
- 介面風格：單一深色主題（夜間開車減少眩光），琥珀色強調色，字型 Noto Sans TC / IBM Plex Mono。
