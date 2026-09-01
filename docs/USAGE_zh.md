# 使用與驗收手冊

一步一步把系統跑一遍，每一步都寫清楚**該看到什麼**。看到不一樣的東西就停下來，
不要往下走 —— 後面每一步都建立在前一步的結果上。

每一步都給兩種驗證方式：**畫面上**（給操作員看的）跟 **curl**（給你確認後端真的
做了那件事，而不是畫面畫得像有做）。

底下假設服務跑在 `http://localhost:8000`。

> **原型程式碼不在這個 repo。** 這裡只放論文與文件；`prototype/` 是另外一份私有
> 的原始碼。手上沒有那份的話，走 Docker image 那條路即可 ——
> 見 [`DOCKER_USAGE_zh.md`](DOCKER_USAGE_zh.md)，第 ⓪ 節「本機開發」以外的每一步
> 都一模一樣。

---

## 這套系統現在的四個階段

```
① log 進來        目前只有兩條路：示範模式，或 POST /api/alarms/ingest
                  （撈 log 的 API 還沒接，介面留著了 —— 見第 ⑫ 節）
        ↓
② 爆量偵測        log 速率突然拉高的那一段
        ↓
③ 事件            有編號、有狀態、可交接。收斂時把結論存下來
        ↓
④ 定位與派工      斷的是哪一條光纖、先救哪幾對設備
```

---

## ⓪ 起服務

### 本機開發

```bash
cd prototype/backend
DEMO_MODE=1 .venv/bin/uvicorn app.main:app --reload --port 8000
```

前端另開一個 terminal（或直接用 `STATIC_DIR` 讓後端一起服務靜態檔）：

```bash
cd prototype/frontend
npm run dev          # http://localhost:5173
```

### Docker

```bash
cd prototype
cp .env.example .env      # 網管 API 位址與憑證填在這裡，其他先不用動
docker compose up -d
docker compose logs -f
```

`.env` 裡預設 `DEMO_MODE=1`、網管 API 留白 —— 也就是「示範／驗收模式」。
正式部署要改的東西見第 ⑪ 節。

**預期**：

```bash
curl -s localhost:8000/health
# {"status":"ok","mode":"prod","forwards_to":null}
```

---

## ① 檢查開機狀態：光纖圖與 peer 關係從哪來

**沒有設定網管 API 時，系統用內建的預設網開機**，不需要匯入任何東西。

畫面上：頂端那條狀態列會顯示 `9 站 · 16 條光纖 · 702 對鄰居`。

```bash
curl -s "localhost:8000/api/topology/status?product=FAB" | python3 -m json.tool
```

**預期**：

| 欄位 | 值 | 意思 |
|---|---|---|
| `fiber.kind` | `seed` | 光纖圖來自內建預設值（匯入過的話會是 `imported`） |
| `fiber.site_count` | `9` | |
| `fiber.fiber_count` | `16` | |
| `baseline.kind` | `embedded` | peer 關係夾在光纖圖檔案裡（接了網管會是 `nms`） |
| `baseline.pair_count` | `702` | |

```bash
curl -s "localhost:8000/api/topology/setup?product=FAB"
# stage 應為 "ready"
```

> **為什麼不用先匯入設備清單**：設備清單唯一的用途是告訴網管「要問哪些設備」。
> 沒接網管就沒有東西要問，這時候還卡著要求匯入，是在要一個給了也不會有作用的檔案。

---

## ② 開始收集

畫面上：如果直接進到設定精靈，按最下面的 **Start collecting logs**。已經在主畫面
的話，狀態列左邊會有一顆綠點寫著「收集中」。

```bash
curl -s -X POST "localhost:8000/api/topology/setup/collecting?product=FAB" \
     -H 'content-type: application/json' -d '{"collecting":true}'
```

**預期**：`"stage": "collecting"`、`"accepting_logs": true`。

