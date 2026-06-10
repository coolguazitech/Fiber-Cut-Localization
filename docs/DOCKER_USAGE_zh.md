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
    coolguazi/fiber-cut-localizer:1.0
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

看到的畫面跟你 local dev mode 完全一樣 —— 左邊 React Flow topology canvas、
右邊 Topology Generator → Research Lab → Peer Inspector → Ranking。所有功能
（Generator slider、Cut simulate、Peer Inspector、path 高亮、Live Alarms tab、
ranking）都能用。

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
    coolguazi/fiber-cut-localizer:1.0
```

然後開 <http://localhost:9090/>（左邊改成你想用的 port、右邊永遠保持 `8000`，
因為 image 裡面 backend 固定聽 8000）。

---

## ⑦ 給別人連（server 模式）

如果你把 image 跑在一台跨網段 server 上，要讓組員從別的電腦連：

```bash
docker run -d --init --rm -p 0.0.0.0:8000:8000 \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:1.0
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

## ⑨ 用自家網路拓樸（不用內建 SW-A..SW-H 範例）

### 為什麼要這樣做

預設 image 內建一張範例網：8 個泛用 switch `SW-A..SW-H`、10 條 fiber、
12 個 baseline neighbor pair（純為了 demo 跟測試）。研究時你會想用：

- 你自己生成的拓樸（從 Topology Generator 拉 slider 生出來的），或
- 你真實 DC 的 fiber 佈線（多少機房、有哪些跨機房 fiber、平常有哪些
  OSPF/BFD/BGP adjacency）

把這些寫成一個 JSON 檔，**透過 docker volume mount 蓋掉 image 內建的那個**。

### 步驟 1：在 host 機器上準備一個 topology.json

放在你選的位置，例如 `/home/coolguazi/my_topology.json`。最小範例
（3 個機房、3 條 fiber 構成三角形、3 對 baseline neighbor）：

```json
{
  "nodes": [
    {"id": "DC-TPE",  "label": "DC-TPE",  "type": "switch", "x": 100, "y": 100},
    {"id": "DC-TCH",  "label": "DC-TCH",  "type": "switch", "x": 400, "y": 100},
    {"id": "DC-KHH",  "label": "DC-KHH",  "type": "switch", "x": 250, "y": 350}
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
| `nodes[]` | `id`（unique 字串）、`label`、`type`、`x`、`y` | 一個機房 / 容器。`x`、`y` 是 UI 上的座標（建議 0–800 之間挑容易看清楚的數字） |
| `edges[]` | `id`（unique）、`source`、`target` | 一條實體光纖。`source`、`target` 必須是 `nodes[].id` 之一 |
| `edges[]` | `length_km`（option，預設 1.0） | fiber 長度，給 length-prior 用 |
| `baseline_neighbors[]` | `a`、`b` | 一對「平常**應該**有 OSPF/BFD/BGP adjacency」的 device-pair。可以重複（同一對 container 多次 = 多個 device pair 共用這個 container channel） |

> 注意：兩個 container 之間可以有**多條**平行 fiber edge（不同 `id` 但同樣
> `source`/`target`），這正是論文允許的 model。

### 步驟 2：mount 進 container

```bash
docker run -d --init --rm \
    -p 8000:8000 \
    -v /home/coolguazi/my_topology.json:/app/data/topology.json:ro \
    --name fiber-cut-localizer \
    coolguazi/fiber-cut-localizer:1.0
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
docker run -d --init --rm -p 8000:8000 --name fcl coolguazi/fiber-cut-localizer:1.0

# 啟動（自家拓樸）
docker run -d --init --rm -p 8000:8000 \
    -v $PWD/my_topology.json:/app/data/topology.json:ro \
    --name fcl coolguazi/fiber-cut-localizer:1.0

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
| Tag | `1.0` / `latest` |
| Platform | `linux/amd64` |
| Size | 41 MB |
| Base | `python:3.12-alpine` |
| CVE | 0 critical / 0 high / 0 medium / 0 low |
| Runtime user | `app` (uid 10001) |
| Exposed port | 8000 |
| Healthcheck | 內建（`python urllib` 戳 `/health`） |
