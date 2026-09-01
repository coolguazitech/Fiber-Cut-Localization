# Docker 部署 SOP

到公司之後照著這份做，**不用改任何程式碼**。從零到畫面上有事件，大約五分鐘。

Docker Hub：<https://hub.docker.com/r/coolguazi/fiber-cut-localizer>　目前版本 **3.0**

---

## 這一版是什麼狀態

| | |
|---|---|
| **log 從哪來** | 內建的示範產生器，**寫死在 image 裡**。撈 log 的 API 還沒接 |
| **光纖圖與 peer 關係** | 沒設定網管 API 時，用 image 內建的預設網（9 站 · 16 條光纖 · 702 對） |
| **要填的東西** | 只有 `.env` 一個檔案。不填也能跑 |
| **對外連線** | 沒設定網管 API 時**完全沒有**。整台機器不會往外打任何一個封包 |

也就是說：**先 `docker compose up -d`，畫面就能用**。要接真實網管再回頭填 `.env`。

---

## ① 確認機器上有 docker

```bash
docker --version
docker compose version
```

沒有的話：

- Linux：`curl -fsSL https://get.docker.com | sh`
- Windows / Mac：裝 Docker Desktop

---

## ② 準備兩個檔案

在你想放的目錄下（例如 `~/fiber-cut-localizer/`）建立**兩個檔案**，其他什麼都不用。

### `docker-compose.yml`

```yaml
services:
  console:
    image: coolguazi/fiber-cut-localizer:3.0
    container_name: fiber-cut-localizer
    restart: unless-stopped
    init: true
    ports:
      - "${HOST_PORT:-8000}:8000"
    environment:
      NMS_BASELINE_URL: "${NMS_BASELINE_URL:-}"
      NMS_AUTH_TOKEN: "${NMS_AUTH_TOKEN:-}"
      NMS_AUTH_HEADER: "${NMS_AUTH_HEADER:-}"
      NMS_POLL_INTERVAL: "${NMS_POLL_INTERVAL:-86400}"
      NMS_TIMEOUT: "${NMS_TIMEOUT:-10}"
      NMS_CONCURRENCY: "${NMS_CONCURRENCY:-8}"
      SEED_DEFAULT_TOPOLOGY: "${SEED_DEFAULT_TOPOLOGY:-}"
      DEMO_MODE: "${DEMO_MODE:-1}"
      DEFAULT_PRODUCT: "${DEFAULT_PRODUCT:-FAB}"
      LOG_LEVEL: "${LOG_LEVEL:-INFO}"
      ALERT_SETTLE_SECONDS: "${ALERT_SETTLE_SECONDS:-25}"
      ALERT_MIN_DOWN_PEERS: "${ALERT_MIN_DOWN_PEERS:-3}"
      EVENT_RETENTION_SECONDS: "${EVENT_RETENTION_SECONDS:-3600}"
      EVENT_BUFFER_SIZE: "${EVENT_BUFFER_SIZE:-50000}"
    volumes:
      - fcl-data:/data
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health',timeout=3).status==200 else 1)"
      interval: 30s
      timeout: 5s
      start_period: 15s
      retries: 3

volumes:
  fcl-data:
```

> `fcl-data` 這個 volume **不要省略**。匯入的光纖圖、設備清單、網管快照，以及
> **所有事件歷史**都住在裡面。少了它，每次 `docker rm` 都要重新匯入，而且結過案
> 的事件會消失 —— 事件歷史是這套系統少數不能重建的東西。

### `.env`

第一次先照抄，**一個字都不用改**：

```bash
# --- 網管 API。第一次部署先全部留白，用內建的預設網 ---
NMS_BASELINE_URL=
NMS_AUTH_TOKEN=
NMS_AUTH_HEADER=
NMS_POLL_INTERVAL=86400
NMS_TIMEOUT=10
NMS_CONCURRENCY=8

# --- 光纖圖。留白 = 依網管有沒有設定自動判斷 ---
SEED_DEFAULT_TOPOLOGY=

# --- 示範資料產生器。目前 log 的唯一來源，先留 1 ---
DEMO_MODE=1

# --- 其他 ---
HOST_PORT=8000
DEFAULT_PRODUCT=FAB
LOG_LEVEL=INFO
```

