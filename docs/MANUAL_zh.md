# Optiview — 部署與維護說明書

版本 **5.4** · `linux/amd64` · 離線檔 37.8 MB · 弱點掃描 CRITICAL 0 / HIGH 0 / MEDIUM 0

這一份是**在公司照著做**用的。從一台空的 Linux 主機到畫面上有事件，照順序做完就好。
不需要讀程式碼，也不需要裝 Python 或 Node。

系統名稱可以自己改（`.env` 的 `APP_NAME`），預設是 Optiview。

> **5.0 是第一個正式版。** 4.x 是做給你試用的迭代；5.0 之後修問題走 5.0.x、
> 加功能走 5.x，說明書會跟著同一個號碼走。

<details>
<summary>5.0 有什麼</summary>

**定位**
- 匯入光纖圖與設備清單 → 向網管撈出邏輯連線 → 收 log → 斷纖時指出最可能的那一條。
- 「斷點如何影響」：一對 peer 一張卡片，點開看**替代路徑**（不經過斷點的實體
  路徑，短的排前面）。沒有替代路徑的會明講 —— 派工優先序從那裡看。
- 現場地圖是樹狀的：樞紐在最上面，往下一層就是往外一跳，同一層保證不重疊。
  點一台設備，跟它斷線的鄰居亮紅圈、還通著的鄰居亮灰圈，線上標「共 N 條 · 斷 M 條」。

**排查**
- **系統日誌**：畫面上直接看，每一筆都帶哪個程式、哪一段、第幾行。錯誤另外
  留一份，清日誌不會把它一起清掉。設定精靈的畫面上也有入口 —— 匯入或撈取
  失敗時最需要它。
- **原始 log 可以匯出 CSV**（含搜尋條件），拿回自己的機器怎麼分析都行。
- 網管撈不到時不會清空既有資料，畫面與日誌都會說「保留上一次成功的結果」。

**保存**
- log 一天一份、每天固定時間清掉過期的（預設留 7 天）。
- Event 永久保存，而且**自帶當時的 log** —— log 過期之後回頭看那件事，
  畫面會標「這是當時留下的紀錄」。

**這一版刻意沒有做的事**
- 不畫「這條邏輯連線實際走哪幾段光纖」。網管只說兩端相連，不說中間怎麼繞 ——
  那份資料在真實部署裡不存在，畫出來的只會是猜測。
- 只處理單一斷點。同時斷兩條以上不在這一版的範圍內。

**5.0.1 修了什麼**
- **舊 Event 的細節頁現在完全靠事件自己那份 log**。原始 log 不在了的時候，
  「斷點如何影響」會顯示「切斷了 0 對設備」—— 它去讀了已經清空的即時緩衝區。
  現在四個分頁（現場、影響、替代路徑、原始 log）都吃事件自帶的那一份。
- 「當時的紀錄」那塊說明收成一行，細節放進問號；而且不再斷定是「超過保留
  期限」—— log 不在了也可能是有人清過，畫面沒有辦法分辨，就不要猜。

**5.1**
- **每個畫面都有自己的網址**：`#/event/FAB-260922-002`、`#/syslog`。重新整理
  會回到同一件事，可以加書籤，也可以把連結貼給同事直接開。上一頁／下一頁
  照常用。連結指向一件已經不存在的事故時，會退回清單並說明原因。
- 分頁與按鈕加上符號，卡片與彈出視窗有進場與滑入的動態（最長 260ms）。
  系統設定「減少動態」時全部關掉 —— 那是無障礙，不是偏好。

**5.1.1**
- 現場地圖補上**圖例**（跟光纖圖一樣放在角落）：點的顏色與點裡的數字、
  線的兩種顏色、點一台設備之後那三種外圈各代表什麼。
  其中**青色的點是「樞紐」**—— 那一群裡邏輯鄰接最多的一台，也是唯一一直
  掛著名字的。切到「+1 跳」時圖會分成更多群，所以樞紐會變多。

**5.2 —— 撈不到網管時，日誌要說得出是哪一種撈不到**
- 撈取失敗時，系統日誌會記下**網管到底回了什麼**：每一種失敗各幾次、前幾筆的
  狀態碼與**回應內容片段**（看得出是 JSON、SSO 登入頁、還是一句錯誤訊息）、
  以及每一種對應的下一步。以前只有一句「64 台查詢失敗」。
