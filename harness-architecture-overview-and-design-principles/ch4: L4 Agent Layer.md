# Ch4 · L4 Agent Layer

> 專職化 AI 執行任務 + 記憶能力
> L4 是最內層——在 L1 的安全邊界、L2 的 context 管理、L3 的流程控制之下，Agent 負責實際的推理和生成

---

## 本章涵蓋

```mermaid
graph LR
    subgraph L4["L4 · Agent Layer"]
        ROLE["角色分類"]
        DEF["Agent 定義"]
        HAND["交接介面"]
        DEBATE["對抗驗證"]
        PROMPT["Prompt 設計"]
        MEM["Memory (T0/T1/T2)"]
    end
```

| 模組 | 核心問題 |
|------|---------|
| 角色分類 | 什麼類型的 agent 該存在？怎麼分工？ |
| Agent 定義 | 一個 agent 需要哪些欄位才能精確執行任務？ |
| 交接介面 | Agent 之間怎麼傳遞工作成果？ |
| 對抗驗證 | 怎麼讓 agent 互相質疑來提升品質？ |
| Prompt 設計 | 怎麼寫 prompt 讓 agent 穩定產出高品質結果？ |
| Memory | Agent 怎麼記住身份、工作經驗和長期知識？ |

---

## 四類角色分類

### 為什麼要分類

如果所有 agent 都是「通用助手」，它們在 pipeline 中的職責邊界就是模糊的。模糊的職責導致重複工作、遺漏、以及「誰該負責這件事？」的問題。

角色分類的目的是讓每個 agent 有明確的專職——它只做一件事，並且把這件事做好。

### 四個類別

```mermaid
graph LR
    subgraph COLLECT["🔍 資料收集"]
        C["從外部世界獲取資料<br/>搜尋、爬取、API 查詢"]
    end
    subgraph REASON["🧠 分析推理"]
        R["理解資料、找出模式<br/>多視角分析、因果推論"]
    end
    subgraph CREATE["💡 創意發散"]
        CR["生成解決方案<br/>方案設計、評估、對比"]
    end
    subgraph QA["🎯 品質控制"]
        Q["驗證品質、質疑假設<br/>對抗驗證、事實查核"]
    end

    COLLECT --> REASON --> CREATE --> QA
```

| 類別 | 核心能力 | 典型 temperature | 典型 model tier |
|------|---------|-----------------|----------------|
| **🔍 資料收集** | 精準執行搜尋/API 呼叫，不添加推測 | 低（0.0-0.2） | 快速、便宜（Tier 3） |
| **🧠 分析推理** | 深度理解、模式識別、因果推論 | 中（0.3-0.5） | 高推理能力（Tier 1-2） |
| **💡 創意發散** | 探索多種可能性、產出新穎方案 | 較高（0.5-0.8） | 高推理能力（Tier 1） |
| **🎯 品質控制** | 嚴格邏輯驗證、找出漏洞和矛盾 | 低（0.0-0.2） | 高推理能力（Tier 1） |

**Temperature 的原則**：需要精準的用低 temperature（收集、驗證），需要多樣性的用高 temperature（創意、發散）。品質控制雖然需要高推理能力，但 temperature 要低——你不希望 Critic 在質疑時「有創意」。

**Model tier 的原則**：不是每個 agent 都需要最貴的模型。資料收集是結構化操作，用便宜快速的模型。分析和品質控制需要深度推理，用高推理模型。創意發散需要最好的模型以產出高品質的新穎方案。

> **範例**：一個 8 step pipeline 可能這樣分配：
> - 🔍 收集：Scout（搜尋）、Validator（去重驗證）
> - 🧠 分析：Analyst ×3（平行多視角）
> - 💡 創意：Ideator（方案生成）、Evaluator（可行性評估）
> - 🎯 品質：Critic（對抗驗證）、Investor Lens（壓力測試）

---

## Agent 定義

### 為什麼每個 Agent 需要完整定義

社群經驗反覆證明：**80% 的精力應該放在 Task 定義，20% 放在 Agent 定義。** 但這裡的 20% 不是隨便帶過——一個定義不完整的 agent 會在 pipeline 中產出不可預測的結果。

Agent 的定義就是它的「工作合約」。合約越精確，agent 的行為越可預測。

### Agent 定義模板

