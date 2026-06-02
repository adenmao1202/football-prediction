# W-5 Football Prediction — Implementation Architecture

---

## 一、實作範圍

本專案只實作**可以用現有資料完成**的部分。
外部 API、社群媒體 sentiment、即時串流**全部不在此次範圍內**。

| 層級 | 實作內容 | 工具 |
|---|---|---|
| Data Lake | understats 靜態 CSV | 本地檔案系統 |
| ETL | 清洗、統一格式、計算 label | pandas |
| Data Warehouse | 結構化比賽資料 | parquet |
| Feature Store | 時間安全的 ML 特徵 | parquet |
| ML Model Farm | 定量基準預測 | XGBoost |
| LLM Cluster | 5 agents 定性推理（純從特徵推理，無外部新聞） | Claude / GPT-4o / Gemini |
| AI Consensus | 2-round 辯論 + 加權投票 | LangGraph |
| Meta-Learner | 整合 ML + LLM 輸出 | LogisticRegression |

---

## 二、資料來源（僅 understats）

所有資料來自 `league/understats/`，共6個聯賽：EPL、La Liga、Bundesliga、Ligue 1、RFPL、Serie A。

| 檔案 | 內容 | 用途 |
|---|---|---|
| `match_info.csv` | 每場比賽：xG、shots、PPDA、deep passes、bookmaker odds、進球數 | 主力資料來源 |
| `match_data.csv` | 同上，含 `forecast_w/d/l`（賠率） | bookmaker 基準線 |
| `player.csv` | 每球員每球季：進球、xG、助攻、xA、上場時間 | 球隊進攻強度特徵 |
| `shot_data.csv` | 逐射門紀錄：xG per shot、situation、shotType | shot quality 特徵 |

**不使用的資料類型**：新聞、社群媒體、即時賠率 API、傷停報告。
LLM agents 的定性推理完全基於上表的量化特徵，不引入任何外部文字資料。

---

## 三、Data Processing & Storage

### 3.1 Data Lake

原始資料存放層，**永遠不修改**。

```
league/understats/
├── EPL/
│   ├── match_info.csv
│   ├── match_data.csv
│   ├── player.csv
│   ├── shot_data.csv
│   └── season.csv
├── La_Liga/ ...
├── Bundesliga/ ...
├── Ligue_1/ ...
├── RFPL/ ...
└── Serie_A/ ...
```

### 3.2 ETL（`src/etl/`）

從 Data Lake 讀取，輸出乾淨的 `matches_clean.parquet`。

**loader.py** 負責：
- 讀取全部聯賽的 `match_info.csv`
- 統一欄位名稱（各聯賽 CSV 有細微差異）
- 加上 `league` 欄位
- 轉換 `date` 為 datetime，依時間排序

**cleaner.py** 負責：
- 計算比賽結果 label：`H`（主場贏）/ `D`（平）/ `A`（客場贏）
- 正規化 bookmaker odds 使加總為 1（implied probability）
- 處理缺失值（早期賽季可能缺少 PPDA 等進階指標）
- 合併 `player.csv`（球季層級 join，取球隊進攻強度聚合值）

### 3.3 Data Warehouse（`data/processed/`）

清洗後的結構化資料，按分析用途分表存放。

**matches_clean.parquet** — 每場比賽一行：
- 比賽基本資訊：date, home_team, away_team, league, season
- 比賽結果：h_goals, a_goals, result（label）
- 當場原始統計：h_xg, a_xg, h_shot, a_shot, h_ppda, a_ppda, h_deep, a_deep
- Bookmaker 隱含機率：bm_h, bm_d, bm_a

**team_season_stats.parquet** — 每隊每球季一行：
- 球季累計：pts, gd, wins, draws, losses
- 球季累計進階：xg_scored, xg_conceded, xg_diff

### 3.4 Feature Store（`data/processed/features/`）

為 ML 模型與 LLM prompt 準備的特徵層。
**核心原則：特徵 M 只能使用比賽 M 發生前的資料**（所有 rolling 操作使用 `.shift(1)`）。

**feature_store.parquet** — 每場比賽一行，~28 個特徵欄位：

*ELO 類*（從 match history 逐場計算，K=32，起始值=1500）
- `h_elo_pre`、`a_elo_pre`、`elo_diff`

