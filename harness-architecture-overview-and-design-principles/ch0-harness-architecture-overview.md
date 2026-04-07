# Ch0 · Harness Architecture 總覽與設計原則

> Multi-Agent 高品質系統的架構設計指南
> 通用四階段骨架，可套用於產品發現、競品監控、投資研究等場景

---

## 為什麼是 Harness，不是 Agent

2025 年大家在比誰的 Agent 更聰明。2026 年的共識是：**Model 是商品，Harness 才是護城河。**

同一個模型（Claude、GPT、Gemini）在不同的 Harness 下表現天差地別。Manus 用相同的基礎模型重寫了五次 Harness，每次都提升了可靠性——Meta 最終以約 20 億美元收購的不是模型，是那個 Harness。Vercel 移除了 80% 的 Agent 工具，結果反而更好。

Agent 是大腦，Harness 是身體。沒有身體的大腦什麼也做不了。

```mermaid
graph LR
    MODEL["🧠 Model<br/>理解需求 · 推理 · 生成文字<br/>─────<br/>可替換的商品"]
    HARNESS["🛡️ Harness<br/>環境感知 · 工具編排<br/>錯誤恢復 · 安全約束<br/>─────<br/>真正的護城河"]
    PRODUCT["✅ 可靠的 Agent 產品"]

    MODEL --> PRODUCT
    HARNESS --> PRODUCT
```

---

## 五層嵌套架構

架構由外到內分五層。外層包住內層，提供約束和服務。

