# W-5 Football Prediction — Replicable Architecture

---

## 一、論文架構 vs 我們能實作的範圍

| 論文層級 | 論文技術 | 我們的實作範圍 | 理由 |
|---|---|---|---|
| Layer 1: Data Ingestion | Streaming API, Web Scraper, REST API | 靜態 CSV + 有限網路抓取 | 無即時需求，backtest 用歷史資料即可 |
| Layer 2: Data Processing | Data Lake → ETL → Data Warehouse → Feature Store | 完整實作（簡化版） | 核心概念可完整重現 |
| Layer 3: AI Core | ML Farm + LLM Cluster + AI Consensus | 完整實作 | 這是論文核心，必須重現 |
| Layer 4: Service | Prediction API, WebSocket, Cache | 跳過 | 無前端需求 |
| Layer 5: Presentation | Web/iOS/Android App | 跳過 | 不在 scope |

---

## 二、資料來源對照

### 論文提到的資料類型

**定量結構化資料 (Quantitative & Structured)**

| 論文資料類型 | 論文描述 | 我們的來源 | 可得性 |
|---|---|---|---|
| Historical Match Data | 勝負紀錄、進球差、主客場結果 | understats `match_info.csv` | ✅ 已有 |
| Player Performance | 上場時間、進球、助攻、紀律紀錄 | understats `player.csv` | ✅ 已有 |
| Advanced Metrics | xG、xA、shot volume | understats `match_info.csv` + `shot_data.csv` | ✅ 已有 |
| Team Dynamics (ELO) | ELO rating 變化 | 從 match history 計算 | ✅ 可計算 |
| Pressing Metrics | PPDA、deep passes | understats `match_info.csv` | ✅ 已有 |
| Market Intelligence | 開盤/收盤賠率、隱含機率 | understats `forecast_w/d/l` | ✅ 已有（收盤賠率）|

**定性非結構化資料 (Qualitative & Unstructured)**

| 論文資料類型 | 論文描述 | 我們的做法 | 可得性 |
|---|---|---|---|
| News & Media | 新聞文章、專欄分析 | 用定量特徵建構 LLM prompt 模擬情境 | ⚠️ 模擬 |
| Social Sentiment | Twitter/Reddit 輿論 | 跳過（需要 API 付費） | ❌ 跳過 |
| Tactical Reports | 賽前記者會、戰術分析 | 由 LLM tactician agent 從 PPDA/pressing 推斷 | ⚠️ 模擬 |
| Injury Reports | 球員傷停資訊 | 跳過（understats 無此欄位） | ❌ 跳過 |

> **設計決策**：論文的定性資料是靠真實新聞 + NLP pipeline。
> 我們的 LLM agents 直接從定量特徵推理，模擬「讀過相關分析後的判斷」。
> 這是最大的差異，也是我們結果可能低於論文的主因之一。

---

## 三、Layer 2 — Data Processing & Storage

論文描述的流程：`Data Lake → ETL/ELT → Data Warehouse → Feature Store`

### 3.1 Data Lake（原始資料層）

**定義**：所有原始資料的存放地，保持原始格式，不做任何修改。

```
league/understats/
├── EPL/
│   ├── match_info.csv      ← 每場比賽的統計（主力資料來源）
│   ├── match_data.csv      ← 同上，含賠率欄位
│   ├── player.csv          ← 球員球季統計
│   ├── shot_data.csv       ← 逐射門紀錄
│   └── season.csv          ← 聯賽賽季資訊
├── La_Liga/ ...
├── Bundesliga/ ...
├── Ligue_1/ ...
└── RFPL/ ...
```

**原則**：
- 這層的資料永遠不修改
- 所有後續處理都從這層讀取，產出新檔案

### 3.2 ETL Pipeline（清洗與整合層）

**定義**：從 Data Lake 讀取、清洗、統一格式、合併多來源。

**應該實作的內容**：
- 統一5個聯賽的欄位名稱與型別（各 CSV 有細微差異）
- 轉換 `date` 為 datetime，依時間排序
- 計算比賽結果 label：H（主場贏）/ D（平）/ A（客場贏）
- 正規化 bookmaker odds，使其加總為 1（implied probability）
- 合併 `match_info` 與 `player` 資料（球季層級 join）
- 處理缺失值（早期賽季可能缺少部分進階指標）
- 輸出：一個統一的 `matches_clean.parquet`

### 3.3 Data Warehouse（結構化分析層）