> `.env` 之後會放憑證，**不要進版控、不要貼到群組**。

---

## ③ 啟動

```bash
docker compose up -d
docker compose ps          # STATUS 要是 healthy（等約 15 秒）
```

第一次會自動下載 image（約 39 MB）。

架構不用指定 —— tag 裡 `amd64` 與 `arm64` 都在，Docker 依機器自己挑。
想確認抓到哪一份：

```bash
docker image inspect coolguazi/fiber-cut-localizer:3.0 --format '{{.Os}}/{{.Architecture}}'
# 公司的 Linux server 應該是 linux/amd64
```

---

## ④ 開瀏覽器

<http://localhost:8000/>

**如果是跑在別台 server 上**，組員開 `http://<server-ip>:8000/`（`ports` 已經是
`0.0.0.0`，不用改）。

---

## ⑤ 走一次完整流程（5 分鐘）

### 5-1　開始收集

第一次進去會看到設定精靈。**什麼都不用匯入** —— 直接按最下面的
**Start collecting logs**。

> 為什麼不用匯入：沒設定網管 API 時，光纖圖與 peer 關係用 image 內建的預設網，
> 設備清單也不需要（它唯一的用途是告訴網管要問哪些設備，而現在沒有網管要問）。

### 5-2　製造一次斷纖

主畫面最上面有一條**黃色斜紋**的「示範資料」列 → **開啟控制台** → 下拉選一條
光纖（或「隨機挑一條」）→ **✂ 切下去**。

會送出 90～200 筆 INT_DOWN，約 3～5 秒送完。

> 黃色斜紋是刻意的：畫面上同時有真實資料跟捏造資料時，唯一防止混淆的東西就是
> 它們看起來不一樣。

### 5-3　等 30 秒

最後一筆 log 之後**再等 25 秒**沒有新的，事件才會從「進行中」收斂成「待處理」。

事件清單會出現一列，長這樣：

```
🔔 FAB-260901-001  待處理    ✂ FIBER-0007  DC2 ↔ DC4  信心高      09/01 14:23:11
   142 條斷線 / 702 · DC1 ↔ DC2、DC1 ↔ DC5、DC1 ↔ DC6 等 33 組
   ↔ 先救 dev-dc7-020 ↔ dev-dc9-009  2/10 斷  12 條可繞  等 126 對
                                            [認領] [結案] [開啟 →]
```

（上面這幾個數字是拿 3.0 的 image 實際跑出來的 —— 切 `FIBER-0007` 就會得到
同一批，因為示範產生器的隨機種子是從光纖編號推出來的。）

**確認這一列的光纖編號，跟你在 5-2 選的那一條一樣。** 內建預設網 16 條逐條測過，
16/16 都定位正確。

### 5-4　點進去看細節

點那一列 → 進細節頁。時間軸會自動框到那段區間，右側出現：

- **搶救順序** —— 先救哪幾對設備。展開有四條規則的實際數值
- **原始 log** —— 那段區間內每一筆，可用設備／介面／站點搜尋

最上面那條 bar 顯示的是**事件收斂當下存下來的判斷**，不隨時間軸移動。你把底下的
時間窗拖走再重算，兩個答案一不一樣，一眼看得到。

#### 搶救順序是怎麼排的：四條明文規則

不是加權分數，是四條寫死的規則。**四條同時作用，強弱不同** —— 就像 Excel 從第 4
欄往回排到第 1 欄，最後規則 1 的條件最強、規則 4 最弱：

| # | 規則 | 哪一邊排前面 |
|---|---|---|
| 1 | **ADM 設備** | hostname **不含** `ADM` 的排在含 `ADM` 的前面 |
| 2 | **邏輯連線總數** | 這對 peer 在網管上的連線**總數越多**越前面 |
| 3 | **斷線比例** | 斷掉數 ÷ 總數的**比例越小**越前面 |
| 4 | **替代光纖路徑** | **沒有**替代路徑可繞的排在有的前面 |

