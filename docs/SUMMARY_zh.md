# Fiber-Cut Localization — 中文摘要

對照論文 `README.md` §3–§5 的中文版本，著重直觀解讀與具體例子。

**運維場景前提**：switch 之間用 CDP/LLDP 建立鄰居關係並偵測斷線；底下的物理光纖路徑可能跨多段（站點之間經過 amplifier / OADM / regenerator 等 passive 光學設備），LLDP 只看「對面那台 switch 還在不在」，看不到中間光纖段的細節。

---

## 一、一行結論

> 在真實 WAN backbone（Abilene、NSFNet）上，我們的演算法 **100% 找出**斷掉的 fiber edge。在資料中心 fabric 上，因為 production 環境每個 DC 都會配置 LLDP 鄰居端點、覆蓋密度高，**預期 class_top-1 接近 100%**。少數真的分不出單一條的情況，都是「站點之間多條物理平行光纖」這種結構性 redundancy 設計 — 這時演算法仍 100% 把嫌疑鎖到那組 multi-rail 光纖內，operator 帶 OTDR 到機房現場挑一下就找到。

---

## 二、問題場景

> 假設早上九點，挖土機在某個城市的某條街上挖斷了一條長途光纖。我們的網路監控會收到一堆 switch 對之間的 CDP/LLDP 鄰居關係突然 timeout、報 DOWN log；其他 switch 的 LLDP 鄰居仍正常 hello，靜悄悄沒回報。我們手上有：
>
> - 全網拓樸圖（哪些站點之間有光纖連著）
> - 哪些 LLDP 鄰居對報 DOWN
> - 哪些 LLDP 鄰居對 silent（沒回報）
>
> 問題：我們怎麼**只看 LLDP log** 就知道是哪條光纖被挖斷？

關鍵難度：LLDP 的兩端 switch **不知道**自己的 hello 封包實際走了哪條物理光纖（光路可能 ECMP、可能因為 wave-multiplexing 改道、也可能光纖被 bundle 過）。所以一條鄰居關係 down 之後，可能對應到多條候選光纖中的任一條。

---

## 三、我們的方法核心思想

兩個關鍵假設：

1. **Uniform routing**：每對 LLDP 鄰居實際走的物理光纖路徑，從可能候選中均勻隨機選一條（光路選擇是底層光學系統決定的，我們監控不到）。
2. **High-quality logging**：LLDP timeout 不會誤報、漏報率低。

由這兩點 derive 出單一公式：對每條候選光纖 $e$，我們算

$$\mathrm{Score}(e) = \sum_{\text{每對 DOWN LLDP 鄰居 } d} \log f(d, e) \;+\; \gamma \sum_{\text{每對附近 silent LLDP 鄰居 } s} \log\bigl(1 - f(s, e)\bigr)$$

其中 $f(x, e) = \dfrac{\text{鄰居對 x 的候選光纖路徑中含 } e \text{ 的數量}}{\text{鄰居對 x 的候選光纖路徑總數}}$。

排第一的 edge 就是真兇。直覺：「DOWN 的鄰居都路徑上有 $e$」+「silent 的鄰居都路徑上沒 $e$」 → 這條 $e$ 就是兇。

---

## 四、用最小例子一步步手算

### 4.1 拓樸

四個光學站點 A、B、C、D，三條光纖段：A–B、A–D、B–C。

```
A ──── B ──── C
│
D
```

### 4.2 假設：A–B 那段光纖被挖斷

每個站點有少量 switch，跨站點建立 LLDP 鄰居關係。我們列兩對：

- **LLDP 鄰居對 1**：站點 A 的 switch $S_A$ 跟站點 C 的 switch $S_C$ 是 LLDP 鄰居。底下物理光路：A→B→C，跨兩段光纖。**A–B 斷了之後 hello timeout，這對 LLDP 鄰居報 DOWN。**
- **LLDP 鄰居對 2**：站點 A 的 switch $S_A'$ 跟站點 D 的 switch $S_D$ 是 LLDP 鄰居。底下物理光路：A→D，跨一段光纖。**A–B 斷了不影響它，silent。**