*Rolling Form 類*（最近 5 場，含 shift）
- `h_form_xg_scored_5`、`h_form_xg_conceded_5`
- `h_form_goals_scored_5`、`h_form_goals_conceded_5`
- `h_form_wins_5`、`h_form_draws_5`
- `h_form_ppda_5`、`h_form_deep_5`
- 客場隊同上（`a_*`）

*Head-to-Head 類*（近 10 次交手）
- `h2h_h_win_rate`、`h2h_draw_rate`、`h2h_avg_goals`、`h2h_n_matches`

*賽季累計類*（截至本場之前）
- `h_season_pts`、`h_season_gd`、`h_season_xg_diff`
- 客場隊同上（`a_*`）

*市場類*
- `bm_implied_h`、`bm_implied_d`、`bm_implied_a`
- `bm_h_minus_a`（主客場隱含機率差，主場優勢代理指標）

**Feature Store 介面**：
- `build_store()` — 一次性建立，存成 parquet
- `get_features(match_id)` — 查詢單場特徵向量

---

## 四、AI Core

### 4.1 ML Model Farm（`src/ml/`）

**輸入**：feature_store 的特徵向量
**輸出**：每場比賽的 `[P(H), P(D), P(A)]`，加總為 1

模型：XGBoost multiclass（`multi:softprob`）

Walk-forward 訓練協議：
```
Fold 1: train 2014-2018 → test 2019
Fold 2: train 2015-2019 → test 2020
Fold 3: train 2016-2020 → test 2021
Fold 4: train 2017-2021 → test 2022
Fold 5: train 2018-2022 → test 2023
```

每個 fold 儲存獨立模型檔（`models/xgboost_fold_{year}.pkl`）。
預期效能：Accuracy ~58-62%，Brier ~0.200。

### 4.2 LLM Cluster（`src/llm/`）

**輸入**：feature_store 的一行資料 → 轉成 match context 文字
**輸出**：每個 agent 的 `{prediction, confidence, reasoning}`

**重要限制**：LLM agents 的推理完全基於量化特徵，不引入任何外部新聞或 sentiment。
Prompt 只包含：ELO、rolling form、H2H、PPDA、deep passes、bookmaker odds。

**5 個 Agent 與供應商分配**：

| Agent | 分析焦點 | 供應商 |
|---|---|---|
| statistician | xG、ELO、shot volume 等純數字 | Claude |
| tactician | PPDA、deep passes、逼搶強度 | GPT-4o |
| form_analyst | 近 5 場動能，忽略長期歷史 | Gemini |
| market_analyst | bookmaker 隱含機率，找出市場與統計的偏差 | GPT-4o |
| devils_advocate | 反向思考，挑戰顯而易見的預測 | Claude |

多供應商原因：不同架構的 LLM 有不同訓練偏見，錯誤不相關，ensemble 效果才真實。

### 4.3 AI Consensus — LangGraph（`src/llm/graph.py`）

**輸入**：match context 文字
**輸出**：`ConsensusResult`（最終機率 + agreement_rate + debate_summary）

LangGraph StateGraph 流程：

```
[START]
   ↓ 平行執行（Round 1）
[r1_statistician] [r1_tactician] [r1_form_analyst] [r1_market_analyst] [r1_devils_advocate]
   ↓ 全部完成後 merge
[r1_collect]
   ↓ 平行執行（Round 2，各 agent 帶入其他 4 人的 Round 1 結果）
[r2_statistician] [r2_tactician] [r2_form_analyst] [r2_market_analyst] [r2_devils_advocate]
   ↓ 全部完成後
[weighted_vote]     ← 權重 = 各 agent 的 confidence 分數
   ↓
[END]
```

`ConsensusResult` 包含：
- `prediction`：H / D / A
- `p_home`、`p_draw`、`p_away`（加權投票後正規化，加總為 1）
- `agreement_rate`：多數方向的總權重佔比
- `debate_summary`：哪些 agent 同意、哪些反對、關鍵分歧

### 4.4 Meta-Learner（`src/fusion/meta_learner.py`）

**輸入**（7 個特徵）：
- `[p_h_xgb, p_d_xgb, p_a_xgb]` — XGBoost 輸出
- `[p_h_llm, p_d_llm, p_a_llm]` — LLM 共識輸出
- `[agreement_rate]` — agents 意見一致程度

**模型**：LogisticRegression（multinomial）

為何不用 PyTorch：論文有 ~48,000 筆 LLM-labeled 樣本可訓練 MLP。
我們預計只有 200-500 筆，MLP 會 overfit，LogisticRegression 不會。