**定義**：清洗後、可直接查詢的結構化資料，按分析用途組織。

**應該包含的表格結構**：

`matches` — 每場比賽一行
- 比賽基本資訊（日期、主客場、聯賽、賽季）
- 比賽結果（進球數、label）
- 當場的原始統計（xG、shots、PPDA、deep passes）
- bookmaker 隱含機率

`team_season_stats` — 每支球隊每個賽季一行
- 球季累計進球、失球、積分、勝負平場數
- 球季累計 xG、xGA
- 聯賽排名（若可計算）

`player_season_stats` — 每球員每賽季一行
- 直接來自 `player.csv`
- 加上球隊聚合欄位（該隊本賽季 top scorer xG 等）

### 3.4 Feature Store（特徵管理層）

**定義**：為 ML 模型準備的、時間感知的特徵集合。
最關鍵的設計原則：**特徵 M 只能使用比賽 M 發生前的資訊**。

**應該儲存的特徵（每場比賽一行，~28個欄位）**：

*ELO 類*
- `h_elo_pre`：主場隊比賽前的 ELO
- `a_elo_pre`：客場隊比賽前的 ELO
- `elo_diff`：兩者差值

*Rolling Form 類（最近5場，含 shift 防止 lookahead）*
- `h_form_xg_scored_5`、`h_form_xg_conceded_5`
- `h_form_goals_scored_5`、`h_form_goals_conceded_5`
- `h_form_wins_5`、`h_form_draws_5`
- `h_form_ppda_5`、`h_form_deep_5`
- 客場隊同上（`a_*`）

*Head-to-Head 類（近10次交手）*
- `h2h_h_win_rate`、`h2h_draw_rate`、`h2h_avg_goals`
- `h2h_n_matches`（交手場數，影響可信度）

*賽季累計類（截至本場之前）*
- `h_season_pts`、`h_season_gd`、`h_season_xg_diff`

*市場類*
- `bm_implied_h`、`bm_implied_d`、`bm_implied_a`
- `bm_h_minus_a`（主客場隱含機率差）

**Feature Store 的兩個介面**：
- `build_store()`：一次性跑完所有特徵計算，存成 parquet
- `get_features(match_id)` 或 `get_features(team, date)`：查詢單場比賽的特徵向量，供即時預測用

---

## 四、Layer 3 — AI Core（W-5 Model）

論文將這層分為三個子系統：**ML Model Farm**、**LLM Cluster**、**AI Consensus Mechanism**，最後由 **Intelligent Synthesis（meta-learner）** 整合。

### 4.1 ML Model Farm（定量基準預測）

**角色**：從結構化特徵產出 Win/Draw/Loss 的基準機率。

**應該實作的內容**：
- 模型：XGBoost（主力）+ LightGBM（可選，作為 ensemble）
- 訓練協議：Walk-forward，每次用5年訓練、預測下一年
- 輸出：每場比賽三個機率值 `[P(H), P(D), P(A)]`，加總為 1
- 模型儲存：每個 fold 存獨立模型檔

**Walk-forward 設計**：
```
Fold 1: 訓練 2014-2018 → 預測 2019
Fold 2: 訓練 2015-2019 → 預測 2020
Fold 3: 訓練 2016-2020 → 預測 2021
Fold 4: 訓練 2017-2021 → 預測 2022
Fold 5: 訓練 2018-2022 → 預測 2023
```

**預期效能**：Accuracy ~58%，Brier ~0.205

### 4.2 LLM Cluster — 多供應商設計

**角色**：對比賽情境進行定性推理，產出各自的預測與信心分數。

**為什麼要多供應商**：
不同公司的 LLM 有不同訓練資料與架構偏見，錯誤不相關，ensemble 效果才真實。
若5個 agent 全用 Claude，共同偏見無法被糾正。

**Prompt 的資訊結構**（無論哪個 agent 都收到同樣的 match context）：
- 比賽基本資訊（聯賽、賽季、主客場）
- ELO 差值
- 雙方近5場 rolling form（xG、進失球、勝負）
- H2H 歷史
- 雙方 PPDA 和 deep passes（進攻強度代理指標）
- Bookmaker 隱含機率

**5個 Agent 的角色與供應商分配**：