畫面上每一列展開後，這四條的實際數值都會逐條列出來 —— 包含沒幫上忙的那幾條。
只列對自己有利的理由讀起來像在說服人，而值班的人需要的是能自己驗算的完整依據。

**不是「滿足規則 1 就不看後面三條」。** 規則 1 相同才是常態（多數 hostname 都不含
`ADM`），所以實際在排順序的通常是 2、3、4。

> **規則 3 看起來反直覺，交接時值得講一次**：分母是網管的 last-seen 連線數，而網管
> 一天才爬一次，所以挖斷當下分母**還是昨天的數字**，不會跟著變少。比例小代表這對
> peer 原本連線很多、目前只斷了一小部分 —— **還有餘裕但正在流失**，比早已全斷的
> 更值得先搶。已經 100% 斷光的那對能救回的邊際效益反而低，而且往往是末端單鏈路。

> **為什麼是規則不是分數**：凌晨三點被叫起來的人看到「嚴重度 0.72」無法據以行動，
> 也無法判斷它憑什麼排在 0.69 前面；權重一調，昨天的第一名今天變第三名，而沒有
> 任何人為此做過決定。規則可以被覆述、被質疑、被主管否決。

### 5-5　認領、結案

事件那一列右邊有「認領」「結案」。結案會問處理結果，填了會存進歷史。

**重啟之後歷史還在**：

```bash
docker compose restart
```

### 5-6　log 一小時後會被清掉 —— 那到底留下了什麼

**先講結論：原始 log 這一版不落地。** 它只活在記憶體裡的環狀緩衝區，
`EVENT_RETENTION_SECONDS`（預設 3600 秒）與 `EVENT_BUFFER_SIZE`（預設 50000 筆）
兩個上限誰先到就先砍，而且**容器一重啟就全部歸零**（上一步 `restart` 之後
`retained_events` 會是 0，事件卻原封不動 —— 那就是這件事的示範）。

會留下來的是**事件**。事件收斂的那一刻，系統把當下算出來的答案整份寫進 `/data`
（compose 裡的 `fcl-data` volume），一個單位一個 JSON 檔，最多保 500 件：

| 一小時後還在（存在 volume） | 一小時後就沒了（只在記憶體） |
|---|---|
| 事件編號、狀態、認領人、結案結論 | 每一筆 INT_DOWN 的原始內容 |
| 嫌疑光纖、第二名、信心 | 時間軸波形與拖曳重算 |
| 斷線 peer 數 / 基準總數、站點組數 | 細節頁的「原始 log」分頁 |
| 搶救順序前 5 名與四條規則的當時數值 | 該區間的鄰域圖 |

也就是說：**事後檢討看得到「當時判斷是哪一條、依據是什麼、誰處理的」，但看不到
那 142 筆 log 長什麼樣。** 這是刻意的取捨 —— 檢討會上要問的是前者，而把 log 長期
存起來是 log 系統的工作，不是這台機器的。

驗證一次（清掉 log，事件不動）：

```bash
curl -s -X POST "localhost:8000/api/demo/clear?product=FAB"   # 只清 log
curl -s "localhost:8000/api/alerts?product=FAB"               # 事件原封不動
```

#### 如果真的需要留原始 log

按代價從輕到重有三條路：

1. **調長保留時間** —— `.env` 裡加 `EVENT_RETENTION_SECONDS=86400`（一天），
   `docker compose up -d`。代價是記憶體：實測一筆約 350 bytes，五萬筆吃滿約
   17 MB。要一起放寬筆數才有意義（`EVENT_BUFFER_SIZE`，兩個都要調，先到的先砍）。
   **重啟仍然全丟** —— 這只是把窗口拉長，不是保存。
2. **在來源端抄一份到容器外** —— log 本來就是打進 `/api/alarms/ingest` 的，
   在送進來之前多打一份到既有的 syslog／Loki／檔案即可。這套系統不是 log 系統，
   不該是那份 log 唯一的所在。