- 新增一種會被抓出來的壞法：**網管回得出設備，但一條鄰接都沒有**（欄位對不上）。
  以前這會被當成「撈取成功」，然後 N=0 —— 畫面看起來設定完成了，定位卻永遠
  算不出答案。現在它會中止並附上一台正常回應的原文。
- 現場地圖底部改成「圖上 58 台（受影響 35 台 ＋ 往外 1 跳 23 台）」，
  不再跟標題的 35 台互相矛盾。

**5.3 —— 對得上你們的網管，接不上也還能示範**
- **支援非 NetBox 形狀的回應**。你們那支把介面放在 `custom_fields.interfaces`、
  cable 兩端寫成 `termination_a_device` / `termination_a_interface`、筆數欄位
  叫 `total` —— 三處都跟 NetBox 不同，所以舊版每一台都查得到卻一條鄰接都沒有。
  現在兩種形狀都認。
- 還是對不上的話，日誌會**指出卡在哪一關**（找不到介面欄位／介面是空的／
  沒有 cable／cable 兩端對不上名字／status 值不認得），並附上那台設備實際
  有哪些欄位。
- 兩個新的環境變數，不必重做 image 就能調整：
  - `NMS_INTERFACE_PATHS=data.ports,attributes.interfaces` —— 介面在別的欄位時
  - `NMS_QUERY=depth=2&limit=100` —— 查詢參數要換一組時
- **撈不到網管時的備案**：設定第 3 步失敗後會出現「改用示範用的邏輯連線」，
  用匯入的光纖圖與設備清單兜一份出來，設定走得完、示範跑得動。
  它會在畫面上一直標紅字、狀態 API 回 `synthetic: true`、每次啟動寫一筆警告
  —— **不能拿來做任何判斷**。網管接得上之後重新撈一次就會蓋掉它。

**5.4 —— 保證演得完**
- image 內附**範例網路**（9 個 DC、12 段光纖、64 台設備）。設定第 1 步有一顆
  「載入內建的範例網路」，手邊沒有 CSV 也能把第 1、2 步一次走完。
- 於是最壞的情況也走得完：**全新機器 + 沒有任何檔案 + 網管完全不通**
  → 載入範例 → 撈取失敗 → 改用示範用的邏輯連線 → 開始監看 → 製造一次斷纖
  → 定位正確。這條路每次出版都會實跑一次。
- 範例資料與示範用的 N 都會在主控台一直標紅字，狀態 API 也回得出來
  （`sample_data` / `synthetic`）。匯入真的光纖圖就會蓋掉範例。

</details>

---

## 你要準備什麼

| 東西 | 要不要 | 說明 |
|---|---|---|
| 一台 x86_64 的 Linux，裝了 Docker | **要** | 其他什麼都不用 |
| 光纖圖 CSV | **要** | 哪兩個 DC 之間有實體光纖 |
| 設備清單 CSV | **要** | 每一台設備在哪個 DC |
| 網管 API 網址與 token | **要** | 系統自己去撈設備之間的連線 |
| log 來源 | **不用** | 這一版的 log 由內建模擬器產生 |

> **log 是模擬的，但不是亂編的。**
>
> 光纖圖、設備清單、邏輯連線全都是真的 —— 前兩份你匯入，邏輯連線是網管撈回來的。
> 模擬器只補上一件網管不會告訴你的事：**每一條邏輯連線實際上繞哪幾段光纖**。
> 那一件抽出來之後，「切這一條會斷哪些連線、產生哪些 log」全部是推導出來的。
>
> 抽出來的走法可以在畫面上整張列出來檢查（示範資料 → 看模擬走法），
> 事後也能反過來看「這個斷點切斷了每一條斷線的哪一段」（細節頁 → 斷點如何影響）。

---

## 名詞對照

看畫面之前先對一下，這三個詞指的是不同層級的東西：