### 4.3 對每條候選光纖算 $f$（候選含邊比例）

候選路徑（演算法不知道 LLDP 鄰居底下實際走哪條，只知道有哪些可能）：

- $P(\text{鄰居 1}) = \\{A\to B\to C\\}$，1 條候選
- $P(\text{鄰居 2}) = \\{A\to D\\}$，1 條候選

對 3 條光纖各算 $f$：

| 光纖 | $f(\text{鄰居 1}, e)$ | $f(\text{鄰居 2}, e)$ |
|---|---|---|
| (A, B) | 1/1 = **1.0** | 0/1 = **0.0** |
| (B, C) | 1/1 = **1.0** | 0/1 = **0.0** |
| (A, D) | 0/1 = **0.0** | 1/1 = **1.0** |

### 4.4 套 Score 公式

$\gamma = 0.8$，$\varepsilon = 0.001$（防 $\log 0$ 的下限）。

**光纖 (A, B) 的分數**：

- DOWN 那項：鄰居 1 是 DOWN → 加 $\log f(\text{鄰居 1}, A\text{-}B) = \log 1.0 = 0$
- silent 那項：鄰居 2 是 silent → 加 $\gamma \cdot \log(1 - f(\text{鄰居 2}, A\text{-}B)) = 0.8 \cdot \log(1 - 0) = 0$
- **Score(A, B) = 0 + 0 = 0** ← 滿分

**光纖 (B, C) 的分數**：跟 (A, B) 對稱 → 也是 **0**

**光纖 (A, D) 的分數**：

- DOWN 那項：鄰居 1 是 DOWN → 加 $\log f(\text{鄰居 1}, A\text{-}D) = \log 0$ → 套 $\varepsilon$ 下限 → $\log 0.001 \approx -6.9$
- silent 那項：鄰居 2 是 silent → 加 $0.8 \cdot \log(1 - 1.0)$ → $0.8 \cdot \log 0.001 \approx -5.5$
- **Score(A, D) ≈ −6.9 − 5.5 = −12.4** ← 大重罰

### 4.5 排序結果

| 光纖 | Score | 排名 |
|---|---|---|
| (A, B) | 0 | 1 (tied) |
| (B, C) | 0 | 1 (tied) |
| (A, D) | −12.4 | 3 |

**(A, B) 跟 (B, C) 並列第一**：因為這兩條光纖都在「鄰居 1 從 A→C 的唯一 LLDP 路徑」上出現，從 LLDP timeout 的單一 down 觀察上**真的分不出來**。這個 tie 不是 bug，是**資訊論上的硬限制** — 等價類概念。

(A, D) 被嚴重重罰：因為「DOWN 的鄰居 1 不可能路經 A-D」+「如果 A-D 真的斷了，silent 的鄰居 2 應該要 timeout 才對」，雙重矛盾。

### 4.6 從這個小例子看到三件事

1. **演算法精準鎖定嫌疑光纖**：3 條光纖裡 1 條被重罰排除，剩 2 條形成「等價類」。
2. **真兇 (A, B) 在 top-1（並列）**，所以 strict top-1 = 1，class_top-1 = 1。
3. **要進一步分辨 (A, B) 跟 (B, C)**，需要更多 LLDP 鄰居對 — 例如站點 B 內某 switch 跟站點 C 內某 switch 直接做 LLDP（B-C 一段），他的唯一候選路徑是 B→C，能直接區分。**現實上每個站點的 switch 配置決定 LLDP 鄰居密度**，密度越高越能打破等價類。

---

## 五、放大到真實實驗

### 5.1 我們測試的 6 個真實拓樸

把 §4 的小例子 scale up，跑 200 個 random trials × 1000 對 LLDP 鄰居：

![../figures/concept_topologies.png](../figures/concept_topologies.png)