> **沒開的話**：送進來的 log 會被**拒收**（不是靜靜丟掉）。這是刻意的 —— 設定還沒
> 完成卻讓 log 靜靜消失，操作員會以為訊號通了而畫面壞了。

---

## ③ 切一條光纖，讓 log 進來

畫面上：事件清單頁上方那條**黃色斜紋**的「示範資料」列 → 開啟控制台 →
下拉選一條光纖（或「隨機挑一條」）→ **✂ 切下去**。

```bash
curl -s -X POST "localhost:8000/api/demo/burst?product=FAB" \
     -H 'content-type: application/json' -d '{"cut_edge_id":"FIBER-0007"}'
```

**預期**：

- 回傳 `"last_scheduled": 142`（不同光纖數字不同，內建預設網大約 90–200 之間）
- 約 **3–5 秒**送完，期間 `"bursting": true`
- 送完後 `retained_events` 等於送出的筆數

```bash
curl -s "localhost:8000/api/demo/status?product=FAB"
```

> **切同一條永遠得到同一批 log**（種子由光纖 id 推出）。示範講到一半重切一次，
> 畫面上會是同一批設備、同一個答案，聽的人才能把兩次看到的東西對起來。

> **黃色斜紋是刻意的**。螢幕上同時有真實資料跟捏造資料時，唯一能防止混淆的
> 東西就是它們看起來不一樣。正式環境把 `DEMO_MODE` 留白，這條列與整組
> `/api/demo` 路由都不存在（不是隱藏，是 404）。

---

## ④ 檢查事件有沒有開出來

**最後一筆 log 之後要再等 25 秒**（`ALERT_SETTLE_SECONDS`），事件才會從
「進行中」收斂成「待處理」。所以從按下去到清單上出現定案的事件，大約 **30 秒**。

畫面上：事件清單會出現一列。

```bash
curl -s "localhost:8000/api/alerts?product=FAB" | python3 -m json.tool
```

**預期**：

| 欄位 | 值 |
|---|---|
| `counts.PENDING` | `1` |
| `alerts[0].alert_id` | `FAB-260901-001` 這種格式（單位－日期－當日流水） |
| `alerts[0].status` | 前 25 秒 `OPEN`，之後 `PENDING` |
| `alerts[0].event_count` | 跟送出的筆數一致 |
| `alerts[0].down_peer_count` | 同上（沒有 flap，所以一樣） |

**中途檢查**（送完之後、25 秒靜默之前查，大約按下去 8–20 秒之間）：`status`
應為 `OPEN`、`actions_computed` 為 `false` —— 派工順序只在收斂時算一次並存
起來，進行中不算。偵測器每 5 秒掃一次，所以按下去頭幾秒清單還會是空的。

---

## ⑤ 檢查定位對不對

這一步是整套系統的重點。你在第 ③ 步**知道自己切了哪一條**，系統不知道。

```bash
curl -s "localhost:8000/api/alerts?product=FAB" | python3 -c "
import json,sys;a=json.load(sys.stdin)['alerts'][0]
print('嫌疑:', a['top_suspect']['fiber_edge_id'], a['top_suspect']['source'],'↔',a['top_suspect']['target'])
print('信心:', a['confidence'])
print('第二:', a['runner_up'] and a['runner_up']['fiber_edge_id'])"
```

**預期**：`top_suspect.fiber_edge_id` **等於你切的那一條**。

內建預設網 16 條光纖逐條切過，**16/16 都定位正確**（2026-09-01 於 3.0 image 複驗）。
其中 14 條 `confidence` 是 `strong`，`FIBER-0006` 與 `FIBER-0013` 是 `moderate` ——
**定位仍然正確**，只是那兩條的次名咬得比較近。信心低不等於答錯，它講的是這張網
在那個位置的可識別性，不是系統有沒有故障。