| 詞 | 指什麼 |
|---|---|
| **DC** | 廠區裡的機房 |
| **光纖** | DC 與 DC 之間的實體光纖（你匯入的那份圖） |
| **邏輯連線** | 設備與設備之間的連線（網管撈回來的），一條邏輯連線會繞經好幾段光纖 |
| **Event** | 系統把一波 log 收成的一件事，有編號、可以認領結案，永久保存 |

---

# 第一部分：部署

## 步驟 1　確認主機

```bash
uname -m          # 要看到 x86_64
docker --version  # 20.10 以上就夠
```

沒有 Docker 的話：

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"   # 加完要登出再登入一次
```

## 步驟 2　取得映像檔

**A. 主機連得到網路**

```bash
docker pull coolguazi/fiber-cut-localizer:5.4
```

> 這個映像檔**只有 `linux/amd64`**（公司的 Linux server 就是這個）。
> 如果你在 Apple Silicon 的 Mac 上試拉，會看到
> `no matching manifest for linux/arm64/v8` —— 那不是壞了，加上平台就好：
>
> ```bash
> docker pull --platform linux/amd64 coolguazi/fiber-cut-localizer:5.4
> docker run --platform linux/amd64 …
> ```

**B. 主機連不到外網**（多數公司內網是這種）

在家裡／有網路的機器上：

```bash
docker pull --platform linux/amd64 coolguazi/fiber-cut-localizer:5.4
docker save coolguazi/fiber-cut-localizer:5.4 | gzip > fcl-5.4.tar.gz
```

把 `fcl-5.4.tar.gz` 拷到公司主機（約 38 MB），然後：

```bash
gunzip -c fcl-5.4.tar.gz | docker load
```

## 步驟 3　建立兩個檔案

開一個目錄，裡面只放這兩個檔：

```bash
mkdir -p ~/fiber-cut-localizer && cd ~/fiber-cut-localizer
```

### `docker-compose.yml`

```yaml
services:
  app:
    image: coolguazi/fiber-cut-localizer:5.4
    container_name: fiber-cut-localizer
    restart: unless-stopped
    ports:
      - "8000:8000"          # 要換 port 只改左邊那個數字
    env_file: .env
    volumes:
      - fcl-data:/data
    healthcheck:
      test: ["CMD", "python", "-c",
             "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health',timeout=3).status==200 else 1)"]
      interval: 30s
      timeout: 5s
      retries: 3

volumes:
  fcl-data:
```

> **`fcl-data` 這一行不能省。** 匯入的資料、所有 Event 歷史、每日 log 都住在裡面。
> 少了它，每次重建容器都要重新設定一次，而且結過案的 Event 會全部消失 ——
> 那是這套系統少數重建不回來的東西。

### `.env`

第一次先照抄，**但 `NMS_BASELINE_URL` 一定要填**。

「設備之間有哪些邏輯連線」只有網管講得出來，光纖圖與設備清單都不含這個資訊。
沒填的話設定會停在步驟 2-3（按了撈取沒有反應，階段一直停在「讀取網管」），
後面每一步都走不下去。

```bash
# ── 網管 API ────────────────────────────────────────────────
# NMS_BASELINE_URL 一定要填，其餘視網管而定
NMS_BASELINE_URL=
NMS_AUTH_TOKEN=
# 網管不是 Bearer 時，用 NMS_AUTH_HEADER 直接給整個 header 的值
NMS_AUTH_HEADER=
# 多久自己重讀一次，秒。預設一天
NMS_POLL_INTERVAL=86400
NMS_TIMEOUT=10
NMS_CONCURRENCY=8

# ── 資料保留 ────────────────────────────────────────────────
# log 留幾天。Event 是永久的，不受這個影響
LOG_RETENTION_DAYS=7

# ── 示範資料 ────────────────────────────────────────────────
# 1＝畫面上有「示範資料」按鈕，可以自己製造一次斷纖來確認系統活著。
# 它會寫入**假的** log，跟真的混在同一條時間軸上 —— 要做正式量測或
# 交報告之前改成 0 再重啟。系統日誌每次啟動都會警告你它是開著的。
DEMO_MODE=1