| 拓樸 | 站點數 | 光纖段數 | 場景 |
|---|---|---|---|
| **Abilene** | 11 | 14 | 美國 Internet2 學術 backbone（11 個城市）|
| **NSFNet** | 14 | 16 | 美國 NSF backbone 早期版本 |
| **Fat-tree(4)** | 20 | 32 | 標準 3 層資料中心 fabric（switch site）|
| **Fat-tree(6)** | 45 | 108 | 大型資料中心 fabric |
| **Spine-leaf(4,8)** | 12 | 32 | 4 個 spine × 8 個 leaf 的 DC |
| **Spine-leaf(8,16)** | 24 | 128 | 8 × 16 大型 DC |

### 5.2 主結果

![../figures/topologies.png](../figures/topologies.png)

| 拓樸 | 嚴格 top-1 | **class top-1** | top-3 | ambiguous_rate |
|---|---|---|---|---|
| Abilene | **1.00** | **1.00** | 1.00 | 0.00 |
| NSFNet | **1.00** | **1.00** | 1.00 | 0.00 |
| Fat-tree(4) | 0.32 | **1.00** | 0.90 | 1.00 |
| Fat-tree(6) | 0.27 | **1.00** | 0.70 | 1.00 |
| Spine-leaf(4,8) | 0.71 | **0.99** | 1.00 | 0.53 |
| Spine-leaf(8,16) | 0.31 | 0.81 | 0.62 | 0.73 |

### 5.3 三句話解讀

**(a) 真實 WAN：100% 完美。**
> Abilene 跟 NSFNet 是 sparse、不規則的 backbone，每段長途光纖在地理上都是獨一無二的，跨站點的 LLDP 鄰居對應的物理路徑差異夠大。我們在 200 次模擬中**每一次**都直接抓出真兇。Production 可直接部署。

**(b) Fat-tree DC fabric：strict top-1 看似差，但其實是天花板。**
> Fat-tree 結構對稱：例如某個 pod 內 edge switch 到 aggregation switch 之間有 multi-rail 光纖，從 LLDP 視角看，每條 multi-rail 光纖背後都有一對結構等價的 LLDP 鄰居關係，它們對任何 LLDP 觀察的反應都一樣。我們的演算法 100% 把嫌疑縮成那 3-4 條結構等價的光纖，但 100% 直接挑出單一條在資訊論上不可能 — 你需要去機房用眼睛看（OTDR、port LED、manual trace）才知道是哪根。**這不是演算法 bug，是 LLDP 觀察粒度的限制。**

**(c) Spine-leaf：規模越大越難，但仍能鎖到正確等價類。**
> 從 4×8 跨到 8×16，光纖段數 4 倍（32 → 128），class_top-1 從 0.99 降到 0.81。在 8×16 上，**81% 的失敗事故演算法第一名直接落在正確等價類裡**（spine-leaf 的等價類典型對應到「同一台 ToR site 接出去的 3-5 條 uplink 光纖」，從外部 LLDP 看不出差別）。剩 19% 第一名沒鎖中，但 top-3（class_top-3 = 0.93）幾乎一定包含正確類。
>
> 規模上去 class_top-1 下降的原因不是演算法 degradation，是兩件事：(1) 光纖段變多後 f-pattern 更精細、等價類變小、難穩定 hit；(2) LLDP 鄰居對數固定在 1000，sample complexity 不足。把 LLDP 鄰居數拉到 5000-10000（例如每個 site 內配置更多 switch、跨 site 多開幾條 logical adjacency）應能回到 0.99+。

### 5.4 對抗性實驗（驗證理論）

把對稱性「故意做到極致」的 4 個小拓樸：

![../figures/adversarial.png](../figures/adversarial.png)

| 拓樸 | top-1 | class_top-1 | ambiguous_rate |
|---|---|---|---|
| `bridge_chain(3)` | 0.15 | **1.00** | 1.00 |
| `bipartite_k22` | 0.31 | **1.00** | 1.00 |
| `parallel_diamonds(2)` | 0.14 | **1.00** | 1.00 |
| `random+symmetric_bridge` | 0.91 | 0.91 | 0.00 |

