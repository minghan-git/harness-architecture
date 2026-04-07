# Harness Architecture — 總結

---

## 已完成的文件

| 文件 | 主題 | 核心產出 |
|------|------|---------|
| **Ch0** | Overview | 四層架構（L1-L4）、同心圓 SVG 圖 A、Pipeline 流程圖 B、七條設計原則 |
| **Ch1** | L1 Harness Shell | Authority（三態決策 + 降級策略）、Observability（三類指標 + alert P1-P4）、Error Recovery（雙路徑 + backoff + partial failure）、Self-Improvement（雙軌觸發 + 約束衝突） |
| **Ch2** | L2 Context Engine | Context Rot 定義與傳播、Initializer Pattern（交接簡報）、Filesystem-as-Memory（分流邏輯）、KV-Cache 三規則、T1→T2 精煉 |
| **Ch3** | L3 Pipeline | Step 定義（YAML schema + output_schema + timeout）、三種通訊模式（+merge 策略 + quorum）、Eval Gate 四類型（+ check 組合邏輯 + 校準機制）、退回迴圈 |
| **Ch4** | L4 Agent Layer | 四類角色、Agent 定義模板、交接 JSON schema、MAD 對抗驗證、Prompt 跨層組裝、Memory 使用（T1 寫入 schema + T2 匹配策略） |
| **Memory** | Memory Architecture | 六種腐化模式、Arbiter 衝突解決、記憶所有權、Memory flush 不變式、檢索三階段演進、OpenClaw 對應 |
| **Ch5** | 模型選型與成本 | 三 Tier、六供應商（含 GPT-5.3/5.4）、三套配置方案、成本疊加計算、Eval 驅動選型流程 |

---

## 關鍵架構決策記錄

| 決策 | 選擇 | 原因 |
|------|------|------|
| 層級數量 | 四層（L1-L4），刪除 L5 | Memory 是 Agent 的記憶能力，不是獨立層 |
| Memory 歸屬 | L4 Agent Layer | 記憶屬於 agent，不屬於基礎設施 |
| 檔案讀取權限 | 做法 C：Pipeline 預載 + Agent 可選讀取 | 確定性保底 + 有界自主 |
| 圖 A 格式 | SVG 同心圓 | Mermaid 無法畫同心圓，參考原圖風格 |
| 圖 B 格式 | Mermaid | 線性流程是 Mermaid 強項 |
| 流程控制 | 確定性引擎，非 LLM 決定 | LLM 的元層級決策力不可靠 |
| Eval 門檻觸發 | 雙軌（頻率 + 嚴重度） | 低頻高損失的問題不應等三次 |
| Prompt 組裝 | Agent 被動接收，四層注入 | Agent 只控制 T0 身份，其餘由框架決定 |

---

## 實施 Checklist

### Phase 0 · 基礎建設（第 1-2 週）

**L1 Harness Shell**
- [ ] 定義全域約束（預算上限、forbidden actions 清單）
- [ ] 為每個 agent 建立獨立的工具白名單
- [ ] 建立 Observability：選擇工具（Langfuse / Braintrust）、定義 span 格式
- [ ] 設定 alert 路由：P1 即時推送、P2 每日匯總、P3 週報
- [ ] 建立 Error Recovery：backoff 參數（2s/×2/30s max）、circuit breaker
- [ ] 建立 Failure Registry（空表，準備記錄）

**L2 Context Engine**
- [ ] 設計交接簡報格式（JSON schema，≤ 500 tokens）
- [ ] 建立檔案系統結構（/outputs/run_{id}/、/memory/T1/、/memory/T2/）
- [ ] 設計 prompt 結構：固定區（T0 + constraints + format + eval）vs 動態區
- [ ] 確認 KV-Cache 三規則：穩定前綴、append-only、確定性序列化（sort_keys）

**L3 Pipeline**
- [ ] 定義每個 step 的 YAML（input 三層、output schema、eval、recovery、dependencies）
- [ ] 設定每 step 的 timeout_seconds
- [ ] 定義通訊模式：哪些 step sequential、哪些 parallel、哪些有 feedback loop
- [ ] 定義 Eval Gate：每 step 至少一個 deterministic check

**L4 Agent Layer**
- [ ] 為每個 agent 寫 identity（role、goal、backstory、constraints）
- [ ] 建立 T0 檔案（SOUL.md / AGENTS.md）
- [ ] 確認 goal 包含可衡量標準和產出規格
- [ ] 確認 constraints 明確列出「不要做什麼」

**模型選型**
- [ ] 全部 step 先用同一個 Tier 2 模型（推薦 Sonnet 4.6）
- [ ] 確認 LLM-Judge 的模型 ≥ 被評估 agent 的模型

### Phase 1 · 跑通與校準（第 3-4 週）

**執行**
- [ ] 跑 10+ 次完整 pipeline run
- [ ] 收集每 step 的 eval pass rate
- [ ] 收集成本數據（per step、per run）
- [ ] 記錄所有 escalation 到 Failure Registry

**Eval 校準**
- [ ] 檢查 pass rate < 70% 的 step → 是 eval 太嚴還是 agent 能力不足？
- [ ] 檢查 pass rate > 95% 的 step → eval 太鬆？可以降級模型？
- [ ] 確認 deterministic checks 沒有 false positive/negative
- [ ] LLM-Judge 的 rubric 和 golden set 版控