# ── 其他 ────────────────────────────────────────────────────
# 畫面左上角的系統名稱。改了重啟就生效
APP_NAME=Optiview
# 畫面與每日 log 檔名用的時區。不設會變成 UTC
TZ=Asia/Taipei
DEFAULT_PRODUCT=FAB
LOG_LEVEL=INFO
```

> **註解要自己一行，不可以寫在 `=` 後面。**
> Docker Compose 讀 `.env` 時**不會**把 `#` 之後的字當成註解 —— `=` 右邊整行
> 都是值。寫成 `NMS_AUTH_HEADER=  # 說明文字` 的話，那句說明文字就會被當成
> 認證標頭送給網管，然後每一台設備都查詢失敗，而錯誤訊息
> （`'ascii' codec can't encode characters…`）完全看不出問題在這一行。

## 步驟 4　啟動

```bash
docker compose up -d
docker compose ps        # STATUS 要看到 healthy
```

## 步驟 5　打開瀏覽器

```
http://<主機IP>:8000/
```

第一次進去是設定精靈，共四步。

---

# 第二部分：第一次設定（跟著精靈走）

## 2-1　匯入光纖圖

CSV，兩欄，一行一條光纖：

```csv
site_a,site_b
71DC1,71DC2
71DC1,71DC3
71DC2,71DC3
```

**這份檔案同時定義了「有哪些 DC」**，第 2-2 步會逐行對照它。

## 2-2　匯入設備清單

CSV，`role` 欄可有可無：

```csv
hostname,dc,role
RT71P1-71P1Z1Y03-1-21-FAB-CORE,71DC1,core
SW71P1-GBD07-25-15-EQP,71DC1,access
AGG-DC2-01,71DC2,access
```

> **`dc` 欄不能空。** 系統不會從設備名稱去猜它在哪個 DC —— 猜錯一台，
> 它斷線時就會算到別的 DC，找出來的斷點也跟著錯。

每一步都是**先預覽再確認**，按取消是真的什麼都沒發生。

## 2-3　讀取網管

按一下，系統會逐台去問網管、算出設備之間有哪些連線。

**沒填 `NMS_BASELINE_URL` 的話這一步過不去。** 按了撈取不會有動靜，階段一直
停在這裡 —— 因為「哪兩台之間有連線」只有網管講得出來，光纖圖與設備清單都沒有
這個資訊。回去把 `.env` 填好，`docker compose up -d` 重來。

撈完會給一份**撈取報告**：問到幾台、哪幾台查無此設備、哪幾台查得到但沒有任何
已接線的介面、網管回報了幾台清單上沒有的設備。

> **這份報告不要跳過。** 它是唯一會講出「有幾條連線沒被算進來」的地方。
> 少算的那幾條在斷線時系統看不到，反而會被當成「這條光纖沒事」的理由。

## 2-4　大功告成

第四步是總覽：幾條光纖、幾台設備、幾對邏輯連線、每個 DC 幾台。
數字對了就按 **「開始監看 →」**。

---

## 保底：什麼都不順的時候怎麼演

三個東西都可能不在：CSV 不在這台機器上、網管連不上、或網管的回應對不上。
下面這條路**完全不需要外部條件**，五分鐘走得完：

1. 第 1 步 →「載入內建的範例網路」（9 個 DC、64 台設備）
2. 第 3 步 → 按「開始撈取」，讓它失敗（沒有網管就是這個結果）
3. 失敗後會出現紅框 →「改用示範用的邏輯連線」→ 確認
4. 「開始監看 →」進主控台
5. 右上「示範資料」→ 製造一次斷纖 → 等約 15 秒 → 事件出現在清單上
6. 點進事件：看定位結果、受影響的現場、斷點如何影響、原始 log

畫面上會一直有紅字說這是範例資料 —— **那是刻意的**。示範可以用它，
任何判斷不行。

# 第三部分：走一次完整流程（確認它真的能用）

## 3-1　製造一次斷纖

主畫面右下角有一顆小小的 **「示範資料」**，點開是一個彈出視窗。

1. 「切斷」選一條光纖（或留「隨機挑一條」）
2. 建議先按 **「看模擬走法」** —— 那是整套模擬唯一隨機的東西，
   會列出每一條邏輯連線繞哪幾段光纖。選了光纖之後，會被它切斷的那幾列標紅
3. 按 **「✂ 切下去」**

