# Fiber Cut Localizer — Docker 使用 SOP

本文檔說明如何用 Docker image 在 Linux x86 機器上跑這個 prototype，
畫面跟 local dev 一樣。

Docker Hub repo：<https://hub.docker.com/r/coolguazi/fiber-cut-localizer>

---

## 前置：你的機器要有 docker

```bash
docker --version   # 確認有
```

沒有的話：

- Linux：`curl -fsSL https://get.docker.com | sh`
- Mac：裝 Docker Desktop
- Windows：裝 Docker Desktop（用 WSL2 backend）

---

## ① 啟動（最簡單版，用內建範例 topology）

```bash
docker run -d --init --rm \
    -p 8000:8000 \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:2.0
```

逐字解釋：

- `-d` 背景跑（你可以關掉 terminal）
- `--init` 讓 container 內 PID 1 處理 zombie process / signal（不加會有小機率 ctrl+C 殺不乾淨）
- `--rm` container 停掉時自動清掉
- `-p 8000:8000` 把 host 的 8000 port 對應到 container 的 8000 port
- `--name fiber-cut-localizer` 給 container 一個好記的名字

第一次 run 會自動 `docker pull` 那個 image（41 MB，約 10 秒下載）。

## ② 開瀏覽器

打開 <http://localhost:8000/>

畫面跟 local dev mode 完全一樣。預設進入 **Timeline** 分頁：

```
┌──────────────────────────────────────────────────────────────┐
│ IFDOWN 密度帶狀條（橫跨整個寬度，中央有固定選取窗）              │
├───────────────────────────────────┬─┬────────────────────────┤
│                                   │▏│ Probable cut FIBER-...  │
│                                   │▏├────────────────────────┤
│          拓樸圖                     │▏│ Top 10 suspects        │
│          (React Flow)             │▏├────────────────────────┤
│                                   │▏│[Components][Rest.][Log]│
└───────────────────────────────────┴─┴────────────────────────┘
                                     ↑ 左右拖曳調整比例
```

操作邏輯只有一句話：**帶狀條中央那個窗框住的時間區間，就是演算法拿來定位的
DOWN 集合 D**。窗移到哪，排名就跟著重算。

- 拖帶狀條移動時間；拖窗的左右邊緣改變窗寬；滾輪縮放視野
- 右上 `LIVE` 把窗釘在最新的 log 上；一拖拉就自動脫離
- 帶狀條上淡色底帶是系統自動偵測到的 burst，點一下直接 snap 過去
- 橘色的 bar 是 unmapped（設備不在 baseline N 裡）—— 會顯示但不進 D

另一個分頁 **Research Lab** 是原本的研究介面（Topology Generator、Cut
simulate、Peer Inspector、Ranking），全部保留。

> **給第一次試的人**：內建那張 8 機房 / 15 對 peer 的範例網**太稀疏**，
> 演算法在上面本來就常常猜錯，別誤以為壞了。先切到 Research Lab 分頁，用
> Topology Generator 拉出 26 節點 / 700 對 peer 左右的拓樸，再切回 Timeline
> 就會看到它穩定命中。或者直接照 ⑨ 掛上你自己的拓樸。

## ③ 確認跑得起來（option，自我健檢）

```bash
curl http://localhost:8000/health
# 預期回 {"status":"ok"}
```

## ④ 看 backend log

```bash
docker logs -f fiber-cut-localizer
# ctrl+C 退出 follow
```

## ⑤ 停掉

```bash
docker stop fiber-cut-localizer
```

因為剛剛加了 `--rm`，停掉同時 container 也清掉了，下次要重啟直接跑 ① 的指令。

---

## ⑥ 換 port（如果 8000 被佔走）

```bash
docker run -d --init --rm -p 9090:8000 \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:2.0
```

然後開 <http://localhost:9090/>（左邊改成你想用的 port、右邊永遠保持 `8000`，
因為 image 裡面 backend 固定聽 8000）。

