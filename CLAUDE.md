# Hontana 本棚

需要密碼才能開的私人閱讀器。書存在公開的 GitHub repo 裡，但每本都加了密。網站透過 GitHub Pages 部署，push 到 main 就會自動更新。

- 回覆、註解和介面文字一律用**繁體中文（臺灣用語）**。
- 不用 build、不用框架、不裝套件。閱讀器只有 `index.html` 一個檔，加密工具只有 `tools.html` 一個檔。

## 檔案結構

```
index.html                 閱讀器：解鎖 → 書庫 → 分頁閱讀
tools.html                 加密上架頁：txt → 加密書檔（在瀏覽器裡跑）
favicon.ico／favicon.svg／apple-touch-icon.png／icon-192.png／icon-512.png  網站 icon（書架上四本書），manifest.webmanifest 給加到主畫面用
words-jp.json              中文詞 → 日文 N5／N4 單字對照表，來源見 THIRD_PARTY.md
books/b-xxxxxxxxxx.json    加密書檔，檔名隨機，看不出書名
books/index.json           書庫清單，由 GitHub Actions 自動產生（只有檔名），不要提交
.github/workflows/pages.yml  先產生 books/index.json，再部署 Pages
```

## 加密

- 主金鑰：`PBKDF2-SHA256(密碼.normalize('NFC'), salt='hontana.library.v1', 600000 次)`，長度 256 bit。
- 每本書的金鑰：`HKDF-SHA256(主金鑰, salt=書檔的 s, info='hontana-book')`，演算法為 AES-GCM-256。
- 書檔格式：`{ v:1, z:1|0, s:salt(16B), iv:(12B), ct:密文 }`，欄位值都是 base64。`z:1` 表示明文加密前先做過 gzip（使用 CompressionStream）。
- 明文是 JSON：`{ title, author, text, added }`。
- `index.html` 和 `tools.html` 各自有一份相同的加密程式碼。**參數改了，兩個檔都要一起改**，不然舊書會打不開。
- 密碼錯誤時 AES-GCM 會驗證失敗。所以解鎖的判斷方式是：只要有任何一本書解得開，密碼就算正確。
- 「記住密碼」不是存密碼本身，而是把推導出的主金鑰以 base64 存在 localStorage 的 `hontana.key`。按登出就會刪掉。

## 閱讀與進度

- 排版用 CSS 多欄：`#flow` 設定 `column-width` 等於版面寬，一欄就是一頁，換頁時用 `translateX(-頁 × stride)` 平移。
- 每一行都會變成 `<p data-o=字元位置>`，章節標題（符合 `CHAP_RE` 且長度在 40 字以內的行）會變成 `<h2>`。
- 強制換頁：整行只有 `==nextpage==`（符合 `PB_RE`，不分大小寫、前後空白不計）會變成 `<div class="pb">`（`break-before: column`），這行本身不顯示，緊接的空行也不補空白。兩頁並排時是換到下一欄，不是下一組兩頁。
- 文字標記（`renderLine` 處理，`FMT_RE`）：`==font-1==…==/font-1==` 字級四級（.f1 1.6em／.f2 1.3em／.f3 原本／.f4 .85em）、`==bold==`、`==italic==`、`==underline==`（實線，和日文單字的虛線分開）、`==color-red==…==/color==`（red／orange／yellow／green／blue／purple／gray 走 `--c-*` 變數，夜間主題有亮色版；也可寫 `#rgb`／`#rrggbb`，夜間不會調整）。
  - 結尾寫 `==/種類==`，`==/font==`、`==/color==` 也算。可以疊用、可以跨行（`fmt` 堆疊跨行沿用，每行頭尾重開重關 span）；只有標記的行不佔一行；遇到章節標題自動全部結束。沒開過的結尾直接隱藏；不認得的標記照原文顯示。
  - 判斷空行、章節標題都用去掉標記後的文字。標記中間的文字才會送進 `markJP`，所以被標記切開的詞不會標日文單字。
- 進度存在 localStorage 的 `hontana.v1` → `prog[書id] = { o 字元位置, pct, p 頁, n 總頁數, sig 排版簽名, at }`。
  - 還原時，如果 `sig` 和總頁數都沒變，直接回到同一頁；有變（換了裝置、字級或旋轉螢幕）就用 `o` 重新找出對應的頁。
