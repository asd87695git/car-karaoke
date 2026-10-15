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
  - 對唱：LRC 行首「男：」「女：」「合：」「(男)」「M:」「F:」等標記會被拿掉，改用藍／粉／紫上色，沿用到下一個標記。
  - 前奏／間奏倒數（`updateCue()`）：前奏、空白行、或同一句停超過 6 秒且離下一句 ≥4 秒時顯示，最後 3 秒變 ●●●。
  - 沒有聲音層面的自動對齊（iframe 跨網域拿不到音訊）。對拍：播放中點某一行歌詞＝「現在唱到這句」，偏移量 = 該行時間 − 播放時間（不扣 LEAD，用來抵人的反應時間）。
- 點歌傳輸：公開的 ntfy.sh 中繼，沒有自己的後端。
  - `<topic>-req`：乘客 → 主機，訊息 `{type:'add', rid, by, items:[{id,title,ch}]}`，主機用 rid 去重（已處理的 rid 存在 localStorage `seenRids`，最多 300 筆）。
    - 主機把最後收到的訊息時間存在 `reqCursor`，重開頁面或重連時用 `since=<時間-1>` 補收（最多回溯 3 小時），避免 iOS 砍掉分頁時漏歌。
  - `<topic>-req` 其他訊息類型（都帶 rid、by）：
    - `{type:'ctl', act:'pause'|'play'|'next'|'restart', vid}`：乘客遙控。next/restart 的 vid 要等於目前播放的歌才執行（避免兩人同時切歌切兩首）。
    - `{type:'fx', fx:'clap'|'cheer'|'drum'|'horn'}`：氣氛音效，主機用 Web Audio 合成播放（`playFx()`），主機要先被點過一下才有聲音（iOS 限制）。
    - `{type:'del', k}`：撤回。點歌項目代碼 `k = rid 前 5 碼 + 批次內序號(36 進位)`，乘客端自己算得出來，存在 `kdesk:myKeys`。
    - ctl／fx 只認 20 秒內的訊息（看 ntfy 的 `time`），重連補收時不會重播舊指令。
  - `<topic>-st`：主機 → 乘客，廣播目前播放與待播清單；乘客用 `since=latest` 取最新狀態。欄位：`now{t,b,a,i,c}`、`q[{t,b,a,k}]`、`n`、`p`（1=播放中）、`y`（跟唱同步）。
    - `y{v,l,o,at,ts,p}`：影片 id、LRCLIB 歌詞 id（null=主機還在找／重試中，0=沒有）、偏移、送出當下播放秒數、送出時間、是否播放。乘客用 `at + (now - ts)` 推算時間，自己去 LRCLIB 抓歌詞。
    - 觸發廣播：換歌、進入播放（含緩衝結束）、暫停、拖進度、重唱、調偏移、歌詞載入完成。第一則至少等 1.5 秒，之後間隔 ≥4 秒合併。
    - 2026-10-16 實測：雙向操作送達約 1.7～2.2 秒，跟唱推算誤差約 4ms（同一台電腦，實機會再加上兩台裝置的時鐘差）。
  - 乘客跟唱：手機上可收合的「跟唱歌詞」，LRCLIB 失敗會隔 2／5／10／20 秒重試（和 iPad 共用熱點 IP 時容易被限流）。主機查歌詞全部失敗時隔 8／20／45 秒重試。
    - ntfy.sh 匿名每 IP 每天上限 250 則：廣播會合併、間隔至少 4 秒，失敗時退避重試（429 退 30 秒起、最多 5 分鐘）。
  - 單則訊息需小於約 4KB（中文字 UTF-8 佔 3 bytes），程式有分批與截斷；每批最多 `MAX_BATCH`（25）首，乘客端和主機端共用。
  - `ntfySub()` 在回到前景（背景超過 20 秒會強制重連）、網路恢復、連線變 CLOSED 時會自動重新訂閱。
  - 點歌碼格式 `kdesk-` + 12 碼隨機小寫英數。
- 搜尋：YouTube Data API v3 `search.list`（每次扣 100 單位，每天預設約 100 次），一次顯示 5 筆。沒有金鑰時只能貼連結；貼連結用 noembed.com 取得影片標題。
- 推薦：YouTube 已停用 `relatedToVideoId`，改成 `guessArtist()` 從標題／頻道猜歌手，再用歌手名搜尋（`ytRecommend()`，同歌手結果在記憶體暫存）。
  - 「你可能喜歡」是按鈕，按了才搜（多扣 100），刻意不自動載入以省額度。
  - 主機「歌單播完自動播推薦」（設定可關，`kdesk:autoRec`）：從最近 5 首有人點的歌隨機挑歌手，避開最近 60 首播過的（`kdesk:played`），一次搜尋的結果當作電台緩衝連續播；沒金鑰時從我的最愛隨機挑。推薦歌標記 `auto: true`，顯示「自動推薦」。
- 唱過／點過：主機記播放過的歌（不含自動推薦）在 `kdesk:histPlayed`，乘客記自己點過的在 `kdesk:histSent`，最多 300 首，點歌元件第三個分頁可切最近／最常。
- 主機操作：重唱、播放／暫停、下一首、進度條（可拖曳），四顆音效鈕。
- 本機測試注意：`127.0.0.1` 開的頁面 YouTube 會回 150 拒絕嵌入，播放相關要在 GitHub Pages 上測。
- 乘客點歌的小 QR Code 與連結收在「設定」裡；斷線時「設定」標題旁會出現紅字提示。沒歌時歌詞區還是會顯示大 QR Code。
- 版面：橫向時操作區（播放器、待播、點歌）在左邊，方便駕駛操作；直向時上下排。
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