---

## ⑦ 給別人連（server 模式）

如果你把 image 跑在一台跨網段 server 上，要讓組員從別的電腦連：

```bash
docker run -d --init --rm -p 0.0.0.0:8000:8000 \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:2.0
```

差別只是把 host 端從預設 `127.0.0.1` 換成 `0.0.0.0`（聽所有網卡）。
組員開 `http://<server-ip>:8000/`。

---

## ⑧ 升級新版

```bash
docker pull coolguazi/fiber-cut-localizer:latest
docker stop fiber-cut-localizer
docker run -d --init --rm -p 8000:8000 \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:latest
```

---

## ⑨ 用自家網路拓樸（不用內建 DC1..DC8 範例）

### 為什麼要這樣做

預設 image 內建一張範例網：8 個容器（datacenter）`DC1..DC8`、10 條跨機房
fiber、12 個 baseline 邏輯鄰居 pair（純為了 demo 跟測試）。研究時你會想用：

- 你自己生成的拓樸（從 Topology Generator 拉 slider 生出來的），或
- 你真實的 fiber 佈線（多少機房、機房之間有哪些實體光纖直連、平常
  有哪些跨機房 logical adjacency）

注意層次：**節點 = 機房（容器）**，**peer 是機房內的設備（switch 等）之間的邏輯鄰居**，
**fiber 是機房之間的實體光纖**。

把這些寫成一個 JSON 檔，**透過 docker volume mount 蓋掉 image 內建的那個**。

### 步驟 1：在 host 機器上準備一個 topology.json

放在你選的位置，例如 `/home/coolguazi/my_topology.json`。最小範例
（3 個機房、3 條 fiber 構成三角形、3 對 baseline neighbor）：

```json
{
  "nodes": [
    {"id": "DC-TPE",  "label": "DC-TPE",  "type": "container", "x": 100, "y": 100},
    {"id": "DC-TCH",  "label": "DC-TCH",  "type": "container", "x": 400, "y": 100},
    {"id": "DC-KHH",  "label": "DC-KHH",  "type": "container", "x": 250, "y": 350}
  ],
  "edges": [
    {"id": "F-TPE-TCH", "source": "DC-TPE", "target": "DC-TCH", "label": "TPE-TCH", "length_km": 8.5},
    {"id": "F-TCH-KHH", "source": "DC-TCH", "target": "DC-KHH", "label": "TCH-KHH", "length_km": 6.2},
    {"id": "F-TPE-KHH", "source": "DC-TPE", "target": "DC-KHH", "label": "TPE-KHH", "length_km": 9.0}
  ],
  "baseline_neighbors": [
    {"a": "DC-TPE", "b": "DC-TCH"},
    {"a": "DC-TCH", "b": "DC-KHH"},
    {"a": "DC-TPE", "b": "DC-KHH"}
  ]
}
```

欄位定義：

| 區塊 | 必填欄位 | 意義 |
|------|----------|------|
| `nodes[]` | `id`（unique 字串）、`label`、`type`、`x`、`y` | 一個**容器**（datacenter），裡面實際上裝著 switch / router 等 device。`type` 用 `container`。`x`、`y` 是 UI 上的座標（建議 0–800 之間挑容易看清楚的數字） |
| `edges[]` | `id`（unique）、`source`、`target` | 一條**機房之間的實體光纖**。`source`、`target` 必須是 `nodes[].id` 之一 |
| `edges[]` | `length_km`（option，預設 1.0） | fiber 長度，給 length-prior 用 |
| `baseline_neighbors[]` | `a`、`b` | 一對「平常**應該**有 logical adjacency」的 peer。Adjacency 是兩個機房內 device 之間建立的（typically OSPF/BFD/BGP/IS-IS 等任一種協定），跟我們 paper 的物理光纖路徑層**不同層**。可以重複（同一對機房之間有多對 device 邏輯鄰居就重複放） |