> **`confidence` 讀法**：它是**相對強度，不是機率**。分級來自 top-1 領先 top-2 的
> log-likelihood 差距除以觀測數。之所以不顯示百分比，是因為早期用 softmax 分數
> 當信心時，204 次模擬裡答錯的那一次分數是 0.99 —— 全場最高。門檻目前只用一次
> 失誤校準過，等真實事故累積夠多對錯案例要重新擬合。

---

## ⑥ 檢查派工順序

畫面上：點事件那一列 → 進細節頁 → 右側「搶救順序」。

```bash
curl -s "localhost:8000/api/alerts?product=FAB" | python3 -c "
import json,sys;a=json.load(sys.stdin)['alerts'][0]
print('受影響設備對:', a['device_pair_count'], '· 無替代路徑:', a['blocked_pair_count'])
for x in a['actions']:
    print(f\"  {x['device_a']} ↔ {x['device_b']}  {x['dc_a']}↔{x['dc_b']}  \"
          f\"{x['down_count']}/{x['total_count']} 斷  \"
          + ('無替代路徑' if x['blocked'] else f\"{x['alt_path_count']} 條可繞\"))"
```

**預期**：`actions` 有 5 筆（收斂時存下來的前 5 名）。

排序是**四條明文規則**，不是加權分數。四條同時作用，強弱不同 —— 就像 Excel
從第 4 欄往回排到第 1 欄，最後規則 1 的條件最強、規則 4 最弱：

1. hostname 不含 `ADM` 的排在含 `ADM` 的前面
2. 該對 peer 的網管邏輯連線總數越多越重要
3. **斷掉數 / 總數的比例越小越重要**
4. 沒有替代光纖路徑的越重要

不是「滿足規則 1 就不看後面三條」—— 規則 1 相同才是常態（多數 hostname 都不含
ADM），所以實際在排順序的通常是 2、3、4。

> **規則 3 看起來反直覺，值得確認一次**：分母是網管的 last-seen 連線數，而網管
> 一天才爬一次，所以挖斷當下分母還是舊資料。比例小代表這對 peer 原本連線很多、
> 目前只斷了一小部分 —— 還有餘裕但正在流失，比早已全斷的更值得先搶。

---

## ⑦ 檢查認領、結案、歷史

畫面上：事件那一列右邊有「認領」「結案」。結案會跳出一個填處理結果的輸入框。

```bash
ID=FAB-260901-001
curl -s -X POST "localhost:8000/api/alerts/$ID/ack?product=FAB" \
     -H 'content-type: application/json' -d '{"by":"老王"}'
# 預期 status 變 ACKED、acked_by 為「老王」

curl -s -X POST "localhost:8000/api/alerts/$ID/ack?product=FAB" \
     -H 'content-type: application/json' -d '{}'
# 預期 409「只有進行中或待處理的事件可以認領」—— 兩個人同時處理同一件事時，
# 「我按了但沒有變」需要被說出來，不能靜靜成功

curl -s -X POST "localhost:8000/api/alerts/$ID/close?product=FAB" \
     -H 'content-type: application/json' -d '{"by":"老王","resolution":"已接續，10:20 復原"}'
# 預期 status 變 CLOSED

curl -s -X POST "localhost:8000/api/alerts/$ID/reopen?product=FAB" -d '{}'
# 預期退回 PENDING（不是 OPEN —— OPEN 是「還在進」，那是偵測器的判斷，不是人能宣告的）
```

**重啟後歷史還在**：

```bash
docker compose restart          # 或重啟 uvicorn
curl -s "localhost:8000/api/alerts?product=FAB&status=CLOSED"
```

**預期**：結過案的事件連同「誰結的、結論是什麼」都還在。事件歷史存在
`/data`（compose 裡的 `fcl-data` volume），是這套系統少數**不能重建**的東西。

---

## ⑧ 檢查誤報有沒有被擋掉

只有一兩條鄰接在彈（設備重啟、線卡不穩）**不應該**開出事件。