### Fig A · Layer relationships
![圖片描述](https://github.com/minghan-git/harness-architecture/blob/main/harness-architecture-overview-and-design-principles/harness_architecture_v5.svg)

### 圖 B · Pipeline 通用流程
```mermaid
graph LR
    subgraph S1["1 · Collect"]
        direction TB
        C1["Scout"]
        C2["Validator"]
        C1 --> C2
    end

    subgraph S2["2 · Analyze"]
        direction TB
        A1["Multi-perspective<br/>(parallel)"]
        A2["Critic"]
        A1 --> A2
    end

    subgraph S3["3 · Create"]
        direction TB
        CR1["Ideator"]
        CR2["Evaluator"]
        CR1 --> CR2
    end

    subgraph S4["4 · Deliver"]
        direction TB
        D1["Writer"]
        D2["Report"]
        D1 --> D2
    end

    S1 -->|"⚡ eval"| S2
    S2 -->|"🤖 eval"| S3
    S3 -->|"📏 eval"| S4
    S4 -->|"🎯 eval"| CMD["Commander"]

    S2 -.->|"❌ ≤3"| S1
    S3 -.->|"❌ ≤2"| S2
```


**Eval legend**: ⚡ Deterministic check · 🤖 LLM-as-Judge · 📏 Business metrics · 🎯 Goal success rate

### 場景套用範例

同一個四階段骨架，不同場景只是替換每個 Agent 的具體任務：

| 階段 | iGaming 產品發現 | 競品情報監控 | 投資標的研究 |
|------|-----------------|-------------|-------------|
| **收集** | 爬 X/Reddit 用戶抱怨 | 爬競品官網/更新日誌 | 爬 SEC 財報/新聞 |
| **驗證** | 交叉比對來源、去重 | 比對歷史版本差異 | 驗證數據與官方來源一致 |
| **分析 A** | 市場規模分析 | 功能差異分析 | 財務健康度分析 |
| **分析 B** | 用戶痛點根因分析 | 定價策略分析 | 產業趨勢分析 |
| **分析 C** | 風險評估 | 技術架構推測 | 風險因子分析 |
| **Critic** | 質疑痛點是否真實 | 質疑分析是否有偏見 | 質疑估值假設 |
| **創造** | 產品解決方案設計 | 反制策略設計 | 投資論述撰寫 |
| **評估** | TAM/單位經濟學 | 實施成本/ROI | 估值模型/退出策略 |
| **交付** | 商業計劃報告 | 競品週報 | 投資備忘錄 |

**Eval 圖例**：⚡ 確定性檢查 · 🤖 LLM-as-Judge · 📏 業務指標 · 🎯 目標達成率

---

## 各層職責速覽

```mermaid
graph LR
    subgraph LAYERS["由外到內"]
        direction TB
        L1x["<b>L1 · Harness Shell</b><br/>保護 Agent 不闖禍<br/>保護系統不崩潰"]
        L2x["<b>L2 · Context Engine</b><br/>控制 Agent 看到什麼<br/>控制成本不爆炸"]
        L3x["<b>L3 · Pipeline</b><br/>控制流程走向<br/>控制品質門檻"]
        L4x["<b>L4 · Agent Layer</b><br/>做實際的思考和生成<br/>每個 Agent 一個專職<br/>T0/T1/T2 記憶能力"]
        L1x --> L2x --> L3x --> L4x
    end
```

| 層級 | 一句話職責 | 包含的組件 |
|------|----------|-----------|
| **L1** Harness Shell | Agent 的安全邊界和運行環境 | Authority、Observability、Error Recovery、Self-Improvement |
| **L2** Context Engine | 控制每個 step 的 context 輸入 | Initializer Pattern、Filesystem-as-Memory、KV-Cache 策略、T1→T2 精煉 |
| **L3** Pipeline | 確定性流程編排 + 品質門檻 | Step 順序定義、Eval Gate（四種類型）、退回迴圈 |
| **L4** Agent Layer | 專職化 AI 執行任務 + 記憶能力 | Agent 角色/prompt/工具/模型定義、**Memory (T0/T1/T2)** |
| **外部** | 人類與交付 | Commander、交付通道（Telegram/Notion/Slack） |

---

## 組件層級歸屬總表

快速查找「某個組件在哪一層」。

| 組件 | 層級 | 說明 |
|------|------|------|
| Authority (權限模型) | **L1** | Model 決定做什麼 ≠ System 允許做什麼 |
| Observability (可觀測性) | **L1** | Token/cost 監控、動作追溯、失敗聚類 |
| Error Recovery (錯誤恢復) | **L1** | API 失敗→重試→escalate 人類 |
| Self-Improvement (自我改善) | **L1** | 錯誤→永久約束、build for deletion |
| Initializer Pattern | **L2** | 每 step 乾淨 context 注入，切斷 rot 傳播 |
| Filesystem-as-Memory | **L2** | 完整產出寫檔，摘要進 context |
| KV-Cache Prompt 結構 | **L2** | 穩定前綴 + append-only + 確定性序列化 |
| T1→T2 精煉週期 | **L2** | 跨日記憶壓縮，防長期 context rot |
| Step 順序定義 | **L3** | YAML/Flows 確定性控制 |
| Eval Gate | **L3** | 每 step 出口的品質/格式門檻 |
| 退回迴圈 (品質) | **L3** | Critic fail → 退回重做（有上限） |
| 退回迴圈 (錯誤) | **L1** | 工具失敗 → 重試 → escalate（屬 Harness 層） |
| Agent 角色定義 | **L4** | 專職化角色 + temperature + 模型層級 |
| T0 identity / T1 working memory / T2 long-term knowledge | **L4** | Agent's memory — identity, daily logs, distilled knowledge |
| Human-in-the-Loop | **外部** | Commander 審閱、高風險操作審批 |
| 交付通道 | **外部** | Telegram / Notion / Slack |

---

## 設計原則

### 原則 1 · 確定性引擎控制流程，Agent 只做創意工作

```mermaid
graph LR
    subgraph WRONG["❌ 反模式"]
        W1["Manager Agent<br/>(LLM)"] -->|"自己決定<br/>下一步去哪"| W2["Agent A"]
        W1 -->|"自己決定<br/>要不要重試"| W3["Agent B"]
        W1 -->|"自己決定<br/>何時結束"| W4["Agent C"]
    end

    subgraph RIGHT["✅ 正確模式"]
        R0["YAML/Flows<br/>確定性引擎"] -->|"Step 1"| R1["Agent A"]
        R0 -->|"Step 2"| R2["Agent B"]
        R0 -->|"Step 3"| R3["Agent C"]
    end
```

讓 LLM 決定流程走向，在小專案可以，在大型系統中會跳步驟、循環依賴、卡在分析迴圈。Agent 擅長在有界問題內生成內容，但對流程排序的元層級決策力很差。

### 原則 2 · 專職化 Agent，不要萬能 Agent

```mermaid
graph LR
    subgraph WRONG2["❌ 反模式"]
        WA["一個 Agent<br/>研究+分析+創意+評估<br/>什麼都做"]
    end

    subgraph RIGHT2["✅ 正確模式"]
        RA["Scout<br/>只做收集"]
        RB["Analyst<br/>只做分析"]
        RC["Critic<br/>只做批判"]
        RD["Ideator<br/>只做創意"]
        RA --> RB --> RC --> RD
    end
```

專職化 agent 組合在一起，永遠比一個萬能 agent 效果好。同一份資料餵給不同視角的 analyst，產出比單一分析師更全面。

### 原則 3 · 每個 Step 嵌入 Eval Gate

```mermaid
graph LR
    subgraph WRONG3["❌ 反模式：品質集中在最後"]
        S1x["Step 1"] --> S2x["Step 2"] --> S3x["Step 3"] --> QA["品質檢查"]
    end

    subgraph RIGHT3["✅ 正確模式：每步都有門檻"]
        S1y["Step 1"] --> E1y["Eval ⚡"] --> S2y["Step 2"] --> E2y["Eval 🤖"] --> S3y["Step 3"] --> E3y["Eval 📏"]
    end
```

垃圾進、垃圾出。如果 S1 收集了假資料，沒有 Eval Gate，這份假資料會一路傳到 S8 最終報告。每個 step 出口攔截，越早發現問題成本越低。

四種 Eval 類型按需選配：

| 類型 | 符號 | 成本 | 適用場景 |
|------|------|------|---------|
| 確定性檢查 | ⚡ | 零 | 格式、數量、來源 URL 驗證 |
| LLM-as-Judge | 🤖 | 中 | 邏輯一致性、品質評分 |
| 自定義業務指標 | 📏 | 低 | 領域特定 KPI（TAM 可驗證？競品 ≥ 3？） |
| 目標達成率 | 🎯 | 中 | 最終任務是否完成、是否可執行 |

### 原則 4 · Model 決定做什麼 ≠ System 允許做什麼

```mermaid
graph TB
    AGENT["Agent 想做某件事"]
    AGENT --> AUTH_CHECK{"L1 Authority<br/>允許嗎？"}
    AUTH_CHECK -->|"白名單內"| EXECUTE["執行"]
    AUTH_CHECK -->|"需要審批"| HUMAN["→ 人類確認"]
    AUTH_CHECK -->|"禁止"| BLOCK["攔截 + 記錄"]
    HUMAN -->|"批准"| EXECUTE
    HUMAN -->|"拒絕"| BLOCK
```

Agent 的能力和權限是兩回事。Scout 有能力發推文（它能呼叫 X API），但 Authority Layer 不允許它這樣做。所有外部 API 呼叫必須在白名單內，付費操作需要人類批准，這兩層檢查在架構上和 Agent 的推理分離。

### 原則 5 · 減法優於加法

```mermaid
graph LR
    subgraph WRONG5["❌ 起步就建 8 agent"]
        W5A["Scout"] --> W5B["Validator"] --> W5C["Analyst×3"]
        W5C --> W5D["Critic"] --> W5E["Ideator"] --> W5F["Biz Eval"]
        W5F --> W5G["Investor Lens"] --> W5H["Report Writer"]
    end

    subgraph RIGHT5["✅ 先驗證最小 pipeline"]
        R5A["Scout"] --> R5B["Analyst"] --> R5C["Critic"]
    end

    RIGHT5 -->|"驗證通過後<br/>逐步擴展"| WRONG5
```

Vercel 移除 80% 工具反而更好。Manus 花俏的子 agent 路由被結構化交接取代。先用 3 個角色跑通，驗證「能從 X 上挖到真實 iGaming 痛點」，再逐步加 agent。每次新增都要問：「這個 agent 解決了什麼 3 agent 版本解決不了的問題？」

### 原則 6 · Build for Deletion

```mermaid
graph TB
    STEP["每個 Step / Agent"]
    STEP --> Q1{"存在理由<br/>還成立嗎？"}
    Q1 -->|"模型進步了<br/>單 agent 能做到"| DELETE["合併或移除"]
    Q1 -->|"Eval 數據顯示<br/>仍然需要"| KEEP["保留"]
    DELETE --> SIMPLER["更簡單的架構<br/>更低的成本"]
```

每段 Harness 邏輯都應有過期日。如果下一代模型能在沒有你的鷹架下完成某件事，就刪掉鷹架。每個 step 標註「存在理由」和「移除條件」，模型升級時定期重新評估。

### 原則 7 · 信任是漸進建立的

```mermaid
graph LR
    P1["Phase 1<br/>100% 人審<br/>建立信任"]
    P2["Phase 2<br/>50% 抽檢<br/>驗證信任"]
    P3["Phase 3<br/>異常才介入<br/>維持信任"]

    P1 -->|"數千次執行<br/>品質自證"| P2
    P2 -->|"Eval 數據穩定<br/>信任度達標"| P3
```

不要第一天就全自動。從 100% 人工審查起步，用 eval 數據驅動降低人工介入比例。社群經驗是改善速度比預期快——花幾週而非幾個月。

---

## IMPACT Checklist — 每個 Agent 上線前必查

```mermaid
graph LR
    I["🎯 Intent<br/>成功長什麼樣？<br/>有 eval 量化嗎？"]
    M["🧠 Memory<br/>讀 T0/T1/T2 哪些？<br/>寫入什麼？"]
    P["📝 Planning<br/>執行前輸出計劃？<br/>計劃可被編輯？"]
    A["🔐 Authority<br/>工具白名單？<br/>哪些需審批？"]
    C["🔀 Control Flow<br/>Pipeline 哪個位置？<br/>退回迴圈上限？"]
    T["🔧 Tools<br/>最小工具集？<br/>有降級方案？"]

    I --> M --> P --> A --> C --> T
```

| 維度 | 層級歸屬 | 核心問題 |
|------|---------|---------|
| 🎯 **Intent** | L3 (Eval) | 這個 step 的 pass/fail 標準是什麼？ |
| 🧠 **Memory** | L4 | 需要讀寫哪些記憶層？ |
| 📝 **Planning** | L3 | Agent 執行前會輸出可審閱的計劃嗎？ |
| 🔐 **Authority** | L1 | 工具白名單和人類審批點定義了嗎？ |
| 🔀 **Control Flow** | L3 | 在 pipeline 的位置、退回迴圈上限？ |
| 🔧 **Tools** | L1 + L4 | 是最小工具集嗎？失敗時有降級方案嗎？ |

---

## 本章小結

```
一句話總結：

  Harness（L1 security）
    wraps Context（L2）
    wraps Pipeline（L3）
    wraps Agent + Memory（L4）
    Commander retains final control

  Model 是可替換的零件。
  Harness 是你的產品。
```

**後續章節**：Ch1-Ch5 分別展開每一層的詳細設計，Ch6 處理模型選型，Ch7 是部署路線圖。