```
agent:
  id:             唯一識別碼
  name:           人類可讀的名稱
  category:       四類之一 (collector / reasoner / creator / quality_controller)
  
  identity:
    role:           一句話角色描述（你是什麼）
    goal:           這個 agent 存在的目的（你要達成什麼）
    backstory:      背景設定（為什麼你適合做這件事）
    constraints:    明確的行為限制（你不能做什麼）
  
  execution:
    model:          使用的 LLM 模型和版本
    temperature:    生成溫度
    max_tokens:     單次回應的 token 上限
  
  tools:
    allowed:        可使用的工具清單
    read_file:      可讀取的檔案範圍（有界自主，見 Ch3）
  
  memory:
    T0:             身份定義檔案路徑（SOUL.md / AGENTS.md）
    T1_write:       是否在 run 結束後寫入 T1 daily log
    T2_query:       是否在啟動時查詢 T2 長期知識
  
  output:
    format:         產出格式規範
    schema:         JSON schema 路徑（I/O 契約，見 Ch3）
    handoff_brief:  交接簡報格式規範
```

### 各欄位的設計指引

**identity 區塊**

| 欄位 | 做什麼 | 常見錯誤 |
|------|--------|---------|
| `role` | 定義 agent 是什麼 | 太模糊——「你是一個分析師」不如「你是一個專門從用戶反饋中提取關鍵問題的分析師」 |
| `goal` | 定義 agent 要達成什麼 | 沒有可衡量的標準——「做好分析」不如「產出至少 3 個有數據支撐的發現，每個附上原始引述」 |
| `backstory` | 建立 agent 的推理框架 | 太長——backstory 是設定推理傾向，不是寫小說。2-3 句足夠 |
| `constraints` | 定義 agent 不能做什麼 | 省略——沒有 constraints 的 agent 會自己發明行為邊界 |

**Goal 的設計是最重要的。** 一個好的 goal 包含：做什麼（動作）、品質標準（怎樣算好）、產出規格（格式和內容要求）。

```
❌ 弱 goal：
   "分析市場趨勢"

✅ 強 goal：
   "從驗證過的資料中識別 3-5 個市場趨勢，
    每個趨勢必須包含：
    (1) 趨勢描述（≤ 50 字）
    (2) 支撐數據（至少 2 個來源）
    (3) 對業務的影響評估（正面/負面/中性）"
```

**Constraints 的設計同樣關鍵。** Constraints 不是對 agent 的不信任，而是幫 agent 聚焦。告訴 agent「不要做什麼」和告訴它「要做什麼」一樣重要。

```
常見的 constraint 類型：
├── 範圍限制：「只分析提供的資料，不要搜尋新資料」
├── 格式限制：「只回傳 JSON，不要加解釋文字」
├── 行為限制：「不要對資料做推測，無法判斷時標記為 unknown」
├── 角色限制：「你是分析師不是決策者，提供分析但不做建議」
└── 成本限制：「每次搜尋最多 10 個查詢」
```

---

## 交接介面規格

### 為什麼需要標準化介面

Agent 之間的溝通方式直接影響 pipeline 的穩定性。如果每個 agent 用自己的格式產出結果，下游 agent 需要「理解」上游的格式——這是 LLM 的推測行為，不可靠。

標準化介面讓 agent 之間的資料傳遞變成確定性的：結構固定、欄位定義明確、可用程式碼驗證。

### 交接簡報 Schema

這是 Ch2 Initializer Pattern 中定義的交接簡報格式，在 L4 由每個 agent 負責填寫。

```json
{
  "step_id": "string",
  "agent_id": "string",
  "timestamp": "ISO8601",
  
  "conclusions": [
    {
      "statement": "string (≤ 50 words)",
      "confidence": "high | medium | low",
      "evidence_refs": ["string (file path or URL)"]
    }
  ],
  
  "key_data_points": [
    {
      "metric": "string",
      "value": "string | number",
      "source": "string"
    }
  ],
  
  "unresolved_issues": [
    {
      "issue": "string",
      "reason": "string",
      "suggested_action": "string"
    }
  ],
  
  "file_paths": {
    "full_output": "string (path)",
    "source_data": "string (path, optional)"
  }
}
```

**每個欄位都必須填寫。** Eval Gate 的第一道 deterministic check 就是驗證這個 schema——任何欄位為空即 fail。這確保了下游 agent 永遠收到完整的交接資訊。

### 交接介面與其他層的關係