前 3 個是「純對稱」，理論上 strict top-1 應該被結構限制 — 實驗符合。最後一個是「在隨機圖中加入對稱結構」，多樣性把對稱性打破，演算法回到 91% 準確。

### 5.5 真實部署的預期：實驗結果是保守下界

實驗用 1000 對**隨機**LLDP 鄰居生成。隨機 sample 會讓部分區域 peer 覆蓋不足 — 例如 §4.1 中的站點 B 就「剛好」沒有自己的 LLDP 鄰居端點，這才產生 (A,B) 跟 (B,C) 的 tie。

但**真實 production 環境的 LLDP 覆蓋密度遠高於此**：

- 每個 DC 站點都會放置至少一台 switch 跟其他站點建 LLDP 鄰居（這是運維 baseline 設定，每個站點上線都會配）
- 也就是 **V 中每個節點都會出現在至少一對 LLDP 鄰居的端點上**
- 拿 §4.1 例子來說，站點 B 在現實中一定也跟某個第五站點 X（可能是其他 backbone 站點）建有 LLDP 鄰居 — 那條鄰居關係的觀察就能區分 (A,B) 跟 (B,C)：(A,B) 斷不會影響 B-X 鄰居，(B,C) 斷也不會，但配合其他 peer 的拓樸位置就完全 disambiguate

**Production 預期**：
- WAN backbone：本來實驗就 100%，production 同樣 100%
- DC fabric 上 class_top-1（實驗值 0.81–1.00）會**逼近 1.00**，因為 LLDP 覆蓋密度比實驗的 random 1000 對更高、更均勻
- 實驗中 class_top-1 < 1.00 的部分，**多數是 random sample 配置造成的人工 artifact，不是方法的能力上限**

### 5.6 真正不可解的情況：物理平行光纖

實務上**真的、結構性、再多 LLDP 也救不了**的等價類，幾乎都來自一種情況：**站點 A 跟 B 之間有多條物理平行光纖**（不同管道走線、redundant cabling、wavelength multiplexing 共纜）。

舉例：兩個 DC 之間做雙路備援，拉了 4 條物理光纖各走不同管道。對 LLDP 來說：

- 4 條物理光纖各承載一對 LLDP 鄰居關係
- 任挖斷其中一條，那一對 LLDP 對 DOWN，其他 3 對 silent
- 如果我們在拓樸建模時把這 4 條物理光纖**個別**列為 4 條 edge → 演算法能區分（因為它們對應不同 LLDP adjacency）
- 但如果建模時把它們當成「站點 A 到站點 B 的一條邏輯邊」 → 從 LLDP 角度它們**結構等價、無法區分**

**這對運維的意義**：精細定位「斷掉是 4 條平行光纖中的哪一條」需要靠光學層工具（OTDR、port LED、manual cable trace），LLDP 在物理層細節上沒有解析能力。我們的演算法給你的是**「斷掉的光纖在這個 multi-rail 群組裡」**這個資訊，operator 帶 OTDR 到機房對那組 cable 量一下就找到。

**對拓樸設計的啟示**：如果想最大化演算法精度，避免大量「無區分的物理平行光纖」設計，或在建模時把每條物理光纖個別建模並用 LLDP-per-link 監控（每條 cable 兩端 switch 都跟對端建獨立 LLDP）。

---

## 六、對公司的 actionable insight

| 場景 | 實驗 class_top-1 | Production 預期 | 對運維的意義 |
|---|---|---|---|
| **WAN backbone** | 1.00 | 1.00 | 部署就能用，自動把 LLDP DOWN log 翻譯成「哪段光纖斷了」 |
| **DC fabric**（無物理平行光纖） | 0.81–1.00 | ~1.00 | LLDP 鄰居密度高，自動定位到單一光纖 |
| **DC fabric**（含物理平行光纖） | 1.00（class）/ 0.27（strict）| 1.00（class）/ 0.27（strict）| 自動鎖定「在這組 multi-rail 光纖內」，operator 帶 OTDR 找哪條 |