**訓練方式**：
- XGBoost 機率取 out-of-sample（walk-forward fold 結果，不用 in-sample）
- 在訓練集隨機抽 200-500 場跑 LLM consensus → 得到 llm_probs + agreement
- 組合成 7 維向量配合真實 label 訓練

---

## 五、整體資料流

```
[Data Lake]
  league/understats/ (raw CSVs)
          ↓
     [ETL Pipeline]
  loader.py → cleaner.py
          ↓
  [Data Warehouse]
  matches_clean.parquet
  team_season_stats.parquet
          ↓
   [Feature Store]
  feature_store.parquet (~28 features/match)
          ↓ ─────────────────────────────────────────┐
  [ML Model Farm]                         [LLM Cluster + LangGraph]
  XGBoost walk-forward                    5 agents × 3 providers × 2 rounds
  → [P(H), P(D), P(A)]_xgb               → ConsensusResult
          ↓                                          ↓
          └──────────── [Meta-Learner] ──────────────┘
                               ↓
                       [PredictionResult]
              prediction / p_home / p_draw / p_away / confidence / debate_summary
```

---

## 六、各模組輸入輸出

| 模組 | 輸入 | 輸出 |
|---|---|---|
| ETL (loader + cleaner) | raw CSVs | `matches_clean.parquet` |
| Feature Store Builder | `matches_clean.parquet` | `feature_store.parquet` |
| XGBoost (per fold) | feature vectors + labels | 模型檔 + `[P(H), P(D), P(A)]` |
| Prompt Builder | feature_store 一行資料 | match context 文字 |
| LangGraph Graph | match context 文字 | `ConsensusResult` |
| Meta-Learner (fit) | xgb_probs + llm_probs + agreement + labels | 訓練好的分類器 |
| Meta-Learner (predict) | xgb_probs + llm_probs + agreement | 最終 `[P(H), P(D), P(A)]` |
| pipeline.predict_match | home_team, away_team, date, league | `PredictionResult` |

---

## 七、檔案結構

```
fifa/
├── league/understats/          ← Data Lake（不修改）
│
├── data/
│   └── processed/
│       ├── matches_clean.parquet       ← Data Warehouse
│       ├── team_season_stats.parquet
│       └── features/
│           └── feature_store.parquet  ← Feature Store
│
├── src/
│   ├── etl/
│   │   ├── loader.py           ← 讀取 raw CSVs，統一欄位
│   │   └── cleaner.py          ← 清洗、label、正規化
│   │
│   ├── features/
│   │   ├── engineer.py         ← ELO、rolling form、H2H、season stats
│   │   └── store.py            ← build_store() / get_features()
│   │
│   ├── ml/
│   │   ├── train.py            ← XGBoost walk-forward
│   │   └── evaluate.py         ← accuracy / brier / log_loss
│   │
│   ├── llm/
│   │   ├── providers.py        ← Claude / GPT-4o / Gemini wrappers
│   │   ├── personas.py         ← 5 agent 定義（system prompt + provider）
│   │   ├── prompt_builder.py   ← feature row → match context text
│   │   └── graph.py            ← LangGraph 2-round debate + weighted vote
│   │
│   ├── fusion/
│   │   └── meta_learner.py     ← LogisticRegression 整合 ML + LLM
│   │
│   └── pipeline.py             ← predict_match() 入口
│
├── models/
│   ├── xgboost_fold_2019.pkl
│   ├── ...
│   └── meta_learner.pkl
│
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_features.ipynb       ← 驗證無 lookahead
│   ├── 03_ml_baseline.ipynb    ← walk-forward 結果
│   ├── 04_llm_consensus.ipynb  ← 單場 LangGraph 測試
│   └── 05_backtest.ipynb       ← 完整評估報告
│
├── config.py
├── checklist.md
└── structure.md
```

---

## 八、預期效能（真實資料）

| 模型 | Accuracy | Brier Score | Log Loss |
|---|---|---|---|
| Bookmaker Odds（基準線） | ~54% | ~0.218 | ~0.941 |
| XGBoost Only | ~58-62% | ~0.200 | ~0.890 |
| 3-LLM Consensus | ~65-70% | ~0.185 | ~0.820 |
| Full W-5 (XGB + 5-LLM + Meta) | ~68-73% | ~0.175 | ~0.790 |

論文的 85.9% 是在 synthetic data 上的結果，不具真實參考價值。
在真實資料上達到 70%+ 已超越絕大多數學術研究。