> 同一份光纖圖與設備清單抽出來的走法固定不變（畫面上有指紋可以核對），
> 重開服務也是同一張 —— 所以示範講到一半重切一次，出現的是同一批設備、
> 同一個答案。

## 3-2　等 30 秒

最後一筆 log 之後再 25 秒沒有新的，系統就把這一波收成一個 **Event**，
配一個編號（例如 `FAB-260922-001`），算出最可能的斷點並存下來。

主畫面清單會出現這一列。斷線少於 3 條不會開 Event（會歸到「規模太小」那一頁）。

## 3-3　點進去看

細節頁應該有這些：

* **左側**：光纖圖，斷點用紅色虛線標出來
* **左下**：前 10 名嫌疑與分數
* **上方**：時間軸，可以框選任一段重新計算
* **中間**：受影響的現場 —— **先救哪一條由你判斷**。系統不排順序：
  設備名稱看不出哪些是主幹、哪些是備援，而那是決定順序的主要因素。
  在圖上點一對設備，才會給它的替代路徑
* **底部**：結論帶 —— `最可能的斷點 FIBER-… 100% 證據充分 · 斷線 29 條 / 82 · 影響 9 個 DC`

中間有三個分頁：

| 分頁 | 看什麼 |
|---|---|
| 受影響的現場 | 一張設備地圖，紅線是斷掉的邏輯連線。鄰居越多的設備群越靠中心 |
| **斷點如何影響** | 每一條斷線攤開，標出這個斷點切斷了它走法上的第幾段 |
| 原始 log | 這段區間內的每一筆 |

「斷點如何影響」長這樣：

```
RT71P3-…  Eth1/8 ↔ RT71P4-…  Eth1/1     71DC3 → 71DC4     共 4 段 · 斷在第 1 段
【FIBER-71DC2-71DC3】 → FIBER-71DC1-71DC2 → FIBER-71DC1-71DCA → FIBER-71DC4-71DCA
```

結論帶給的是**答案**，這一頁給的是**解釋**。派工出去挖一條光纖之前，
「為什麼是這一條」不該只是一個要相信的數字。

看到這些，就表示這台機器上的東西是完整的。

---

# 第四部分：接上真實網管

填 `.env` 的兩行：

```bash
NMS_BASELINE_URL=https://你們的網管/api/dcim/devices/
NMS_AUTH_TOKEN=你們的token
```

```bash
docker compose up -d       # 重新套用 .env
```

然後到設定頁按 **「← 重新讀取網管」**，看那份撈取報告。

系統打的請求（`GET`，帶 `Authorization: Bearer <token>`）：

```
{NMS_BASELINE_URL}?include_interfaces=true&is_active=true&offset=0&limit=100&hostname=<主機名>
```

期望回來的形狀（NetBox 相容）：

```json
{"count": 1, "results": [{"name": "RT71P1-…", "interfaces": [
  {"name": "Eth1/1", "status": {"name": "Up"},
   "cable": {"termination_a": {"device": {"name": "RT71P1-…"}, "name": "Eth1/1"},
             "termination_b": {"device": {"name": "RT71P2-…"}, "name": "Eth2/3"}}}
]}]}
```

不是這個形狀的話要改解析程式，找開發者處理。

---

# 第五部分：維護

## 資料留多久

| | 留多久 | 存在哪 |
|---|---|---|
| log | **7 天**（`LOG_RETENTION_DAYS`），每天 04:00 刪掉過期的整天 | `/data/<單位>/logs/YYYY-MM-DD.jsonl` |
| Event | **永久，沒有數量上限** | `/data/<單位>/alerts/<編號>.json`，一件一個檔 |
| Event 裡的 log | 跟著 Event 永久保存 | 同上 |

Event 收斂的那一刻，會把它那段時窗的全部 log 抄一份進去。
所以半年後點開一件舊 Event，畫面會標明「這是當時留下的紀錄」，
然後照常畫出當時的斷點、地圖與排名 —— 即使原始的每日 log 檔早就被清掉。

查目前的保留狀況：

```bash
curl -s http://localhost:8000/api/topology/retention
```

## 日常指令

```bash
docker compose logs -f              # 看服務在做什麼
docker compose restart              # 重開（資料都在）
docker compose down                 # 停掉（資料都在）
docker compose down -v              # 停掉並刪掉所有資料 ← 小心
```

