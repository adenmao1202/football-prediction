# FIFA Project — 資料總覽

## 1. mix/ — StatsBomb Open Data（事件級別，JSON）

**來源**：StatsBomb 官方開放資料（https://github.com/statsbomb/open-data）
**格式**：JSON，依 match_id 命名
**座標系**：0~1 正規化（實際球場 105x68 公尺，需 x105 / x68 換算）

### 子資料夾

| 資料夾 | 檔案數 | 內容 |
|--------|--------|------|
| competitions.json | 1 | 所有開放賽事索引（80 賽季） |
| matches/{comp_id}/{season_id}.json | 80 | 比賽基本資訊 |
| events/{match_id}.json | 4,235 | 逐事件追蹤（每場約 3,000 事件） |
| lineups/{match_id}.json | 4,235 | 出賽名單 |
| three-sixty/{match_id}.json | 426 | 360度球員座標快照（子集） |

### 涵蓋賽事（與訓練相關）

| 賽事 | 地區 | 賽季 | 場數 |
|------|------|------|------|
| FIFA World Cup | 國際 | 1958/1962/1970/1974/1986/1990/2018/2022 | 148 |
| UEFA Euro | 歐洲 | 2020(辦於2021), 2024 | 102 |
| Copa America | 南美 | 2024 | 32 |
| African Cup of Nations | 非洲 | 2023 | 52 |
| Champions League | 歐洲 | 2009~2019（決賽）+ 早期 | 16 |
| La Liga | 西班牙 | 2004~2021（17 賽季） | 819 |
| Premier League | 英格蘭 | 2003/04, 2015/16 | 418 |
| Bundesliga | 德國 | 2015/16, 2023/24 | 68 |
| Serie A | 義大利 | 1986/87, 2015/16 | 381 |
| Ligue 1 | 法國 | 2015/16, 2021/22, 2022/23 | 435 |

### 事件欄位（kloppy 壓平後）

```
event_id, event_type, period_id, timestamp, end_timestamp,
ball_state, ball_owning_team,
team_id, player_id, receiver_player_id,
coordinates_x, coordinates_y, end_coordinates_x, end_coordinates_y,
result, success,
set_piece_type, body_part_type, pass_type, duel_type, goalkeeper_type,
is_under_pressure, is_counter_attack, card_type
```

event_type 種類（33種）：

PASS, CARRY, PRESSURE, SHOT, DUEL, CLEARANCE,
INTERCEPTION, TAKE_ON, RECOVERY, GOALKEEPER, SUBSTITUTION,
FORMATION_CHANGE, BALL_OUT, CARD, PLAYER_ON/OFF, FOUL_COMMITTED,
MISCONTROL, GENERIC:*

### matches JSON 欄位

```
match_id, match_date, kick_off, competition, season,
home_team, away_team, home_score, away_score,
match_status, match_week, competition_stage, stadium, referee
```

---

## 2. league/ — Understat 六大聯賽（比賽/球員/射門級別，CSV）

**來源**：Kaggle mexwell/understat-database
**格式**：CSV（shot_data / match_info 使用 sep=';'）
**聯賽**：EPL, La_Liga, Bundesliga, Ligue_1, Serie_A, RFPL
**年份**：2014~2024

### season.csv（每隊每場比賽一筆，各聯賽 ~6,000~7,700 筆）

```
id, title(隊名), year, h_a(主/客),
xG, xGA, npxG, npxGA,
deep, deep_allowed,
scored, missed,
xpts, result(w/d/l), date,
wins, draws, loses, pts, npxGD,
ppda.att, ppda.def, ppda_allowed.att, ppda_allowed.def
```

### match_info.csv（每場比賽一筆）

```
id, fid, h(主隊id), a(客隊id), date, league_id, season,
h_goals, a_goals, team_h, team_a,
h_xg, a_xg, h_w, h_d, h_l,
h_shot, a_shot, h_shotOnTarget, a_shotOnTarget,
h_deep, a_deep, a_ppda, h_ppda
```

### player.csv（每球員每賽季一筆）

```
id, player_name, games, time,
goals, xG, assists, xA, shots, key_passes,
yellow_cards, red_cards, position, team_title,
npg, npxG, xGChain, xGBuildup, year
```

### shot_data.csv（每次射門一筆，最細粒度）

```
id, minute, result(Goal/SavedShot/BlockedShot/MissedShots/OwnGoal),
X, Y,
xG, player, h_a, player_id,
situation(OpenPlay/FromCorner/SetPiece/Penalty/DirectFreekick),
season, shotType(RightFoot/LeftFoot/Head),
match_id, h_team, a_team, h_goals, a_goals, date,
player_assisted, lastAction
```

---

## 3. worldCup/ — 世界盃歷史資料（比賽/球員/結果級別，CSV）

### WorldCupMatches.csv（4,572 筆，1930~2014）

```
Year, Datetime, Stage, Stadium, City,
Home Team Name, Home Team Goals, Away Team Goals, Away Team Name,
Win conditions, Attendance,
Half-time Home Goals, Half-time Away Goals,
Referee, RoundID, MatchID,
Home Team Initials, Away Team Initials
```

### WorldCupPlayers.csv（37,784 筆）

```
RoundID, MatchID, Team Initials, Coach Name,
Line-up(S/N), Shirt Number, Player Name, Position, Event
```

### WorldCups.csv（20 筆，1930~2014 冠軍歷史）

```
Year, Country(主辦), Winner, Runners-Up, Third, Fourth,
GoalsScored, QualifiedTeams, MatchesPlayed, Attendance
```

### intl/results.csv（49,353 筆，1872~2026）

```
date, home_team, away_team, home_score, away_score,
tournament, city, country, neutral
```

198 種賽事，含 Friendly、WC Qualification、UEFA Euro Q、Copa América、AFCON 等，涵蓋至 2026-06-27

### intl/goalscorers.csv

```
date, home_team, away_team, team, scorer, minute, own_goal, penalty
```

### intl/shootouts.csv

```
date, home_team, away_team, winner, first_shooter
```

---

## 資料量彙總

| 資料夾 | 粒度 | 筆數 | 年份範圍 |
|--------|------|------|---------|
| mix/events | 逐事件（每場約3,000） | 約1,200萬事件 | 1958~2025 |
| mix/matches | 比賽 | 3,961 場 | 1958~2025 |
| league/season | 隊x場 | 約41,744 筆 | 2014~2024 |
| league/player | 球員x賽季 | 數萬筆 | 2014~2024 |
| league/shot_data | 射門 | 數十萬筆 | 2014~2024 |
| worldCup/matches | 比賽 | 4,572 場 | 1930~2014 |
| worldCup/intl | 國際賽比賽 | 49,353 場 | 1872~2026 |