| Agent | 分析焦點 | 供應商 | 原因 |
|---|---|---|---|
| statistician | 純數字：xG、ELO、shot volume | Claude | 結構化推理強 |
| tactician | 戰術指標：PPDA、逼搶、地面控制 | GPT-4o | 戰術模式識別能力強 |
| form_analyst | 近5場動能，忽略長期歷史 | Gemini | 序列趨勢分析 |
| market_analyst | 賠率市場隱含資訊，尋找偏差 | GPT-4o | 第二個 OpenAI，與 Claude 形成對比 |
| devils_advocate | 反向思考，找出冷門可能 | Claude | 對抗性推理能力 |

### 4.3 AI Consensus Mechanism（LangGraph）

**角色**：協調 5 個 agents 進行結構化辯論，產出集體共識。

**論文描述的機制**：
1. 每個 agent 獨立分析 → 產出預測 + 信心分數 + 推理
2. 每個 agent 看到其他 agent 的分析 → 可修改或維持自己的預測
3. 用加權投票（權重 = 信心分數）產出最終共識機率

**用 LangGraph 實作的原因**：
- 管理跨 round 的 state（round1 結果要傳給 round2）
- 支援 round 1 的5個 agent **平行呼叫**
- 內建 retry 和 error handling
- State graph 讓整個辯論流程可 debug、可 visualize

**Graph 的節點結構**：

```
[START]
   ↓ 平行分支（5個 round 1 nodes 同時執行）
[r1_statistician] [r1_tactician] [r1_form_analyst] [r1_market_analyst] [r1_devils_advocate]
   ↓ 全部完成後 merge
[r1_collect]
   ↓ 平行分支（5個 round 2 nodes 同時執行，各自帶入其他4人的 round 1 結果）
[r2_statistician] [r2_tactician] [r2_form_analyst] [r2_market_analyst] [r2_devils_advocate]
   ↓ 全部完成後
[weighted_vote]
   ↓
[END] → ConsensusResult
```

**ConsensusResult 包含**：
- 最終預測（H/D/A）
- `p_home`, `p_draw`, `p_away`（加權投票後正規化）
- `agreement_rate`：多數方向的總權重佔比（高 = agents 共識強）
- `debate_summary`：哪幾個 agent 同意、哪幾個反對、關鍵分歧點

### 4.4 Intelligent Synthesis（Meta-Learner）

**角色**：學習如何整合 ML 基準機率 與 LLM 共識機率，產出最終預測。

**論文用 PyTorch（小型 MLP）；我們用 LogisticRegression**

原因：
- 論文有 ~48,000 筆 LLM-labeled 訓練樣本（485,000 × 10%）
- 我們預計只有 200-500 筆（LLM 成本限制）
- MLP 在 300 筆樣本上會 overfit；LogisticRegression 不會
- 如果未來 LLM 標注樣本累積到 2,000+，可考慮換成 2層 MLP

**輸入特徵（7個）**：
- `[p_h_xgb, p_d_xgb, p_a_xgb]`：XGBoost 輸出的三個機率
- `[p_h_llm, p_d_llm, p_a_llm]`：LLM 共識的三個機率
- `[agreement_rate]`：5個 agents 的同意程度（影響 LLM 信號的可信度）

**它能學到什麼**：
- 當 XGBoost 和 LLM 一致時 → 提高信心
- 當兩者分歧時 → 依歷史哪個比較準來加權
- 當 agreement_rate 低時（agents 意見分裂）→ 退回到 XGBoost

**訓練方式**：
- 使用 walk-forward fold 的 out-of-sample XGBoost 機率（不用 in-sample）
- 在 training set 中隨機抽樣跑 LLM consensus → 拿到 llm_probs + agreement
- 組合成 7 維特徵向量，配合真實結果 label 訓練

---

## 五、整體資料流

```
[Data Lake]
  understats CSVs (raw)
       ↓
[ETL Pipeline]
  統一格式、計算 label、正規化賠率
       ↓
[Data Warehouse]
  matches_clean.parquet
  team_season_stats.parquet
       ↓
[Feature Store]
  feature_store.parquet
  （每場比賽 ~28 個時間安全特徵）
       ↓ ─────────────────────────────────────────────┐
[ML Model Farm]                              [LLM Cluster via LangGraph]
  XGBoost walk-forward                         5 agents × 2 rounds
  → [P(H), P(D), P(A)]_xgb                    → ConsensusResult
       ↓                                              ↓
       └──────────────── [Meta-Learner] ──────────────┘
                              ↓
                      [PredictionResult]
                  prediction / p_home / p_draw / p_away
                  confidence / debate_summary
```

