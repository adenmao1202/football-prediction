# W-5 Implementation Checklist

## Phase 1 — Data & Feature Engineering

### 1.1 Data Loading (`src/features/loader.py`)
- [ ] 讀取單一聯賽的 `match_info.csv`，確認欄位完整
- [ ] 統一5個聯賽的欄位名稱（各聯賽 CSV 格式可能有細微差異）
- [ ] 合併5聯賽成一個 DataFrame，加上 `league` 欄位
- [ ] 確認 `date` 欄位轉成 datetime 型別，排序正確
- [ ] 確認資料筆數合理（EPL 每季約 380 場 × N 年）
- [ ] 處理重複或缺失的 match_id

### 1.2 ELO 計算 (`src/features/engineer.py`)
- [ ] 初始化所有球隊 ELO = 1500
- [ ] 依 date 排序後逐場更新 ELO（不能 vectorize，必須 loop）
- [ ] 確認 ELO 是用比賽**前**的值作為特徵（先記錄再更新）
- [ ] 驗證：強隊（曼城、巴薩）的 ELO 應顯著高於弱隊
- [ ] 不同聯賽的 ELO 是否共用還是分開？（共用較符合真實相對強度）

### 1.3 Rolling Form 計算
- [ ] 為每支球隊建立「時序比賽記錄」（每場都有 home/away 兩種角色）
- [ ] 實作 `get_team_last_n_games(team, date, n)` — 取該日期前 n 場
- [ ] 計算 rolling xG scored、xG conceded、goals scored、goals conceded、wins
- [ ] **關鍵驗證**：第一場比賽的 rolling 特徵應為 NaN，確認沒有 lookahead
- [ ] 決定 NaN 填補策略（用聯賽平均值或丟掉前 N 場）

### 1.4 Head-to-Head 計算
- [ ] 找出兩隊歷史交手記錄（同時考慮主客場互換）
- [ ] 計算 H2H 主場勝率、平均進球、交手場數
- [ ] 交手場數少於 3 場時的填補策略（用各自 ELO 差異估算）

### 1.5 其他特徵
- [ ] Season cumulative points & GD（用 group by season + cumsum，注意 shift）
- [ ] Bookmaker implied probability（正規化 forecast_w/d/l 使其加總為 1）
- [ ] Rolling PPDA 和 deep passes（rolling 5場）
- [ ] Rolling shot on target rate

### 1.6 Lookahead 驗證（必做）
- [ ] 取一場已知比賽，手動驗證其所有特徵只用了比賽日前的資料
- [ ] 印出某隊的 rolling form，確認序列順序正確
- [ ] 確認 ELO 特徵是比賽前的 ELO，不是更新後的

### 1.7 Feature Store
- [ ] 儲存為 `feature_store.parquet`（比 CSV 快 10x）
- [ ] 記錄特徵欄位清單（後面 ML 和 prompt_builder 都需要對齊）
- [ ] 確認最終 feature 數量和每欄的分佈（無異常 outlier）

---

## Phase 2 — ML Baseline (`src/ml/`)

### 2.1 資料切割
- [ ] 依年份切割 train/test（不能用 random split，必須時間序列切割）
- [ ] 確認 test set 中沒有任何特徵使用了 test 期間的資料
- [ ] 定義 label encoding：H=0, D=1, A=2（或其他，保持一致）

### 2.2 XGBoost 訓練
- [ ] 安裝 `xgboost` 套件
- [ ] 用一個 fold 先跑通（train 2014-2018 → test 2019）
- [ ] 確認輸出是 3-class 機率（`predict_proba` shape = [n, 3]）
- [ ] 儲存每個 fold 的模型到 `models/xgboost_fold_{year}.pkl`

### 2.3 Walk-forward 驗證
- [ ] 跑完所有 fold（5個 fold 對應5個 test 年份）
- [ ] 匯總所有 fold 的 test 預測結果

### 2.4 評估
- [ ] 計算整體 prediction accuracy（應在 55-62% 之間）
- [ ] 計算 Brier Score 和 Log Loss
- [ ] 與 bookmaker implied probability 比較（bookmaker 約 54%，你的模型應該要贏）
- [ ] 畫出 feature importance，確認合理（ELO 和 xG 應該是最重要的）
- [ ] 如果 accuracy < 54%，代表特徵工程有問題（比不過瞎猜）

---

## Phase 3 — LLM 基礎設施 (`src/llm/`)

### 3.1 Provider 連線測試 (`providers.py`)
- [ ] 設定環境變數：`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`
- [ ] 測試 Claude API 連線（一個簡單 hello world 呼叫）
- [ ] 測試 OpenAI API 連線
- [ ] 測試 Gemini API 連線
- [ ] 確認三個 provider 都能在 30 秒內回應
- [ ] 實作統一的錯誤處理（API timeout, rate limit → retry with backoff）

### 3.2 Prompt Builder (`prompt_builder.py`)
- [ ] 從 feature_store 取一筆資料，手動確認所有欄位都有值
- [ ] 實作 `build_match_prompt()` — 產出格式化的 match context 文字
- [ ] 在 prompt 末尾明確要求 JSON 格式回應
- [ ] 手動閱讀 prompt 輸出，確認資訊清楚、無歧義
- [ ] 加入 PPDA 的說明（低 PPDA = 高逼搶，多數人不懂這個）