```
L3 Pipeline     定義「哪個 step 的簡報傳給哪個 step」（routing）
L2 Context      Initializer 把簡報組裝進 agent 的 context（injection）
L4 Agent        Agent 負責「填寫」簡報（production）
L3 Eval Gate    驗證簡報的完整性和品質（validation）
```

Agent 不需要知道自己的簡報會被誰讀——它只需要按照 schema 填寫。Routing 和 injection 是上層的職責。

---

## 對抗驗證：Multi-Agent Debate

### 為什麼需要對抗驗證

單一 agent 做自我修正（self-reflection）有一個根本限制：LLM 傾向於在修正時維持原來的觀點。研究稱之為「思維固化」（Degeneration of Thought）——模型在自我反思時依賴同質的思考過程，不會真正從不同角度重新評估。

Multi-Agent Debate（MAD）通過讓多個 agent 從不同立場互相質疑，打破思維固化。研究顯示，多輪辯論能讓 LLM 修正彼此的錯誤、提升邏輯一致性。

### 對抗驗證的設計模式

在 pipeline 架構中，對抗驗證不是獨立的「辯論系統」，而是嵌入在 Feedback Loop（Ch3）中。

**模式 A：Producer-Critic 雙角色（最常用）**

```mermaid
graph LR
    P["Producer<br/>temperature: 0.5<br/>角色：生成內容"] -->|"產出"| C["Critic<br/>temperature: 0.1<br/>角色：質疑內容"]
    C -->|"✅ pass"| NEXT["→ 下一階段"]
    C -->|"❌ fail + feedback"| P
```

Critic 不是「評分員」——它是「反方律師」。它的 goal 是找出產出中的漏洞、邏輯矛盾和缺失的證據。

設計要點：
- Producer 和 Critic 使用不同的 temperature：Producer 較高（生成多樣性），Critic 較低（嚴格邏輯）
- Critic 的 prompt 明確要求「從反面質疑」，而非「評估品質」——措辭影響行為
- Critic 必須產出結構化反饋（見 Ch3 Critic 反饋結構），不能只說「不好」

**模式 B：Matrix Debate 多人格辯論**

多個 agent 各自代表不同立場，對同一份產出進行辯論。

```mermaid
graph TD
    INPUT["產出待驗證"]
    INPUT --> A["Persona A<br/>（例：樂觀主義者）"]
    INPUT --> B["Persona B<br/>（例：懷疑論者）"]
    INPUT --> C["Persona C<br/>（例：風險管理者）"]

    A -->|"論點"| JUDGE["Judge<br/>（綜合裁決）"]
    B -->|"論點"| JUDGE
    C -->|"論點"| JUDGE

    JUDGE --> VERDICT["裁決：pass / fail + 綜合分析"]
```

Matrix Debate 比 Producer-Critic 更重但品質更高。適用場景：

| 場景 | 使用模式 |
|------|---------|
| 日常品質控制（大部分 step） | Producer-Critic（簡單、快速、便宜） |
| 高風險決策（投資建議、策略方向） | Matrix Debate（多視角、更嚴格） |
| 事實查核（資料可信度存疑） | Matrix Debate + 工具查核 |

### Matrix Debate 的設計參數

| 參數 | 說明 | 建議值 |
|------|------|--------|
| `personas` | 參與辯論的角色數 | 3-5 個（太少沒有多樣性，太多成本過高） |
| `rounds` | 辯論輪數 | 1-2 輪（通常一輪就能暴露主要問題） |
| `judge_model` | 裁決者使用的模型 | ≥ debaters 的模型等級 |
| `diversity_strategy` | 確保 persona 之間足夠不同 | 給不同 persona 不同的推理方法指令 |

**多樣性是 debate 有效的前提。** 如果所有 persona 使用相同的推理方法，辯論會退化為「同一個觀點的重複」。研究建議給每個 persona 不同的推理策略（如：一個用正面論證、一個用反面質疑、一個用類比推理），而非僅僅分配不同的「性格」。

> **範例**：在產品策略評估中，Matrix Debate 可能使用：
> - Persona A（用戶代言人）：這個產品解決了真正的用戶痛點嗎？
> - Persona B（財務懷疑者）：單位經濟學撐得住嗎？競爭壁壘在哪？
> - Persona C（技術風險評估）：技術方案能在預算內落地嗎？
> - Judge：綜合三方論點，裁決產出的整體可信度。

---

## Prompt 組裝與跨層職責

### 80/20 法則

社群反覆驗證的經驗：**80% 的精力放在 Task 設計（agent 要做什麼），20% 放在 Agent 設計（agent 是什麼）。**