## 備份

Event 歷史重建不回來，建議排程備份：

```bash
docker run --rm -v fcl-data:/data -v "$PWD":/out alpine \
  tar czf /out/fcl-backup-$(date +%F).tar.gz -C /data .
```

還原：

```bash
docker run --rm -v fcl-data:/data -v "$PWD":/in alpine \
  tar xzf /in/fcl-backup-2026-09-22.tar.gz -C /data
```

## 升級

```bash
docker compose pull && docker compose up -d
```

`/data` 不會動。舊版的單一 `alerts.json` 會在啟動時自動拆成一件一個檔，
原檔改名成 `alerts.json.migrated` 保留著，不會刪。

## 日誌檔太多的時候

系統日誌一天一個檔，放在 `/data/syslog/`。錯誤另外抄一份到 `errors.jsonl`。

在系統日誌那一頁下方：

* **清除日誌** —— 刪掉每日的檔案，**錯誤那一份保留**
* **連錯誤一起清** —— 兩份都刪

> 清除會自己記一筆，所以事後看得出日誌為什麼斷在那裡。
>
> 兩顆按鈕是故意分開的：清空間想清的是幾千行 INFO，不是那十筆錯誤 ——
> 共用一顆的話，總有人會在排查中途把唯一的線索刪掉。

## 換 port

`docker-compose.yml` 裡 `"8000:8000"` 改左邊那個數字，例如 `"9000:8000"`，
然後 `docker compose up -d`。

---

# 第六部分：撈不到網管資料時怎麼查

> 這一節是為了**照著 SOP 做卻撈不到、或網管回 200 但沒東西**寫的。
> 那種情況畫面只能說「查得到但沒鄰接」，而那句話蓋著至少五種原因。

## 6-1　先開「系統日誌」

**標題列右上角有一個「系統日誌」分頁**，點進去是這套系統的日誌頁 ——
出過什麼錯、系統做過什麼，都在那裡。

每一列長這樣：

```
03:00:10  INFO   app.sources.nms_source.refresh:252   inventory sweep ok: … -> |N|=82
03:00:12  INFO   app.audit.set_collecting:513         開始收集 log                    +
11:02:33  ERROR  app.sources.nms_source.refresh:237   撈取中止：38 台查無此設備        +
```

中間那一欄是**哪個程式、哪一段**（模組 → 函式 → 行號）。有 `+` 的點開會展開
細節與錯誤堆疊。上面的下拉可以切：

| 選項 | 看什麼 |
|---|---|
| 只看錯誤 | 出事時先看這個 |
| **做了什麼** | 誰在幾點匯入了清單、按了開始／暫停、認領結案、清了日誌 |
| 搜尋框 | 比對訊息、模組、細節 |

> **「做了什麼」這一類很值得先看。** 定位突然變差，多半不是程式壞了，
> 是有人換了一份清單或按了暫停 —— 而那不會產生任何錯誤訊息。

## 6-1-1　先看系統日誌裡那一筆撈取紀錄

撈取失敗時，系統日誌會留下一筆 **ERROR**，點開就有排查要的全部材料：

```
撈取中止：0 台查無此設備、64 台查詢失敗（共 64 台）。主要原因：http_401×64
  context ↓
  url        https://你們的網管/api/dcim/devices/
  by_kind    {"http_401": 64}          ← 每一種失敗各幾次
  samples    [{ kind: http_401, status: 401,
               response_sample: {"detail": "invalid token"},
               next_step: "認證失敗。換一個 token；網管不是 Bearer 的話用
                           NMS_AUTH_HEADER 給整個標頭。" }]
```

`by_kind` 先看：它告訴你主因是哪一種。`samples` 裡的 `response_sample` 是
**網管實際回的內容**，`next_step` 是那一種對應的動作。