> 注意：兩個 container 之間可以有**多條**平行 fiber edge（不同 `id` 但同樣
> `source`/`target`），這正是論文允許的 model。

### 步驟 2：mount 進 container

```bash
docker run -d --init --rm \
    -p 8000:8000 \
    -v /home/coolguazi/my_topology.json:/app/data/topology.json:ro \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:2.0
```

關鍵那一行 `-v`：

- 左邊 `/home/coolguazi/my_topology.json` 是 **host** 上你那份 JSON 的路徑
  （請改成你實際的絕對路徑）
- 右邊 `/app/data/topology.json` 是 **container 裡** image 內建那份 sample
  的位置（這個固定，照抄）
- 結尾 `:ro` 表示 read-only（container 不能寫回 host）

### 步驟 3：開瀏覽器確認

<http://localhost:8000/> → 應該會看到你定義的 nodes（例如上面範例就是
`DC-TPE`、`DC-TCH`、`DC-KHH` 三角形），右邊 Topology Generator 的
"now: X nodes · Y fibers" 顯示你 JSON 的數字。

### 隨時切回內建範例

按 Topology Generator 的 **"Reset to sample"** 按鈕。但其實這個 reset 是讀
container 裡 `/app/data/topology.json`，由於你 mount 上去了，reset 也只會
回到你那份 JSON。要回到內建 sample 就**停掉 container 重跑，不要再加 `-v`**。

---

## 一頁速查

```bash
# 啟動（內建範例）
docker run -d --init --rm -p 8000:8000 --name fcl coolguazi/fiber-cut-localizer:2.0

# 啟動（自家拓樸）
docker run -d --init --rm -p 8000:8000 \
    -v $PWD/my_topology.json:/app/data/topology.json:ro \
    --name fcl coolguazi/fiber-cut-localizer:2.0

# 用瀏覽器
open http://localhost:8000/

# 看 log
docker logs -f fcl

# 停掉
docker stop fcl

# 升級
docker pull coolguazi/fiber-cut-localizer:latest
```

---

## Image 規格快速參考

| 項目 | 值 |
|------|------|
| Repository | `coolguazi/fiber-cut-localizer` |
| Tag | `2.0` / `latest`（同一個 digest） |
| Digest | `sha256:fdef9f895964006adfa2c5b8651bbd0133e0e1a18d3a66e259eeecae6f4d5eed` |
| Platform | `linux/amd64` |
| Size | 39 MB 下載 / 174 MB 解壓後佔磁碟 |
| Base | `python:3.12-alpine`（Alpine 3.24.1、Python 3.12.13）|
| CVE | 0 critical / 0 high / 0 medium / 0 low（Trivy 0.69 與 Docker Scout 雙掃，2026-08-04）|
| Runtime user | `app` (uid 10001)，非 root |
| Exposed port | 8000 |
| Healthcheck | 內建（`python urllib` 戳 `/health`） |

要自己複驗 CVE：

```bash
trivy image --platform linux/amd64 coolguazi/fiber-cut-localizer:2.0
docker scout cves --platform linux/amd64 coolguazi/fiber-cut-localizer:2.0
```

### 2.0 相對 1.2 改了什麼

- 新增 **Timeline** 分頁並取代原本的 Live Alarms：操作者自己拖時間窗決定 D，
  而不是由系統的 incident 視窗替你決定
- 新增 `/api/timeline/density`、`/api/timeline/select`、`/api/timeline/bursts`
- log 型別支援 `INT_DOWN`（兩端 hostname 的邏輯鄰居關係）
- scoring engine 內部改寫成稀疏累加 + 路徑列舉快取，26 節點 / 915 對 peer
  的完整 joint MLE 從數十秒降到約 0.16 秒，拖曳時才有辦法即時重算
- 修正同一組設備之間有多條不同 interface 鄰居關係時，D 被錯誤合併的問題