| 投入重點 | 具體做什麼 | 為什麼 |
|---------|-----------|--------|
| **Task（80%）** | Goal 定義、output schema、eval 標準、constraints | 這些決定了 agent 的「行為邊界」，是可驗證的 |
| **Agent（20%）** | Role、backstory、personality | 這些影響 agent 的「推理傾向」，是軟性的 |

大部分團隊把時間花在調 agent 的 backstory 和 personality，但真正影響產出品質的是 task 定義的精確度。Role、goal、backstory、constraints 各欄位的語意和設計指引見前面的「Agent 定義」段落。本段聚焦：**這些欄位怎麼組裝成完整 prompt，以及每一塊由哪一層負責提供。**

### 跨層組裝職責

Agent 在 prompt 組裝上是**被動的接收者**。它自己只控制 T0 身份定義，其他所有內容都是外層注入的。

```
Prompt 組裝——每一塊的來源和層級：

┌──────────────────────────────────────────────────────────────┐
│ 區塊                        │ 來源          │ 固定/動態     │
├──────────────────────────────────────────────────────────────┤
│ 1. Agent Identity (T0)      │ L4 定義，L2 注入  │ 固定（cache） │
│ 2. L1 Constraints (權限/預算)│ L1 Authority 注入 │ 固定（cache） │
│ 3. Output format + schema   │ L3 Step 定義      │ 固定（cache） │
│ 4. Eval criteria            │ L3 Step 定義      │ 固定（cache） │
│ 5. Context briefing (交接)  │ L2 Initializer    │ 動態          │
│ 6. T2 Memory query result   │ L2 查詢 L4 Memory │ 半固定（週更）│
│ 7. Task instruction         │ L3 Step 定義      │ 動態          │
└──────────────────────────────────────────────────────────────┘
```

```mermaid
graph TD
    L1["L1 Authority<br/>注入 constraints<br/>（權限/預算提醒）"]
    L2["L2 Initializer<br/>注入 context briefing<br/>+ T2 知識片段"]
    L3["L3 Pipeline<br/>注入 output format<br/>+ eval criteria<br/>+ task instruction"]
    L4["L4 Agent<br/>提供 T0 身份定義"]

    L4 --> PROMPT["完整 Prompt"]
    L1 --> PROMPT
    L2 --> PROMPT
    L3 --> PROMPT
    PROMPT --> EXEC["Agent 執行"]
```

**關鍵洞察**：Agent 自己控制的只有 T0（它是誰）。它被指派什麼任務（L3）、看到什麼 context（L2）、有什麼限制（L1）——全部由外層決定。這就是 Harness 架構的核心：**Agent 的行為由框架約束，不是由 Agent 自己決定。**

### Prompt 結構與 KV-Cache 的對齊

區塊 1-4（固定區）每次呼叫完全相同，命中 KV-Cache（Ch2），以 0.1× 價格計算。區塊 5-7（動態區）每次不同，全價計算。

固定區佔比越高，成本越低。因此：
- **不要在固定區放每次不同的內容**（如時間戳）
- **不要在動態區放每次相同的內容**（如 eval 標準）
- 固定區的內容順序不可隨意調整——任何順序變動都會導致 cache miss

### 常見組裝錯誤

| 錯誤 | 後果 | 修正 |
|------|------|------|
| 把 eval 標準放在動態區 | 每次 cache miss | Eval 標準是固定的，放在固定區 |
| 在 system prompt 開頭加時間戳 | 破壞整個固定區的 cache | 時間戳放在動態區最前面 |
| Backstory 太長（> 500 tokens） | 佔用 context window，推高成本 | 2-3 句足夠 |
| 沒有 output schema | 產出格式每次不同，下游無法解析 | 強制 JSON schema |
| 用自然語言描述格式 | Agent 會自由發揮 | 用 JSON schema + 範例 |
| 沒有標註 constraints 的來源 | Agent 無法區分哪些是硬限制 | L1 的 constraints 明確標記為「系統強制」 |

### Prompt 版控

Prompt 是 agent 行為的核心——改一個字可能改變整個產出。版控邏輯和 Ch3 的 Rubric 版控一致：

```
Prompt 版控流程：
├── 所有 agent 的 prompt 模板存放在版控系統中
├── 變更 prompt 時，用 golden set 跑 regression test
│   └── golden set = 一組已知輸入 + 預期產出品質
├── 記錄每個 prompt 版本對應的 eval pass rate
│   └── pass rate 下降 = prompt 變更有問題
└── 重大變更需要人類審核後才部署
```