---

## 六、各模組的輸入輸出規格

| 模組 | 輸入 | 輸出 |
|---|---|---|
| ETL | raw CSVs | `matches_clean.parquet` |
| Feature Store Builder | `matches_clean.parquet` | `feature_store.parquet` |
| XGBoost (一個 fold) | feature vectors + labels | 模型檔 + test set 的 `[P(H), P(D), P(A)]` |
| Prompt Builder | 一行 feature store 資料 | match context 文字字串 |
| LangGraph Graph | match context 文字 | `ConsensusResult` |
| Meta-Learner (fit) | xgb_probs + llm_probs + agreement + labels | 訓練好的分類器 |
| Meta-Learner (predict) | xgb_probs + llm_probs + agreement | 最終 `[P(H), P(D), P(A)]` |
| Pipeline (predict_match) | 球隊名稱 + 日期 + 聯賽 | `PredictionResult` |

---

## 七、檔案結構

```
fifa/
│
├── data/
│   ├── raw/                        ← Data Lake（等同 league/understats/）
│   └── processed/
│       ├── matches_clean.parquet   ← Data Warehouse
│       ├── team_season_stats.parquet
│       └── features/
│           └── feature_store.parquet  ← Feature Store
│
├── src/
│   ├── etl/
│   │   ├── loader.py               ← 讀取 raw CSVs，統一欄位
│   │   └── cleaner.py              ← 清洗、計算 label、正規化
│   │
│   ├── features/
│   │   ├── engineer.py             ← ELO、rolling form、H2H、season stats
│   │   └── store.py                ← build_store() / get_features()
│   │
│   ├── ml/
│   │   ├── train.py                ← XGBoost walk-forward
│   │   └── evaluate.py             ← accuracy / brier / log_loss
│   │
│   ├── llm/
│   │   ├── providers.py            ← Claude / GPT-4o / Gemini API wrappers
│   │   ├── personas.py             ← 5 agent 定義（system prompt + provider）
│   │   ├── prompt_builder.py       ← feature row → match context text
│   │   └── graph.py                ← LangGraph state machine（2-round debate）
│   │
│   ├── fusion/
│   │   └── meta_learner.py         ← LogisticRegression 整合 ML + LLM
│   │
│   └── pipeline.py                 ← predict_match() 入口
│
├── models/
│   ├── xgboost_fold_2019.pkl
│   ├── xgboost_fold_2020.pkl
│   ├── ...
│   └── meta_learner.pkl
│
├── notebooks/
│   ├── 01_eda.ipynb                ← 了解資料分佈
│   ├── 02_features.ipynb           ← 驗證特徵無 lookahead
│   ├── 03_ml_baseline.ipynb        ← XGBoost walk-forward 結果
│   ├── 04_llm_consensus.ipynb      ← 單場測試 LangGraph pipeline
│   └── 05_backtest.ipynb           ← 完整評估報告
│
├── config.py                       ← API keys, 超參數, 路徑常數
├── checklist.md
└── structure.md
```

---

## 八、我們跳過的部分與原因

| 論文元素 | 跳過原因 |
|---|---|
| Real-time streaming (Kafka/Spark) | 我們做 backtest，不需要即時資料流 |
| Feast Feature Store | Feast 解決線上 serving 的毫秒級查詢問題，parquet 對我們夠用 |
| PyTorch meta-learner | LLM 標注樣本不足，LogisticRegression 更適合 |
| Social media sentiment | 需要 Twitter/Reddit API 授權，且對 backtest 幫助有限 |
| Injury reports | understats 無此資料 |
| FastAPI / WebSocket | 無前端需求 |
| 485k 筆訓練資料 | 論文用 synthetic data，可信度存疑；我們用真實資料更有價值 |

---

## 九、預期效能目標（基於真實資料）

| 模型 | Accuracy | Brier Score | Log Loss |
|---|---|---|---|
| Bookmaker Odds（基準線） | ~54% | ~0.218 | ~0.941 |
| XGBoost Only | ~58-62% | ~0.200 | ~0.890 |
| 3-LLM Consensus | ~65-70% | ~0.185 | ~0.820 |
| Full W-5 (XGB + 5-LLM + Meta) | ~68-73% | ~0.175 | ~0.790 |

> 論文聲稱 85.9% 是在 synthetic data 上測試的結果，不具備真實參考價值。
> 在真實資料上達到 70%+ 已超越絕大多數學術研究。