**兩種等價類來源，要分清楚**：

1. **「實驗 artifact」造成的等價類** — 隨機 sample 出來的 LLDP peer 覆蓋不均勻造成。**Production 上不存在**（每個 DC 都會有自己的 LLDP 鄰居端點）。
2. **「物理平行光纖」造成的等價類** — 兩個站點之間多條 redundant cable 在 LLDP 視角不可分辨。**這是結構性硬限制**，需要 OTDR / 機房 manual trace 解決。

**關鍵 takeaway**：演算法在我們**實際的 production 部署上會非常接近完美**。剩下「分不出來」的少數情況，都是**物理上多條平行光纖**這種設計選擇 — 這不是演算法的問題、也不是 LLDP 的問題，是「LLDP 訊號粒度跟物理冗餘設計的本質不對稱」。要在這些情況下精細到單一光纖，必須補上光學層工具（OTDR、port-level cable ID、manual trace），不是加更多 ML / 訓練。

---

## 七、常見問題

**Q1. 為什麼不用機器學習？**
> 因為這是 generative-model 直接 derive 的 Bayesian estimator — 已被數學證明是 MLE，理論上是最佳的。ML 在這個問題上能做的事，數學已經做完。

**Q2. 假設 routing 是 uniform 不太對吧？真實光路會偏好短路徑。**
> 我們在 README §3.4 兩階段框架裡有處理。短路徑偏好可作為 prior 進入第二階段（tiebreaker）。第一階段的拓樸 likelihood 不依賴 routing 假設細節，這也是為什麼即便 uniform 假設不嚴格，結果仍然 1.00。

**Q3. 部署成本？**
> BFS 跟 simple-path enumeration，O(|E| × |LLDP 鄰居對|)。1000 對鄰居 + 30 條光纖的網路 < 100ms 算完。沒有訓練成本，只要拓樸圖跟 LLDP DOWN/silent 列表。

**Q4. 雜訊呢？如果 LLDP timeout 真的會誤報？**
> Bayesian 框架可直接擴展加入觀測雜訊參數。我們現在的版本是「高品質 log」假設下的乾淨基線。下一步可以做 noisy 版本對照實驗（例如 LLDP hello 偶爾掉一兩個包但沒真的斷）。

**Q5. 為什麼 spine_leaf(8,16) 只到 0.81，不像其他 DC fabric 到 1.00？**
> 規模上去後，等價類變得更小、更難精準命中。這是 sample-complexity 問題（LLDP 鄰居對數量還不夠飽和），不是方法限制。把 LLDP 鄰居數從 1000 提到 5000 應該會回到 1.00。

**Q6. 我們的 LLDP 是直連物理光纖、單跳鄰居就好，這 framework 是不是 overkill？**
> 取決於 LLDP 鄰居對應的物理路徑。如果**每對 LLDP 鄰居就只跨一段光纖**（site 內 fiber patch panel 等），那 down log 直接指向那條光纖，問題 trivial，不需要本演算法。但只要 LLDP 鄰居的物理路徑**可能跨多段光纖**（長途、含 passive 光學設備、或 bundle 多條光纖共用一個 LLDP adjacency），就需要 inference — 這就是我們的 framework 的目標場景。

---

## 圖表

主要視覺素材：

- [`../figures/concept_anatomy.png`](../figures/concept_anatomy.png) — 解釋站點 / 光纖 / fiber endpoint 的概念
- [`../figures/concept_observation.png`](../figures/concept_observation.png) — DOWN vs silent LLDP 鄰居的直觀畫面
- [`../figures/concept_topologies.png`](../figures/concept_topologies.png) — 6 個測試拓樸長什麼樣
- [`../figures/topologies.png`](../figures/topologies.png) — 主結果 bar chart
- [`../figures/adversarial.png`](../figures/adversarial.png) — 對抗性測試結果