---

## Memory：Agent 怎麼使用記憶

Memory 的生命週期管理（精煉流程、品質控制、注入時機）在 Ch2 已詳細說明。本段聚焦 **Agent 的視角**：agent 怎麼寫入記憶、怎麼從記憶中取得知識。

### Agent 與三層記憶的互動

| 層級 | Agent 的角色 | 由誰控制 |
|------|------------|---------|
| **T0 · 身份** | Agent 按照 T0 的定義行事 | L4 定義，L2 注入（Agent 是被動接收者） |
| **T1 · 工作記憶** | Agent 在 run 結束後寫入 T1 | L3 Pipeline 觸發，Agent 按 schema 填寫 |
| **T2 · 長期知識** | Agent 在 run 開始時讀取 T2 相關片段 | L2 Initializer 查詢並注入（Agent 是被動接收者） |

Agent 在記憶系統中**主動參與的環節只有 T1 寫入**。T0 是預先定義的，T2 是由 L2 自動注入的。

### 哪些 Agent 需要哪些記憶層

不是每個 agent 都需要所有三層記憶。

| Agent 類型 | T0 身份 | T1 工作記憶 | T2 長期知識 |
|-----------|---------|-----------|-----------|
| 資料收集型 | ✅ 需要（定義搜尋策略） | ❌ 通常不需要 | ⚠️ 看場景（如常用搜尋模式） |
| 分析推理型 | ✅ 需要（定義分析框架） | ✅ 需要（記住之前的分析經驗） | ✅ 需要（跨日知識累積） |
| 創意發散型 | ✅ 需要（定義創意方向） | ⚠️ 看場景 | ✅ 需要（避免重複產出） |
| 品質控制型 | ✅ 需要（定義評估標準） | ✅ 需要（記住常見錯誤模式） | ✅ 需要（永久約束來自 L1 SI） |

### T1 寫入：Agent 給自己留筆記

T1 的目的和交接簡報不同。交接簡報是給下游 step 的（「這是我的結論」），T1 是給自己未來版本的（「這是我學到的」）。

**觸發機制**：Pipeline 在 run 結束後觸發每個需要 T1 的 agent 執行一次 T1 寫入。這不是 agent 自己決定要不要寫——而是 L3 Pipeline 的確定性流程。

**T1 daily log schema**：

```json
{
  "run_id": "string",
  "agent_id": "string",
  "date": "ISO8601",
  
  "conclusions_for_self": [
    "string (和交接簡報不同——重點是「我學到什麼」)"
  ],
  
  "difficulties_encountered": [
    {
      "description": "string (遇到什麼困難)",
      "resolution": "string (怎麼解決的)"
    }
  ],
  
  "delta_from_previous": [
    "string (和上次 run 比有什麼新發現)"
  ],
  
  "suggested_improvements": [
    "string (建議系統層級的改善，供 L1 Self-Improvement 參考)"
  ]
}
```

**結構化 > 自由敘述**。結構化的 T1 讓 T1→T2 精煉（Ch2）能自動提取模式——從 `difficulties_encountered` 中找出重複出現的問題，從 `delta_from_previous` 中找出趨勢變化。自由敘述的 T1 則需要額外的 LLM 呼叫來解析，增加成本和不確定性。

### T2 讀取：Agent 怎麼取得相關知識

Agent 不直接查詢 T2——L2 Initializer 替它做。但匹配策略影響 agent 收到的知識品質。

| 方案 | 做法 | 適用時機 |
|------|------|---------|
| **Keyword match** | 從本次任務描述提取關鍵字，匹配 T2 條目的 tag | T2 條目 < 50 條（早期） |
| **Semantic search** | 將任務描述和 T2 條目都向量化，用 embedding 相似度匹配 | T2 條目 > 50 條（成熟期） |

起步建議：用 keyword match。它簡單、不需要向量化基礎設施、在條目少的時候足夠精準。當 T2 累積超過 50 條後，keyword match 會開始漏掉語意相關但用詞不同的條目——這時考慮遷移到 semantic search。

**不論哪種方案，T2 注入都不是全量的。** Initializer 只取最相關的 N 條（建議 N=3-5），控制 context 大小。Agent 收到的不是「所有記憶」，而是「和本次任務最相關的幾條知識」。

---

## 六個模組的協作