| `by_kind` 的分類 | 意思 | 要做什麼 |
|---|---|---|
| `http_401` / `http_403` | token 不對 / 沒權限 | 換 token；不是 Bearer 就用 `NMS_AUTH_HEADER` |
| `http_404` | 網址路徑不對 | 常見是少了結尾斜線 |
| `http_5xx` | 網管自己出錯 | 等一下再撈，或找網管管理員 |
| `timeout` | 沒在時限內回應 | 確認網路可達，必要時調大 `NMS_TIMEOUT` |
| `connect` | 連不上 | DNS、防火牆、TLS 憑證 |
| `parse` | 回應不是 JSON | 看 `response_sample`，開頭是 `<html>` 就是 SSO 登入頁 |
| `unknown` | 網管查無此 hostname | 設備清單過時，重新匯入 |
| `empty` | 查得到但沒有可用的介面／纜線 | 欄位名稱可能不同，往下跑分段診斷 |

還有一種不會出現在上表、但一樣會被擋下來的情況：**每一台都問到了，卻一條
鄰接都沒有**。那通常是欄位對不上（介面在另一個端點、cable 兩端用 FQDN 而
清單用短名）。這一輪會被判定失敗並保留舊資料，日誌裡附上一台正常回應的原文。

## 6-2　再跑分段診斷

撈不到網管資料時，這條路由會對網管打**一次**查詢，一段一段告訴你走到哪裡停下來：

```bash
curl -s http://localhost:8000/api/topology/nms-probe | python3 -m json.tool
```

結果同時寫進系統日誌，所以也可以直接在那一頁看（搜「NMS 診斷」）。

```
✓ 設定          URL 有設定；Bearer …（長度 40）
✓ 連線          連上了
✓ HTTP 狀態     200 OK
✓ JSON          解析成功（dict）
✓ results       count=1，results 有 1 筆
✓ 設備名稱      網管回報的名稱 = 'RT71P1-…'
✓ 介面          6 個介面
✓ status = Up   6/6 個是 Up；出現過的 status 值：['Up']
✓ 有 cable      5/6 個 Up 的介面有 cable 欄位
✗ cable 兩端對得上   0/5 條線的兩端有一端等於 'RT71P1-…'
```

最後一行就是答案 —— 它會附上 `next_step` 說要做什麼。

想指定問哪一台：`?hostname=RT71P1-…`

**這條路由用的是撈取自己在用的解析程式**，所以它過了而撈取失敗、或反過來，
都不會發生。

## 6-3　各段卡住的意思

| 卡在 | 意思 | 要做什麼 |
|---|---|---|
| 設定 | `NMS_BASELINE_URL` 沒填 | 填 `.env`，`docker compose up -d` |
| 設定 | 每一台都失敗，訊息是 `'ascii' codec can't encode characters…` | `.env` 有一行把註解寫在 `=` 後面，整段註解被當成值送出去了。把註解移到上一行再重啟。系統日誌會直接指出是哪一個變數 |
| 連線 | DNS 解不出來 / TLS 不被信任 / 防火牆 | 在**容器裡**試：`docker compose exec app python -c "import httpx;print(httpx.get('<網管URL>').status_code)"` |
| HTTP 狀態 401/403 | token 不對、過期、範圍不足 | 換 token；網管不是 Bearer 就用 `NMS_AUTH_HEADER` 給整個 header |
| HTTP 狀態 404 | URL 路徑不對（常見：少了結尾斜線） | 對照網管的 API 文件 |
| JSON | 回應不是 JSON | 多半是 SSO 登入頁或代理錯誤頁 —— 那些也回 200。看 `response_sample` 開頭是不是 `<html>` |
| results | 200 但 `results` 空的 | ①這台真的不在網管裡 ②查詢參數名稱不是 `hostname`（有的叫 `name`/`q`/`device`）③token 看不到這台 |
| 介面 | 查得到設備但沒有介面 | 少了 `include_interfaces`，或介面在別的端點 |
| status = Up | 介面都不是 `Up` | 診斷會印出實際出現過的 status 值（例如 `['up']`、`['active']`）。大小寫不同就要改解析規則 |
| 有 cable | 是 Up 但沒有接線記錄 | 網管的 cable 欄位是空的 |
| **cable 兩端對得上** | **「回 200 卻沒東西」最常見的原因** | cable 的 termination 名稱必須跟 `results[0].name` 字元級相同。常見差異：一邊 FQDN 一邊短名、大小寫、前後空白。診斷會把兩邊實際的字串並排印出來 |
| 對端在清單上 | 撈到了，但對端不在設備清單 | 把那些設備補進設備清單重新匯入 |