```bash
# 同一對設備的同一個介面連彈 8 次。
# 這兩個 hostname 取自內建預設網 —— 換成自己的網路時要改成實際存在的名字，
# 否則會因為「認不得這台設備」而被擋下，那是另一種原因，
# 證明不到門檻有沒有在運作。
for i in $(seq 1 8); do
  curl -s -X POST "localhost:8000/api/alarms/ingest?product=FAB" \
    -H 'content-type: application/json' -d "{
      \"alarm_id\":\"FLAP-$i\",
      \"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
      \"device\":\"dev-dc1-022\",\"neighbor\":\"dev-dc2-013\",
      \"interface\":\"Eth3/25\",\"alarm_type\":\"INT_DOWN\",\"severity\":\"minor\"}" > /dev/null
  sleep 0.2
done
sleep 30
curl -s "localhost:8000/api/alerts?product=FAB" | python3 -c "
import json,sys;d=json.load(sys.stdin);print('counts', d['counts'])"
```

**預期**：`SUPPRESSED` 加一、`PENDING` 不變，而且那一筆的 `down_peer_count`
是 **1** —— 八筆 log 收斂成一條邏輯鄰接。這正是門檻在看的東西。

**預設清單看不到它**，但
`counts.SUPPRESSED` 有數字，清單下方會寫「另有 N 筆未達門檻」，切到
「未達門檻」分頁就翻得出來，也能手動撈回待處理。

> **為什麼不直接丟掉**：操作員看到時間軸上有柱子而告警清單是空的，會得到
> 「這系統會漏報」的結論，而那是最難查的一種不信任。

> **門檻是「斷線 peer 數」不是「log 筆數」**（預設 3）。同一個介面彈跳二十次是
> 二十筆 log 但只有一個 peer —— 用 log 數當門檻等於把 flap 當成事故，
> 正好是要擋的東西。

---

## ⑨ 檢查事件活得比 log 久

```bash
curl -s -X POST "localhost:8000/api/demo/clear?product=FAB"   # 只清 log
curl -s "localhost:8000/api/alerts?product=FAB"
```

**預期**：`retained_events` 歸零，但事件清單**原封不動**。

這是整套告警存在的主要理由之一：log 預設只留一小時
（`EVENT_RETENTION_SECONDS`），而事後檢討要看的是「當時我們判斷是哪一條」。
事件卡片上的結論是**收斂那一刻存下來的快照**，不是事後重算的。

細節頁最上面那條 context bar 顯示的就是那份快照，**不隨時間軸移動**。
底下拖動視窗重算出來的答案跟它一不一樣，一眼看得到。

---

## ⑩ 檢查三個單位互相獨立

FAB / OA / DC 是**三張不同的網路**，不是同一張的篩選。

```bash
for p in FAB OA DC; do
  echo -n "$p: "; curl -s "localhost:8000/api/alerts?product=$p" | python3 -c "import json,sys;print(json.load(sys.stdin)['counts'])"
done
```

**預期**：剛剛在 FAB 產生的事件**不會**出現在 OA 或 DC。

畫面上切換右上角的分頁時，整個畫面會重掛 —— 保留前一個單位的資料在畫面上，
等於拿一張網路的 log 配另一張網路的圖，沒有任何一個瞬間那是有用的。

---

## ⑪ 正式部署：接上網管 API

改 `prototype/.env` 這一個檔案就好：

```bash
NMS_BASELINE_URL=https://your-nms.internal/dcim/devices/
NMS_AUTH_TOKEN=<你的 token>
DEMO_MODE=                 # ← 留白！正式環境不要有捏造 alarm 的按鈕
```

```bash
docker compose up -d        # 重新套用
```

**`NMS_BASELINE_URL` 這一個變數決定部署處於哪個階段：**