```mermaid
graph TD
    ROLE["角色分類<br/>決定 agent 類型"]
    ROLE --> DEF["Agent 定義<br/>role / goal / constraints / tools"]
    DEF --> PROMPT["Prompt 結構<br/>固定區 + 動態區"]
    PROMPT --> MEM["Memory 注入<br/>T0 身份 + T2 知識"]
    MEM --> EXEC["Agent 執行"]
    EXEC --> HAND["交接簡報<br/>結構化 schema"]
    HAND --> DEBATE["對抗驗證<br/>Critic 或 Matrix Debate"]
    DEBATE -->|"✅ pass"| NEXT["→ 下一個 Step（L3）"]
    DEBATE -->|"❌ fail"| PROMPT
```

**從外到內的完整路徑**：
1. L1 設定安全邊界（Authority）和監控（Observability）
2. L2 組裝乾淨的 context（Initializer + Filesystem + KV-Cache）
3. L3 決定執行順序和品質門檻（Pipeline + Eval Gate）
4. L4 的 Agent 在這個框架內做推理和生成
5. Agent 產出交接簡報，經過 Eval Gate 驗證
6. 通過後傳給下一個 step；不通過則退回重做

Agent 是 pipeline 中「唯一不確定的部分」——但它被四層確定性的框架包住，使得整體系統的行為是可預測、可監控、可改善的。

---

## 後續章節

- **Ch5 · 模型選型與成本策略** — 多供應商比較、Tier 分層、成本優化
- **Ch6 · 部署路線圖** — Phase 0-3、信任度演進、驗證標準

---

## References

**Agent 角色設計**
- [Crafting Effective Agents - CrewAI](https://docs.crewai.com/en/guides/agents/crafting-effective-agents) — 「80% 精力放在 Task 設計，20% 放在 Agent 定義」；role/goal/backstory 的設計指引
- [Building Effective Agents - Anthropic](https://www.anthropic.com/engineering/building-effective-agents) — Agent 的能力邊界、工具設計、prompt 結構化
- [Why Multi-Agent Systems Fail - Galileo](https://galileo.ai/blog/why-multi-agent-systems-fail) — 角色混淆（role confusion）是 multi-agent 系統最常見的失敗模式之一

**Multi-Agent Debate**
- [Tool-MAD: Multi-Agent Debate for Fact Verification](https://arxiv.org/html/2601.04742v1) — MAD 框架概覽、tit-for-tat debate 結構、多種協調策略（majority voting / argument diversification / deliberation trees）
- [Mitsubishi Electric: Multi-agent AI for Expert-level Decisions](https://us.mitsubishielectric.com/en/pr/global/2026/0120/) — 對抗生成（GAN 啟發）應用於 multi-agent debate，用於安全分析和風險評估
- [Diverse Multi-Agent Debate (DMAD) - OpenReview](https://openreview.net/forum?id=t6QHYUOQL7) — 「思維固化」問題的定義；不同推理方法比不同人格更能打破固化
- [Multi-Agent Debate — AutoGen](https://microsoft.github.io/autogen/stable//user-guide/core-user-guide/design-patterns/multi-agent-debate.html) — AutoGen 的 MAD 實作模式：多輪交換 + 基於反饋的精煉

**Prompt 設計**
- [Agent Engineering: IMPACT Framework](https://www.morphllm.com/agent-engineering) — Intent（goal 設計）作為六維度的第一個，決定 agent 行為的基礎
- [Lessons From 2 Billion Agentic Workflows - CrewAI](https://blog.crewai.com/lessons-from-2-billion-agentic-workflows/) — Expected output 的重要性；「取得最大成果的團隊善於把 task 定義精確到可驗證」
- [GAN-Inspired Multi-Agent Harnesses](https://medium.com/@gwrx2005/gan-inspired-multi-agent-harnesses-for-long-running-autonomous-software-engineering-architecture-37a8c2d59b6b) — Planner–Generator–Evaluator 三角色模式；temperature 分離策略

**Memory**
- [Using OpenClaw as a Force Multiplier](https://towardsdatascience.com/using-openclaw-as-a-force-multiplier-what-one-person-can-ship-with-autonomous-agents/) — 三層記憶實踐（SOUL.md / daily logs / MEMORY.md）
- [Context Engineering for AI Agents - Manus](https://manus.im/blog/context-engineering-for-ai-agents) — Filesystem-as-memory、agent 記憶與 context 管理的關係