3. **接真實 log 來源** —— `sources/log_source.py` 的介面留著了但**還沒實作**
   （`HttpLogSource.fetch` 直接丟 `NotImplementedError`，不回空陣列 ——
   空陣列跟一張安靜的網路長得一模一樣）。接上去之後原始 log 本來就在上游，
   這裡的緩衝區只是為了畫時間軸。

> 不要把 `EVENT_RETENTION_SECONDS` 設成 0 或負數：那是「不依時間清除」，
> 只剩筆數上限在擋，記憶體會一路長到五萬筆才開始輪替。

---

## ⑥ 之後：接上真實網管 API

只改 `.env` 這一個檔案：

```bash
NMS_BASELINE_URL=https://你的網管位址/dcim/devices/
NMS_AUTH_TOKEN=你的token
DEMO_MODE=                 # ← 留白！正式環境不要有捏造 alarm 的按鈕
```

```bash
docker compose up -d       # 重新套用，不用重新下載
```

**`NMS_BASELINE_URL` 這一個變數決定機器處於哪個階段：**

| | 留白（現在） | 有填（正式） |
|---|---|---|
| 光纖圖 | image 內建的預設網 | 你匯入的那份 |
| peer 關係 | 光纖圖夾帶的 | 每天向網管撈一次 |
| 設備清單 | 不需要 | **必須匯入** |
| 對外連線 | 完全沒有 | 每天一次全掃 |

接上網管之後的第一次設定，在畫面上依序做：

1. 匯入**光纖圖**（哪些站點之間有實體光纖，CSV 或 JSON，精靈裡有範本可下載）
2. 匯入**設備清單**（要問網管哪些設備，一行一個 hostname）—— 匯入後自動開始掃描
3. 等掃描跑完（畫面上有進度），確認 peer 對數合理
4. 開始收集

掃描預設一天一次，在凌晨 03:00 跑（可在精靈裡改）。掃描期間仍用上一份資料服務，
不會出現半套的網路。

---

## ⑦ 日常操作

```bash
docker compose logs -f          # 看 log（Ctrl+C 離開）
docker compose ps               # 看狀態
docker compose restart          # 重啟（資料留著）
docker compose down             # 停掉（資料留著）
docker compose down -v          # 停掉並【刪除所有資料與事件歷史】← 小心
```

### 換 port

`.env` 裡改 `HOST_PORT=9090`，然後 `docker compose up -d`。
（container 內部固定聽 8000，不要改。）

### 升級

```bash
docker compose pull
docker compose up -d
```

資料在 volume 裡，不會動到。

---

## ⑧ 出問題先看這三個

| 症狀 | 先確認 |
|---|---|
| 按了切光纖但什麼都沒發生 | 有沒有按過「開始收集」。沒開的話 log 會被拒收，畫面上那條黃列會有紅字提示 |
| log 有進來但事件沒開 | 等滿 30 秒了嗎（送完 + 25 秒靜默）。還有清單下方的「未達門檻」有沒有加一 —— 斷線數沒到 3 條不當成事故 |
| 事件開了但嫌疑是別條 | 看「信心」。「信心低」代表前兩名幾乎分不出來，那是網路本身的可識別性問題，不是系統故障 |

要更細的逐步驗收（含 curl 指令與預期輸出），看 [`USAGE_zh.md`](USAGE_zh.md)。

---

## 一頁速查

```bash
# 第一次
mkdir -p ~/fiber-cut-localizer && cd ~/fiber-cut-localizer
# 建立 docker-compose.yml 與 .env（內容見上面 ②）
docker compose up -d

# 開瀏覽器 → http://localhost:8000/
# 設定精靈按 Start collecting logs
# 黃色那條「示範資料」→ 開啟控制台 → ✂ 切下去 → 等 30 秒

docker compose logs -f      # 看 log
docker compose restart      # 重啟
docker compose down         # 停掉（資料留著）
```

---

## Image 規格