**模型調整**
- [ ] pass rate < 70% 的 step → 升級到 Tier 1
- [ ] pass rate > 95% 的 step → 試降級到 Tier 3
- [ ] 降級後再跑 10 次驗證（pass rate 變化 < 5% 才確認降級）

**Memory**
- [ ] 確認 T1 daily log 按 schema 寫入
- [ ] 確認 memory flush 在 compaction 前正常觸發
- [ ] 手動精煉第一批 T1 → T2

### Phase 2 · 優化與自動化（第 5-8 週）

**成本優化**
- [ ] 啟用 prompt caching（確認固定區 cache hit rate > 60%）
- [ ] 評估 Batch API（非即時場景可用）
- [ ] 確認 tiered routing 配置穩定

**Self-Improvement**
- [ ] 審查 Failure Registry → 任何 pattern ≥ 3 次？
- [ ] 將重複模式轉為永久約束（eval rule / prompt update / tool config / authority policy）
- [ ] 審查 Build for Deletion → 任何 step 可以簡化或移除？

**Memory 自動化**
- [ ] 設定 T1→T2 自動精煉（每 7 天或 T1 > N 條時觸發）
- [ ] 設定精煉品質檢查（頻率門檻、一致性檢查、可量化要求）
- [ ] 如果 T2 > 50 條，評估從 keyword match 升級到 hybrid search

**Eval 演進**
- [ ] 設定 eval pass rate 監控 alert（持續 2 週 > 95% 或 < 50%）
- [ ] 月度 meta-eval：人工抽樣審核 eval 判定
- [ ] Rubric regression test 流程建立

### Phase 3 · 擴展與成熟（第 9 週+）

**擴展**
- [ ] 多供應商策略：設定備用供應商 + 自動 fallback
- [ ] 新場景：複製 pipeline 骨架，替換 step 的具體任務
- [ ] 團隊擴展：dashboard 分享、alert 路由到正確的人

**持續改善循環**
- [ ] Weekly：Failure Registry 審查
- [ ] Monthly：Build for Deletion 審查 + 成本趨勢 + eval 校準
- [ ] On model upgrade：重跑 eval suite，檢查配置是否需要調整
- [ ] Quarterly：整體架構審查——有沒有新的社群最佳實踐值得引入？

---

## 文件間的交叉引用地圖

```
Ch0 Overview
 ├── 圖 A (SVG) → Ch1-Ch4 的層級關係
 ├── 圖 B (Mermaid) → Ch3 Pipeline 骨架
 └── 七原則 → 貫穿所有章節

Ch1 L1 Harness Shell
 ├── Authority 降級策略 → Ch4 Agent 工具清單
 ├── Observability span 格式 → Ch2 Tracing
 ├── Error Recovery backoff → Ch3 退回迴圈
 ├── Self-Improvement → Memory T2 寫入
 └── Alert P1-P4 → Ch5 成本 alert

Ch2 L2 Context Engine
 ├── Initializer → Ch4 Prompt 組裝（動態區）
 ├── Filesystem → Ch4 交接簡報 schema
 ├── KV-Cache → Ch5 成本優化（prompt caching）
 └── T1→T2 精煉 → Memory Architecture

Ch3 L3 Pipeline
 ├── Step YAML → Ch4 Agent 定義（output schema 對應）
 ├── Eval Gate → Ch4 LLM-Judge（模型 ≥ agent）
 ├── Eval 校準 → Ch5 模型選型（eval 驅動升降級）
 └── 退回迴圈 → Ch1 Error Recovery（escalation）

Ch4 L4 Agent Layer
 ├── Prompt 跨層組裝 → Ch1(constraints) + Ch2(context) + Ch3(task)
 ├── T1 寫入 → Memory Architecture（schema）
 ├── T2 讀取 → Ch2 Initializer（相關性匹配）
 └── 四類角色 → Ch5 Tier 對應

Memory Architecture
 ├── 六種腐化 → Ch1 Self-Improvement（永久約束）
 ├── Retrieval strategy → Ch2 T2 注入
 └── OpenClaw 對應 → 實際部署參考

Ch5 模型選型
 ├── Tier 對應 → Ch4 四類角色
 ├── Eval 驅動 → Ch3 Eval Gate pass rate
 └── 成本優化 → Ch2 KV-Cache + Batch API
```

---

## 寫作經驗沉澱

1. **先搜集社群實戰再歸納**——不要一步到位
2. **通用框架先行**——具體場景用 blockquote 範例後置
3. **命名對應結構**——L5 Memory 矛盾的教訓
4. **一張圖一個問題**——Mermaid 適合流程，SVG 適合同心圓
5. **Why → How → Example**——每個模組的黃金結構
6. **改概念前先 grep 全文**——一個改動可能影響 6 個章節
7. **讓使用者質疑設計決策**——暴露矛盾比隱藏矛盾好
8. **Reference 驗證再動筆**——區分「社群有據」和「AI 幻想」

---

## 待完成

- [ ] **Ch6 · 部署路線圖** — Phase 0-3 的詳細計畫、信任度演進、驗證標準
- [ ] **章節目錄更新** — 統一 index 文件的章節列表
- [ ] **全文一致性檢查** — grep L5、Ch5 Memory 等已棄用的引用
