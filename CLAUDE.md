# Hontana 本棚

需要密碼才能開的私人閱讀器。書存在公開的 GitHub repo 裡，但每本都加了密。網站透過 GitHub Pages 部署，push 到 main 就會自動更新。

- 回覆、註解和介面文字一律用**繁體中文（臺灣用語）**。
- 不用 build、不用框架、不裝套件。閱讀器只有 `index.html` 一個檔，加密工具只有 `tools.html` 一個檔。

## 檔案結構

```
index.html                 閱讀器：解鎖 → 書庫 → 分頁閱讀
tools.html                 加密上架頁：txt → 加密書檔（在瀏覽器裡跑）
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
- 進度存在 localStorage 的 `hontana.v1` → `prog[書id] = { o 字元位置, pct, p 頁, n 總頁數, sig 排版簽名, at }`。
  - 還原時，如果 `sig` 和總頁數都沒變，直接回到同一頁；有變（換了裝置、字級或旋轉螢幕）就用 `o` 重新找出對應的頁。
- 設定存在 `hontana.v1.set`，包括字級 `fs`、行距 `lh`、字型 `font`（serif／sans）、主題 `theme`（sepia／white／night）。
- 操作方式：點畫面左 30% 往前一頁、點右 30% 往後一頁、點中間叫出選單；也可以左右滑動或用方向鍵翻頁。

- 新書提示 `checkNew()`：每 60 秒、切回前景時（間隔至少 30 秒）讀 `books/index.json`，有書庫還沒載入的檔名就跳頂部提示條 `#upd`。「看看」重抓書庫，「稍後」同一批新書不再跳。

## 測試

```bash
python3 -m http.server 8765
```

- 用 Playwright 開頁面實測（Chromium 位於 `/opt/pw-browsers/chromium-*/chrome-linux/chrome`）。
- 本機沒有 Actions，所以要自己產生一份 `books/index.json` 給閱讀器讀。

## 部署

- 第一次要先到 GitHub repo 的 Settings → Pages，把 Source 設成 **GitHub Actions**。
- 上版前要先跟使用者確認。