## 6-4　把資料撈回來給開發者

**最省事的方式：** 系統日誌那一頁的下方有日誌檔清單，每一個都有「下載」。
把當天那個 `.jsonl` 與 `errors.jsonl` 下載下來寄出即可。

**加上一份系統快照**（設定、資料狀態、一次網管診斷）：

```bash
curl -s http://localhost:8000/api/topology/diagnostics > fcl-diag-$(date +%F-%H%M).json
```

裡面有：環境變數（**token 只說有沒有設定，不含值**）、設定階段、光纖圖與設備
清單的狀態、撈取結果、保留策略、以及一次分段診斷。約 6 KB。

**加上服務的 log**（分段診斷也會寫進這裡，所以兩者對得起來）：

```bash
docker compose logs --no-color --tail 500 > fcl-logs-$(date +%F-%H%M).txt
```

把這兩個檔一起寄出即可。

> **寄出前自己看一眼。** 診斷包不含 token，但**會**有你們的設備名稱與 DC 代碼
> —— 那是排錯需要的。如果對外分享有顧慮，先確認過再寄。

## 6-5　要送進 syslog 的話

服務把 log 寫到 stdout（容器的標準做法），所以交給 Docker 的 log driver 轉發就好，
程式不用改。在 `docker-compose.yml` 的 `app:` 底下加：

```yaml
    logging:
      driver: syslog
      options:
        syslog-address: "udp://127.0.0.1:514"
        tag: "fiber-cut-localizer"
```

然後 `docker compose up -d`。之後 log 會進系統的 syslog，用你們原本的方式收集
（`journalctl -t fiber-cut-localizer` 或 `/var/log/syslog`）。

想更詳細的話把 `.env` 的 `LOG_LEVEL` 改成 `DEBUG`，那會多印每一個網管請求的
URL 與狀態碼。**排錯完記得改回 `INFO`** —— DEBUG 在撈取時每台設備一行，
一輪下來幾百行。

---

# 第七部分：其他問題怎麼查

先跑這三行：

```bash
docker compose ps
docker compose logs --tail 100
curl -s http://localhost:8000/api/topology/status
```

| 症狀 | 多半是 | 怎麼辦 |
|---|---|---|
| 網頁連不上 | port 被佔或防火牆 | `ss -tlnp \| grep 8000`；換 port |
| 畫面打得開但時間軸空的 | 沒按「開始監看」 | 主畫面左上角那顆大按鈕，要顯示「監控中」 |
| 切了光纖沒開出 Event | 斷線條數沒過門檻 | 主畫面底部「Event 的認定條件」會說出目前的門檻 |
| 切了光纖完全沒有 log | 那一刀沒有任何走法經過 | 示範面板會直接說；換一條，或先看「看模擬走法」挑承載多的 |
| 撈取報告說一堆設備查無 | 設備清單過時 | 重新匯出一份清單再匯入 |
| 定位結果怪怪的 | 邏輯連線少算了 | 看撈取報告的「網管回報了 N 台清單上沒有的設備」，把它們補進清單 |

容器起不來時看完整錯誤：

```bash
docker logs fiber-cut-localizer
```

---

# 一頁速查

```bash
# 第一次
mkdir -p ~/fiber-cut-localizer && cd ~/fiber-cut-localizer
# 建立 docker-compose.yml 與 .env（內容見步驟 3）
docker compose up -d
# 瀏覽器開 http://<主機IP>:8000/ ，跟著精靈走完四步
# 主畫面右下角「示範資料」→ 看模擬走法 → 切下去 → 等 30 秒 → 點開 Event

# 接真實網管
vi .env                     # 填 NMS_BASELINE_URL 與 NMS_AUTH_TOKEN
docker compose up -d
# 設定頁 →「← 重新讀取網管」→ 看撈取報告

# 日常
docker compose logs -f
docker compose pull && docker compose up -d      # 升級
docker run --rm -v fcl-data:/data -v "$PWD":/out alpine \
  tar czf /out/fcl-backup-$(date +%F).tar.gz -C /data .   # 備份
```