### 3.3 單一 Agent 測試
- [ ] 用 statistician persona 呼叫 Claude，傳入 prompt
- [ ] 確認回應可以被 JSON parse（處理模型有時回傳 markdown code block 的情況）
- [ ] 確認 prediction 只有 H/D/A 其中一個
- [ ] 確認 confidence 是 0-100 的整數
- [ ] 測試 5 個 persona 各呼叫一次，觀察推理風格差異是否明顯

---

## Phase 4 — LangGraph Consensus (`src/llm/graph.py`)

### 4.1 LangGraph 環境設定
- [ ] 安裝 `langgraph` 套件
- [ ] 閱讀 LangGraph StateGraph 基本概念（State, Node, Edge）
- [ ] 用一個簡單的2-node graph 確認 LangGraph 可以運行

### 4.2 Round 1 設計
- [ ] 定義 `ConsensusState` TypedDict（match_prompt, round1, round2, consensus）
- [ ] 為每個 agent 建立 Round 1 node function
- [ ] 確認5個 Round 1 nodes 可以**平行**執行（LangGraph Send API 或 parallel branches）
- [ ] 測試：跑 Round 1，確認收到5個不同 agent 的回應

### 4.3 Round 2 設計
- [ ] 實作 `build_peer_summary()` — 把其他4個 agent 的 Round 1 結果整理成文字
- [ ] 為每個 agent 建立 Round 2 node function（加入 peer summary）
- [ ] 確認 Round 2 prompt 正確傳入了 peer views
- [ ] 觀察：agent 在看到 peer views 後，有沒有改變預測？

### 4.4 Weighted Vote
- [ ] 實作 `aggregate_votes()` — 依 confidence 加權投票
- [ ] 確認輸出的 p_home + p_draw + p_away = 1.0
- [ ] 計算 agreement_rate（多數預測方向的總權重佔比）
- [ ] 實作 `build_debate_summary()` — 文字摘要：哪些 agent 同意、哪些反對、關鍵理由

### 4.5 完整 Graph 測試
- [ ] 用1場真實比賽跑完整個 graph
- [ ] 計時：完整 2-round consensus 花多少秒？（預期 15-30 秒）
- [ ] 確認 graph 可以正確處理 API 呼叫失敗的情況（某個 agent timeout）
- [ ] 用3-5場不同類型的比賽測試（強弱懸殊、勢均力敵、H2H 歷史特殊）

---

## Phase 5 — Meta-Learner & Pipeline

### 5.1 準備訓練資料 (`src/fusion/meta_learner.py`)
- [ ] 從 walk-forward fold 結果取得 XGBoost 的 out-of-sample 機率（不能用 in-sample）
- [ ] 在訓練集中隨機抽樣 200-500 場，跑 LLM consensus
- [ ] 建立訓練矩陣：[p_h_xgb, p_d_xgb, p_a_xgb, p_h_llm, p_d_llm, p_a_llm, agreement]
- [ ] 確認 label 和 XGBoost fold 的 label encoding 一致

### 5.2 Meta-Learner 訓練
- [ ] 安裝 `scikit-learn`（應該已有）
- [ ] 訓練 LogisticRegression（multinomial）
- [ ] 確認 training accuracy 合理（不應該 100%，那代表 overfit）
- [ ] 儲存模型到 `models/meta_learner.pkl`

### 5.3 End-to-End Pipeline (`src/pipeline.py`)
- [ ] 實作 `predict_match(home, away, date, league)` — 串起所有步驟
- [ ] 定義 `PredictionResult` dataclass
- [ ] 測試5場不同比賽，確認輸出格式正確
- [ ] 確認 confidence 分數合理（不應該所有比賽都是 90%+ confidence）

---

## Phase 6 — Evaluation (`notebooks/05_backtest.ipynb`)

### 6.1 基準線建立
- [ ] 計算 bookmaker odds 的 accuracy、Brier、Log Loss（這是你的 lower bound）
- [ ] 計算 XGBoost only 的三個指標（應略勝 bookmaker）
- [ ] 整理成 comparison table

### 6.2 LLM Consensus 評估
- [ ] 在 test set 中取樣 100-200 場跑 full consensus
- [ ] 計算 3-agent（statistician, form_analyst, market_analyst）的 accuracy
- [ ] 計算 5-agent full consensus 的 accuracy
- [ ] 比較 Round 1 vs Round 2 的 accuracy（辯論有沒有幫助？）

### 6.3 Full W-5 評估
- [ ] 對有 LLM labels 的場次跑 meta-learner 預測
- [ ] 計算完整系統的三個指標
- [ ] 畫 calibration plot（confidence vs actual accuracy）
- [ ] 依聯賽（EPL vs 其他）分析表現差異

### 6.4 整理報告
- [ ] 建立 comparison table（對應論文 Table 2 格式）
- [ ] 記錄你的真實數字（不要假設會達到論文的 85.9%）
- [ ] 分析哪個 agent persona 最準確、最常改變心意
- [ ] 記錄 LLM API 總花費

---

## 依賴套件安裝清單

```
xgboost
lightgbm        # optional, for comparison
scikit-learn
pandas
pyarrow         # for parquet
langgraph
anthropic
openai
google-generativeai
python-dotenv   # for API key management
joblib          # model serialization
```