| | 留白 | 有填 |
|---|---|---|
| 光纖圖 | 內建預設值 | 匯入的那份（`imported` 優先於一切） |
| peer 關係 | 光纖圖夾帶的 | 每天向網管撈一次 |
| 對外連線 | 完全沒有 | 每天一次全掃 |
| 設備清單 | 不需要 | **必須匯入** —— 系統靠它知道要問網管哪些設備 |

接上網管之後的第一次設定：

1. 畫面上匯入**光纖圖**（哪些站點之間有實體光纖）
2. 匯入**設備清單**（要問網管哪些設備）—— 匯入後會自動開始掃描
3. 等掃描完成（狀態列會顯示進度），確認 peer 對數合理
4. 開始收集

其餘可調的環境變數見 `.env.example`，每一條都寫了為什麼。

**掃描時間**：預設一天一次（`NMS_POLL_INTERVAL=86400`），在 `refresh_at`
設定的本地時間跑，預設 03:00 —— 對齊上游網管通常也是一天爬一次。掃描期間仍然
用上一份 peer 關係服務，不會出現半套的網路。

---

## ⑫ 還沒做的：撈 log 的 API

**現況**：log 只有兩條路進來 —— 示範模式，或直接 `POST /api/alarms/ingest`。

**介面已經定好了**（`backend/app/sources/log_source.py`）：

```python
class LogSource(Protocol):
    def fetch(self, cursor: Optional[str], limit: int) -> LogBatch: ...
    def describe(self) -> str: ...
```

接真實來源時**只有 `HttpLogSource.fetch` 要寫**，其餘不動。目前那個 class 存在
但呼叫會直接丟 `NotImplementedError` —— 一個回傳空陣列的假實作，跟一張安靜的
網路長得一模一樣，主控台會就這樣看起來很健康地坐在那裡。

> **為什麼是 cursor 不是時間戳**：斷纖時幾百筆 log 擠在同一秒內，用
> `ts > last_seen` 過濾會丟掉跟上一筆同時間戳的那些，往另一邊挪則會重複。
> 而漏掉的 log 會被公式讀成「這條沒斷」的證據 —— 正好在最不能漏的時候漏。

`.env.example` 裡有一條註解掉的 `LOG_SOURCE_URL`，是預留的位置，現在填了不會生效。

---

## 一頁速查

```bash
# 起服務（示範模式）
cd prototype && cp .env.example .env && docker compose up -d

# 健康檢查
curl -s localhost:8000/health

# 開始收集
curl -s -X POST "localhost:8000/api/topology/setup/collecting?product=FAB" \
     -H 'content-type: application/json' -d '{"collecting":true}'

# 切一條光纖
curl -s -X POST "localhost:8000/api/demo/burst?product=FAB" \
     -H 'content-type: application/json' -d '{"cut_edge_id":"FIBER-0007"}'

# 等 30 秒後看事件
curl -s "localhost:8000/api/alerts?product=FAB" | python3 -m json.tool

# 認領 / 結案
curl -s -X POST "localhost:8000/api/alerts/<ID>/ack?product=FAB"   -d '{"by":"你的名字"}'
curl -s -X POST "localhost:8000/api/alerts/<ID>/close?product=FAB" -d '{"resolution":"已接續"}'

# 清 log（事件會留著）
curl -s -X POST "localhost:8000/api/demo/clear?product=FAB"
```

---

## 出問題時先看這三個

| 症狀 | 先確認 |
|---|---|
| 按了切光纖但什麼都沒發生 | `setup?product=` 的 `accepting_logs` 是不是 `true`。false 的話 log 會被拒收 |
| log 有進來但事件沒開 | 等滿 30 秒了嗎（送完 + 25 秒靜默）。還有 `counts.SUPPRESSED` 是不是加了 —— 斷線數沒到 3 條 |
| 事件開了但嫌疑是別條 | 看 `confidence`。`weak` 代表前兩名幾乎分不出來，那是網路本身的可識別性問題，不是系統故障 |