- 設定存在 `hontana.v1.set`，包括字級 `fs`、行距 `lh`、字型 `font`（serif／sans）、主題 `theme`（sepia／white／night）。
- 操作方式：點畫面左 30% 往前一頁、點右 30% 往後一頁、點中間叫出選單；也可以左右滑動或用方向鍵翻頁。

- 新書提示 `checkNew()`：每 60 秒、切回前景時（間隔至少 30 秒）讀 `books/index.json`，有書庫還沒載入的檔名就跳頂部提示條 `#upd`。「看看」重抓書庫，「稍後」同一批新書不再跳。

- 日文單字標示：讀中文時，把有 N5／N4 日文說法的中文詞標出來（例：新聞→ニュース、報紙→新聞、房間→部屋）。開書時 `buildFlow` 每行經過 `markJP(line, seen)`，用 `words-jp.json` 的中文詞組成正規表示式（長的優先）比對，命中的包成 `<span class="jw" data-k=中文詞>`。點一下 `showWord()` 跳小卡，列出所有對應的日文（寫法、讀音、級別、🔊 用 speechSynthesis ja-JP 唸）。`db.set.jp` 開關，切換時 `rebuild()` 重建內文並保留位置。
  - 同一個中文詞在同一章只標第一次（`seen` 遇到章節標題就清空；沒有章節的書每 `JP_SPAN` 5000 字清空一次），避免常見詞滿頁底線。
  - `words-jp.json` 格式：`{ w: [[日文, 讀音, 級別(4/5)], …], k: { 中文詞: [w 的索引, …] } }`。中文詞都是兩字以上的臺灣用語（「吃」「看」這類單字詞不收，會滿頁都是）。一個中文詞可以對到多個日文（例：學生→学生、生徒）。兩筆「キロ」分別是公斤、公里，不是重複。
- 日文書（反過來標）：`buildFlow` 開頭用 `isJaText()` 看前 2 萬字的假名占比（≥15% 且至少 50 個假名），是日文書就設 `R.ja`，`markJP` 改走 `markJA`：標出書裡 N5／N4 的日文詞，`<span class="jw" data-w="w 索引,…">`，點了小卡顯示原形、讀音、級別、🔊 和中文意思（`JP.zh`：由 `k` 反查）。開關、每章只標一次的規則和中文書共用，`seen` 用 w 索引當 key，所以不同活用算同一個詞。
  - `jaIndex()` 把詞表展開成比對表（開頭字 → 寫法清單，長的優先），約 8 千種寫法。`jaForms()` 展開活用：一段／五段看讀音判斷（`JA_GODAN_RU` 是例外）、い形容詞、〜する；`JA_NOT_V`／`JA_NOT_A` 是長得像動詞、形容詞但不是的詞。
  - 寫法剛好是某個詞的辭書形時只對到那個詞（「知らせる」不算「知る」的活用）。「行う」不展開「行っ〜」，因為「行った」幾乎都是不在詞表裡的「行く」。
  - 寫法補充：`JA_FIX`（詞表的「終る」「曲る」「落る」「楽む」「落す」「下る」→ 一般寫法，小卡也顯示一般寫法）、含「御」的詞加「ご」寫法、`JA_KANA` 裡常寫成假名的詞加讀音寫法。
  - 誤判防護（`jaOk`）：單一漢字的詞前後不能接漢字（「以上」的「上」不標）；片假名詞前後不能接片假名（「バスケット」的「バス」不標）；兩字以下的平假名詞（もう、まだ、でも…）整個不標。
  - 「降ります」（降る／降りる）、「通って」（通る／通う）、「やさしい」這種本來就有歧義的，小卡兩個都列。
  - 章節偵測 `CHAP_RE` 另外加了「プロローグ」「エピローグ」「あとがき」。

## 測試

```bash
python3 -m http.server 8765
```

- 用 Playwright 開頁面實測（Chromium 位於 `/opt/pw-browsers/chromium-*/chrome-linux/chrome`）。
- 本機沒有 Actions，所以要自己產生一份 `books/index.json` 給閱讀器讀。

## 部署

- 第一次要先到 GitHub repo 的 Settings → Pages，把 Source 設成 **GitHub Actions**。
- 上版前要先跟使用者確認。