| 項目 | 值 |
|------|------|
| Repository | `coolguazi/fiber-cut-localizer` |
| Tag | `3.0` / `latest` |
| Platform | `linux/amd64` + `linux/arm64`（同一個 tag，Docker 自己挑。Intel／AMD server 與 Apple Silicon Mac 都是原生跑，不用模擬） |
| Size | 約 39 MB 下載 / 168 MB 解壓後 |
| Base | `python:3.12-alpine`（Alpine 3.24.1、Python 3.12.14） |
| **CVE** | **0 critical / 0 high / 0 medium / 0 low**（Trivy，2026-09-01 於 3.0 實測） |
| Runtime user | `app`（uid 10001），非 root |
| Port | 8000 |
| Healthcheck | 內建（`python urllib` 戳 `/health`） |
| pip | **已從 runtime 移除** |

自己複驗：

```bash
trivy image --platform linux/amd64 coolguazi/fiber-cut-localizer:3.0
```

> **為什麼移掉 pip**：pip 會 vendor 自己的 msgpack 與 setuptools，掃描器把那些讀成
> 已安裝套件 —— 那是兩個 HIGH，既不屬於這個應用也不屬於它的任何相依，而且
> requirements.txt 改什麼都修不掉。runtime 不需要 pip，移掉之後歸零。順帶讓
> container 不再是一個 `pip install` 就能塞進沒審過的程式碼的環境。

---

## 3.0 相對 2.2 改了什麼

- **事件告警**：爆量會升級成有編號、有狀態、可交接的事件（`FAB-260901-001`），
  收斂時把結論整份存下來 —— log 一小時後會被清掉，事件不會
- **版面改成兩層**：清單頁回答「有什麼還沒處理」，點進去才是細節
- **示範模式**：`DEMO_MODE` 控制，這版寫死開啟。只有一個選項「切哪一條光纖」
- **搶救順序的排序依據改成四條明文規則**，畫面上逐條顯示實際數值（2.2 顯示的
  加權分數其實跟名次無關，方向還常常相反）
- **內建預設網換掉**：舊的 8 站範例只有 3/10 條切下去會開出事件；新的 9 站 /
  16 條 / 702 對，16/16 全部定位正確
- **修掉兩個偵測 bug**：安靜的網路上小規模斷纖永遠偵測不到；爆量視窗被 bin
  邊界截斷
- **修掉事件卡片少報站點組數**：「等 N 組」以前拿的是被截斷成 12 筆的顯示清單長度，
  一次影響 33 組站點的斷纖會顯示成「等 12 組」—— 而 12 是個看起來完全合理的數字，
  沒有任何地方會透露它其實是上限。現在總數與顯示清單分開存
- **主控台全中文化**
- **CVE 歸零**（移除 runtime pip）
- **改成 amd64 + arm64 雙架構**：2.2 之前只有 amd64，在 Apple Silicon 的 Mac 上
  `docker compose up -d` 會直接失敗（`no matching manifest for linux/arm64`），
  不是慢、是根本起不來。現在同一個 tag 兩種架構都在，Docker 自己挑

### 這一版的驗收紀錄（2026-09-01，拿要推出去的那顆 image 實跑）

| 檢查 | 結果 |
|---|---|
| 依這份 SOP 從零起服務 | `healthy`，約 15 秒 |
| 內建預設網 | 9 站 · 16 條光纖 · 702 對，不需匯入任何東西 |
| 切 `FIBER-0007` | 142 筆 log → `FAB-260901-001` 待處理，嫌疑 `FIBER-0007`（DC2 ↔ DC4）信心高 |
| 16 條逐條切 | **16/16 定位正確**（14 條信心高、`FIBER-0006` 與 `FIBER-0013` 信心中） |
| 認領 → 重複認領 → 結案 | 正常；重複認領回 409，不會靜靜成功 |
| `docker compose restart` | 事件與結案結論都在；`retained_events` 歸零 |
| 一個介面連彈 8 次 | 8 筆 log 收斂成 1 個 peer → 未達門檻，不開事件 |
| Trivy | 0 critical / 0 high / 0 medium / 0 low |
