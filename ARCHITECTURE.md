# DEUS GLOBAL CORE — SYSTEM ARCHITECTURE
> **For developers and AI assistants**
>
> Comprehensive technical guide on architecture, mathematical algorithms, and project file structure. After reading this document, one must fully understand the system logic and be ready to safely modify the code.

---

## 1. SYSTEM PHILOSOPHY

**Deus Global Core** is a modular **quantamental** trading system for the G8 FX market. It is a **pure macro oracle**: the system ranks 8 currencies by their fundamental strength and generates directional signals. Position sizing and risk management are separated from the main pipeline and are executed via the interactive script `scripts/position_sizer.py`.

**Key principles:**
1. **8 currencies (G8):** USD, EUR, GBP, JPY, CHF, AUD, CAD, NZD.
2. **28 currency pairs:** all combinations C(8,2) = 28.
3. **Daily cycle:** data → normalization → scoring → matrix → filtering → DB.
4. **Hybrid data ingestion:** automatic APIs (FRED, Yahoo Finance, DBnomics, CFTC) + manual input of rare leading indicators (Flash PMI, SNB Deposits, COT Index, CESI) once a week. The **Graceful Degradation** mechanism guarantees resilience against API failures (fallbacks to local caches and proxy metrics).
5. **Fair calibration:** all thresholds are validated by a walk-forward test on 2020–2026 data without lookahead bias. Robustness is evaluated via split-sample testing: In-Sample (IS: 2020–2022) and Out-of-Sample (OOS: 2023–2026) using the Anti-Overfitting Mandate — see Section 9.
6. **Regime-based trade management:** The model is used exclusively as a macro compass for entry with phased scaling (stepwise addition) over a month. Automatic "smart exits" are prohibited, as they break trends.
7. **Regime Dependency:** The model shows maximum advantage only during periods of high inflation and divergence of monetary policies. In the ZIRP (zero interest rate policy) era, the system generates a minimum of signals.

---

## 2. ARCHITECTURAL PIPELINE

```mermaid
graph TD
    subgraph INGESTION [1. Data Ingestion]
        A1[(FRED API)] -->|Auto: rates, CPI, PMI, employment| D_RAW[Raw Data]
        A2[(Yahoo Finance)] -->|Auto: FX pairs, commodities, VIX, DXY| D_RAW
        A3[(manual_alpha.json / manual_yields.json)] -->|Manual entry weekly| D_RAW
        A4[(DBnomics OECD)] -->|Auto: Macro Surprise Index| D_RAW
        A5[(CFTC COT)] -->|Auto: Net positions of speculators| D_RAW
        A6[(RSS Google News)] -->|Auto: Sentiment + Panic Detector| D_RAW
    end

    subgraph NORMALIZATION [2. Normalization]
        D_RAW --> B1[Rolling Z-score Engine]
        B1 -->|z_window=180 days daily / 24 months monthly| C_SCORER
    end

    subgraph SCORING [3. Scoring]
        D2[Sentiment 3pct + Panic + MacroSurprise + RegimeFilter] --> D1
        D1 -->|Final Composite Score (Point-in-Time)| E_MATRIX[MatrixEngine]
    end

    subgraph MATRIX [4. Matrix & Filtering]
        E_MATRIX -->|Signal = Score_A - Score_B, 28 pairs| F_VETO[VetoFilters]
        G_COT[(COT Z-score)] -->|Diagnostic / Info-only| F_VETO
        F_VETO -->|Clean signals| H_OUT[Signals]
    end

    subgraph OUTPUT [5. Output]
        H_OUT --> DB[(data/deus.db SQLite)]
        H_OUT --> CONSOLE[Console Scorecard]
    end
```

---

## 3. MATHEMATICAL CORE

### 3.1. Rolling Z-score with data frequency adaptation

All macroeconomic metrics are normalized via `rolling(window).mean()` / `rolling(window).std()`:

```
# Hybrid Z-score (Fast/Slow windows) to suppress noise.
# Cold-start protection: slow window is filled by fast (fillna),
# preventing NaN propagation without data leaks.
Z = (X - rolling_mean) / rolling_std
Z = clip(Z, -3.0, +3.0)
```

**Critically important — the `_freq_window(series, base_window)` method** in `CurrencyScorer`:

| Data Frequency | Median Interval | Window with base=180 |
|---|---|---|
| Daily (yields, VIX) | ≤ 20 days | 180 trading days |
| Monthly (CB rates, CPI, PMI) | 21–60 days | max(24, 180//30) = **24 months** |
| Quarterly (CPI AUD/NZD) | > 60 days | max(8, 180//91) = **8 quarters** |

The `z_score_window: 180` parameter is taken from `config.yaml`. **MatrixEngine reads it during initialization** — do not pass `z_window` manually unless there is a special need.

### 3.2. Composite Score

```
Composite = Σ(Z_i × w_i) / Σ|w_i|
```

Weights are normalized dynamically — only across metrics that are actually available on the current date. If CHF is missing NFP, the other groups proportionally receive a larger weight.

### 3.2.1. Cycle Blender and stagflation integration

With `CycleBlender` enabled, the Composite Score calculation transitions to dynamic blending of business cycle phases:
1. For each factor group, the net average Z-score of its active incoming metrics is calculated (taking into account their signs and stagflation adjustment).
2. The stagflation filter is applied before calculating the group averages: if CPI Z > 0.0 against a falling PMI, the weights of inflation and PMI are dynamically reduced (or inverted).
3. The average values for the groups are weighted by the coefficients of the current cycle phase (Expansion, Slowdown, Contraction, Recovery) to obtain the base composite.
4. The base composite is compressed via `variance_penalty` and supplemented by external overlays (`regime_adjustment` and biases).
5. In `score_components` (scorecard on screen), the group contributions are displayed as net blended contributions:
   $$groups[grp] = raw\_group\_score \times \frac{w^{blend}_{grp}}{\sum w^{blend}} \times VP$$
   This eliminates double weighting, ensuring that the sum of the components is identically equal to the final Composite Score.

### 3.3. Matrix Signal

```
Signal(A/B) = CompositeScore(A) − CompositeScore(B)
```

If `|Signal| ≥ ENTRY_THRESHOLD (1.5 σ)` → the signal passes to the filters.

**Sign and direction:**
- `Signal > 0` → A is stronger than B → LONG A/B
- `Signal < 0` → B is stronger than A → SHORT A/B

The ticker direction is determined via `TICKER_MAP` in `backtesting/__init__.py`. For inverted pairs (e.g. USDJPY when JPY/USD is signaled), the system automatically inverts the sign.

### 3.4. Entry and Exit Rules (Variant B)

#### Entry Conditions (Position Opening)
Opening a new position requires meeting **three sequential conditions** (Variant B — no regime pair veto):
1. **Threshold Discrepancy:** The magnitude of the absolute signal $|Signal| = |Score_A - Score_B| \ge 1.5\sigma$.
2. **Opposite Sides:** The long currency must have a strictly positive composite score ($Score_{long} > 0$), and the short currency must have a strictly negative score ($Score_{short} < 0$).
3. **Three-day Stability:** The signal must maintain its direction (the same sign) for $3$ or more consecutive trading days.
*(Note: The regime universe pair veto is disabled. All active pairs Tier 1–3 are traded in any regime without universe restrictions).*

#### Exit Conditions (Position Closing)
Position exit and signal deactivation occur when any of the following conditions are met:
1. **Signal Decay:** 
   - The absolute signal $|Signal| = |Score_A - Score_B|$ falls below the threshold **$1.4\sigma$** (immediately), **OR**
   - The absolute signal falls below the threshold **$1.5\sigma$** and stays below this level for **2 consecutive days** (2 days of stability).
2. **Signal Reversal:** The signs of the scores reverse (the long currency becomes $\le 0$, or the short currency becomes $\ge 0$).
3. **Time Limit (Max Hold Horizon):** Forced position closure after **63 trading days**.

> [!NOTE]
> The logic of exits by signal decay has been tested in detail on historical IS (2020-2022) and OOS (2023-2026) periods. The results and comparison of various exit thresholds (0.75σ, 1.0σ, 1.5σ) are documented in the report [stop_loss_decay_backtest_report.md](docs/stop_loss_decay_backtest_report.md).

### 3.5. Trading Universe and Pair Tiers
Currency pairs are classified into tiers based on their historical performance during the walk-forward backtest period of 2020-2026. Complete win rate, profit factor, and blacklist statistics are located in the file [PAIR_TIERS.md](PAIR_TIERS.md):
- **Tier 1 (High Quality, PF > 3.0):** GBP/JPY, EUR/JPY, USD/JPY, JPY/AUD.
- **Tier 2 (Good Quality, PF > 1.4):** USD/GBP, JPY/CAD, JPY/NZD, USD/AUD, USD/NZD, CHF/AUD.
- **Tier 3 (Marginal, PF > 1.0):** EUR/CHF, USD/CAD, GBP/NZD.
- **BLACKLIST (Negative Expectation or Insufficient Data, PF < 1.0):** USD/CHF, CHF/CAD, CHF/NZD, GBP/CHF, EUR/CAD, USD/EUR, CAD/NZD, EUR/NZD, JPY/CHF, EUR/GBP, EUR/AUD, GBP/CAD, AUD/CAD, AUD/NZD, GBP/AUD. These pairs are strictly excluded from trading.

### 3.6. Regime Filtering (Disabled for Pair Selection)
The `RegimeFilter` module continues to evaluate the market regime across three states (`Risk-On`, `Neutral`, `Risk-Off`) for macro context analysis and logging:
- **Risk-On:** regime_score > 0.3.
- **Neutral:** -0.3 <= regime_score <= 0.3.
- **Risk-Off:** regime_score < -0.3.

> [!IMPORTANT]
> **Universe Filtering Disabled (Variant B):** To mitigate overfitting and increase the portfolio Sharpe ratio on OOS data (OOS Sharpe improvement of +0.1035), the regime universe filter in `get_active_pairs` is completely disabled. All active pairs Tier 1–3 are traded in all regimes without limitations. The status `[REGIME FILTERED]` is no longer assigned to active pairs.

### 3.7. Risk Management and Lot Sizing (Position Sizer)
The size of the opened macro position is calculated dynamically based on the daily volatility (D1) or weekly volatility (W1) of the instrument smoothed via ATR(14) using the formula:

\[Size (lots) = \frac{Risk (\$)}{StopPips \times PipValue}\]

where:
1. **Risk ($)**: Target risk in USD, determined from the account balance (e.g., $Risk = Account \times Risk\%$).
2. **StopPips**: Distance of the virtual stop-loss in pips, calculated as:
   \[StopPips = \frac{ATR(14) \times Multiplier}{PipSize}\]
   - *PipSize* is `0.01` for pairs with JPY and `0.0001` for other pairs.
   - *Multiplier* — a scaling factor for medium-term holding (recommended range: `1.5 - 3.0`).
3. **PipValue**: Dollar value of one pip per standard lot ($100,000$ units of the base currency).
   - If the quote currency of the pair is USD (e.g., EUR/USD): $PipValue = 100,000 \times PipSize = \$10.0$.
   - If the base currency of the pair is USD (e.g., USD/CAD): $PipValue = \frac{100,000 \times PipSize}{Rate}$.
   - For cross rates (e.g., EUR/GBP): $PipValue = (100,000 \times PipSize) \times CrossUSDRate$, where $CrossUSDRate$ is the current rate of GBP/USD (the quote currency to USD).

The script rounds the final value down to the nearest step of $0.01$ lot and outputs the actual macro risk of the trade taking rounding into account.

---

## 4. COMPONENT MAP

### 4.1. Configuration

| File | Purpose |
|---|---|
| [config.yaml](config.yaml) | FRED tickers for 8 currencies, Yahoo tickers for FX/commodities/VIX, parameters: `z_score_window: 180`, `z_score_clip: 3.0` |
| [.env](.env) | `FRED_API_KEY`, `GEMINI_API_KEY` (key for free sentiment analysis) |
| [data/manual_alpha.json](data/manual_alpha.json) | Flash PMI, CESI, SNB Deposits, COT Index, Rate Expectations |
| [data/manual_yields.json](data/manual_yields.json) | Yields, CHF/NZD unemployment, iron ore momentum |

### 4.2. Data Ingestion (`modules/data_ingestion/`)

| File | Purpose |
|---|---|
| [fred_connector.py](modules/data_ingestion/fred_connector.py) | FRED API: rates, CPI, PMI, employment |
| [yahoo_connector.py](modules/data_ingestion/yahoo_connector.py) | Yahoo Finance: FX pairs, VIX, HY Spread, commodities |
| [cache_manager.py](modules/data_ingestion/cache_manager.py) | Parquet cache to speed up and protect against rate limits |
| [dbnomics_connector.py](modules/data_ingestion/dbnomics_connector.py) | OECD data for Macro Surprise Index |
| [sentiment_connector.py](modules/data_ingestion/sentiment_connector.py) | RSS Google News + VADER + Panic Detector (console offline) |
| [gemini_sentiment_connector.py](modules/data_ingestion/gemini_sentiment_connector.py) | AI sentiment via Google Gemini API REST with cached results |
| [cftc_connector.py](modules/data_ingestion/cftc_connector.py) | COT data (Socrata API) |
| [snb_connector.py](modules/data_ingestion/snb_connector.py) | SNB API: automatic ingestion of Swiss sight deposits with fallbacks |
| [manual_alpha_loader.py](modules/data_ingestion/manual_alpha_loader.py) | Reads `manual_alpha.json` |
| [manual_yields_loader.py](modules/data_ingestion/manual_yields_loader.py) | Reads `manual_yields.json` + unemployment_manual for CHF/NZD |

### 4.3. Normalization (`modules/normalization/`)

| File | Purpose |
|---|---|
| [z_score_engine.py](modules/normalization/z_score_engine.py) | `ZScoreEngine`, `rolling_z_score`, `adaptive_z_score` |

### 4.4. Scoring (`modules/scoring/`)

| File | Purpose |
|---|---|
| [currency_scorer.py](modules/scoring/currency_scorer.py) | Main class `CurrencyScorer`. Method `_freq_window()` scales the z-window to the frequency. Method `score_series()` returns a time series of Composite Score |
| [macro_overlay.py](modules/scoring/macro_overlay.py) | Applies sentiment (3%), MacroSurprise, BTP-Bund spread to the historical series |

### 4.5. Matrix and Context (`modules/matrix/`, `modules/context/`)

| File | Purpose |
|---|---|
| [matrix_engine.py](modules/matrix/matrix_engine.py) | `MatrixEngine`: reads `z_score_window` from `config.yaml`, runs `CurrencyScorer` × 8 (passing it `RegimeFilter` for row-by-row point-in-time evaluation without look-ahead bias), builds a 28-row matrix |
| [regime_filter.py](modules/context/regime_filter.py) | Market regime by VIX + HY Spread + Carry Unwind Detector (USDJPY/USDCHF). Integrated within `CurrencyScorer` |

### 4.6. Execution (`execution/`)

| File | Purpose |
|---|---|
| [veto_filters.py](execution/veto_filters.py) | COT veto (overbought/oversold > 2.0σ), confidence filter |
| [pair_selector.py](execution/pair_selector.py) | Selection of traded pairs from those that passed the filter |

### 4.7. Backtesting (`backtesting/`)

| File | Purpose |
|---|---|
| [__init__.py](backtesting/__init__.py) | Constants `G10`, `TICKER_MAP`, function `_lookup_ticker` |
| [vector_evaluator.py](backtesting/vector_evaluator.py) | Predictive power test: Hit Rate and IC over horizons 5/21/63 days and thresholds 0.5–2.0σ |
| [scripts/](backtesting/scripts/) | Scripts to run vector tests and simulations (e.g., `run_vector.py`) |

> [!NOTE]
> PnL simulator, BacktestEngine, Grid Search Optimizer, and VectorBT Bridge have been removed. The system does not simulate trades — it only evaluates the predictive power of the macro narrative via Hit Rate and Information Coefficient.

### 4.8. Scripts (`scripts/`)

| File | Purpose |
|---|---|
| [live_run.py](scripts/live_run.py) | Production daily script. Thresholds are read from `config.yaml`. COT is disabled by default. Automatically rolls back calculation date to the last trading Friday when run on weekends. |
| [index_run.py](scripts/index_run.py) | Pipeline script: daily calculation and signal generation for G8 stock indices. |
| [index_backtest.py](scripts/index_backtest.py) | Vector backtest script for G8 stock indices on monthly data. |
| [telegram_bot.py](scripts/telegram_bot.py) | Async Telegram bot (whitelisting, FSM input, Matplotlib charts, lot sizer calculator) |
| [position_sizer.py](scripts/position_sizer.py) | Interactive position size (lot) calculator based on a given risk ($) and ATR(14) volatility |
| [update_alpha.py](scripts/update_alpha.py) | Interactive input for `manual_alpha.json` |
| [update_yields.py](scripts/update_yields.py) | Interactive input for `manual_yields.json` |
| [smoke_test.py](scripts/smoke_test.py) | Functionality check of all connectors |
| [web_dashboard.py](scripts/web_dashboard.py) | Local web workspace (zero-dependency Web UI, port 8080) with Gemini and VADER support |
| [live_media_dashboard.py](scripts/live_media_dashboard.py) | Interactive console news dashboard with Gemini/VADER support |
| [run_gemini_sentiment.py](scripts/run_gemini_sentiment.py) | CLI script to run AI sentiment analysis of news/CB statements |

---

## 5. DATABASE SCHEMA

```sql
-- Run sessions
CREATE TABLE IF NOT EXISTS sessions (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    run_date    TEXT    NOT NULL,           -- YYYY-MM-DD
    run_ts      TEXT    NOT NULL,           -- ISO timestamp of run
    regime      TEXT,
    vix         REAL,
    hy_spread   REAL,
    g4_liquidity_z REAL,
    threshold   REAL,
    n_signals   INTEGER DEFAULT 0,
    notes       TEXT
);

-- Component-level currency scoring
CREATE TABLE IF NOT EXISTS scores (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      INTEGER NOT NULL REFERENCES sessions(id),
    run_date        TEXT    NOT NULL,
    currency        TEXT    NOT NULL,
    composite       REAL,
    monetary_z      REAL,
    growth_z        REAL,
    inflation_z     REAL,
    credit_z        REAL,
    commodities_z   REAL,
    alpha_z         REAL
);

-- Signals
CREATE TABLE IF NOT EXISTS signals (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      INTEGER NOT NULL REFERENCES sessions(id),
    run_date        TEXT    NOT NULL,
    pair            TEXT    NOT NULL,       -- e.g. "AUD/CHF"
    long_ccy        TEXT,
    short_ccy       TEXT,
    direction       TEXT    DEFAULT 'long',
    signal_strength REAL,
    oos_sharpe      REAL,
    ticker          TEXT,
    signal_status   TEXT    DEFAULT 'NEW',  -- NEW | ACTIVE | CLOSED
    signal_days     INTEGER DEFAULT 1,      -- consecutive days signal is active
    status          TEXT    DEFAULT 'open', -- open | closed | expired
    close_date      TEXT,
    entry_price     REAL,
    exit_price      REAL,
    pnl_pct         REAL,
    pnl_bp          REAL
);

-- Snapshot of manual data (for reproducibility)
CREATE TABLE IF NOT EXISTS alpha_history (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id  INTEGER NOT NULL REFERENCES sessions(id),
    run_date    TEXT    NOT NULL,
    currency    TEXT    NOT NULL,
    cesi        REAL,
    rate_exp    REAL,
    gdt_dairy   REAL,
    snb_dep     REAL,
    diy_cesi    REAL
);

-- Equity Indices Signals
CREATE TABLE IF NOT EXISTS index_signals (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      INTEGER NOT NULL REFERENCES sessions(id),
    run_date        TEXT    NOT NULL,
    instrument      TEXT    NOT NULL,       -- e.g. "SP500"
    country         TEXT    NOT NULL,       -- e.g. "USD"
    direction       TEXT    NOT NULL,       -- LONG | SHORT
    signal_strength REAL,
    vix_entry       REAL,
    signal_status   TEXT    DEFAULT 'NEW',  -- NEW | ACTIVE | CLOSED
    signal_days     INTEGER DEFAULT 1,
    status          TEXT    DEFAULT 'open', -- open | closed
    close_date      TEXT,
    entry_price     REAL,
    exit_price      REAL,
    pnl_pct         REAL,
    pnl_bp          REAL
);
```

---

## 6. DEVELOPMENT INSTRUCTIONS

### 6.1. Adding a new FRED metric

1. Add the ticker in `config.yaml` under the appropriate currency.
2. In `currency_scorer.py`, assign it to a group in `COMPONENT_GROUPS`.
3. If necessary, specify the preprocessing logic in `_preprocess()`.
4. Run `python scripts/live_run.py --force-refresh`.

### 6.2. Adding a manual metric

1. Add a field in `data/manual_alpha.json` or `data/manual_yields.json`.
2. Ensure that `ManualAlphaLoader` / `ManualYieldsLoader` reads the new field.
3. Add a weight to `DEFAULT_WEIGHTS` in `currency_scorer.py`.

### 6.3. Changing the Z-window

Change only in `config.yaml`:
```yaml
parameters:
  z_score_window: 180
```
`MatrixEngine` reads this value automatically upon initialization.

### 6.4. Changing the entry threshold

Change only in `config.yaml`:
```yaml
parameters:
  entry_threshold: 1.5
```
The `live_run.py` script and optimizers will read it automatically.

### 6.5. Working with AI sentiment and external connectors

#### 6.5.1. AI Sentiment (Gemini 3.1 Flash Lite)
- **Configuration**: Key parameters for AI sentiment are configured in `.env` (`GEMINI_API_KEY`).
- **Default Model**: `gemini-3.1-flash-lite` (limit of 1500 requests/day, minimal response latency).
- **Caching**: Results of requests to Gemini are cached locally by news text hash in `data/cache/gemini_sentiment_cache.json`. Repeated scans on the same news do not consume the API quota.
- **Support for Links (`HeadlineStr`)**: To maintain backward compatibility with old paths (DataFrame, JSON cache, SQLite), a `HeadlineStr(str)` class has been implemented. It behaves like a normal string but contains a `.link` attribute. This allows passing the article's URL to the sentiment connector without changing the data schema.
- **Parallel Scraping of Full Text**: For batch media analysis (`analyze_headlines_batch`), the system uses `concurrent.futures.ThreadPoolExecutor` to download HTML pages in parallel (up to 5 threads) and parse `<p>` content using the `BeautifulSoup4` library. This allows extracting clean text from articles in less than 2.5 seconds total, giving Gemini full article context instead of a short headline.
- **Prompt Engineering Rules (Noise vs. Signal)**:
  - The Gemini prompt strictly cuts out market noise, analytical commentary, and speculation (which are assigned a neutral sentiment of `0.0`).
  - Only real fundamental events are evaluated: interest rate changes, quantitative easing/tightening (QE/QT) programs, hard macroeconomic releases, and official CB forecasts.
- **VADER Fallback**: If limits are exceeded (HTTP 429 status code) or a network failure occurs, REST requests automatically fail over to the offline VADER module. The response returns a `"fallback": "VADER"` flag, which is correctly processed in the local Web UI ([web_dashboard.py](scripts/web_dashboard.py)) and the console ([live_media_dashboard.py](scripts/live_media_dashboard.py)).

#### 6.5.2. Specifics of External Connectors (G8 Central Banks)
When integrating external macro data connectors, the following technical characteristics must be taken into account:
1. **BIS REER (`bis_data.py`):** Requests must filter by countries (e.g. `countries=['US']`), otherwise the BIS server times out (30 seconds) due to the volume of returned XML.
2. **BoC BCPI (`boc_data.py`):** Series codes for annual commodity indices (`A.BCPI`) have a 1.4-year lag. To reduce the lag to ~41 days, monthly series should be requested (prefix `M.` instead of `A.`: `M.BCPI`, `M.ENER`, etc.).
3. **RBA (`rba_data.py`):** Blocked by Akamai CDN when called from cloud networks. Requires a local proxy or VPN to bypass.
4. **SNB ([snb_connector.py](modules/data_ingestion/snb_connector.py)):** Ingestion of Swiss sight deposits under the `snbgwdmigirow` cube is fully automated. Weekly Z-scores are calculated via `adaptive_z_score` (slow_hl=26, fast_hl=6, min_periods=30, clip=3.0). The resilience mechanism supports automatic fallback to a local CSV (`data/snb-data-snbgwdmigirow-en-all-*.csv`) or manual FSM input in `manual_alpha.json` (`snb_dep`). The policy rate series `snbgwdzid` (Policy Rate, SARON) works stably.
5. **OECD GDP (`oecd_data.py`):** Requires a 13-dimensional key structure (SDMX v2) and the `defusedxml` library to parse responses.

---

## 7. ARCHITECTURAL DECISIONS

- **Adaptive Hybrid EWM Z-score:** A hybrid Z-score (a statistical measure of value deviation from the mean) using exponential smoothing (EWM — Exponential Moving Average). Fast and slow windows smooth out noise, and scaling the window to the publication frequency (daily/monthly/quarterly) protects against false signals on rare data.
- **Stagflation Adjustment:** An adaptive algorithm that recognizes stagflation (high inflation + falling PMI). When activated, inflation begins to be interpreted negatively for the currency (since the CB cannot hike rates due to recession threat), and the weights of growth indicators are reduced.
- **Toxic Yield Trap:** Protection mechanism against "toxic yield". During market panic, high yield on a currency (e.g. of emerging economies) signals sovereign risk rather than economic strength. The algorithm inverts the yield's impact on the score if there is a high correlation with VIX.
- **Inverse Correlation Weighting (ICW) (DISABLED):** Weight decorrelation method within a single indicator group (e.g. inflation). Disabled in favor of Equal Weighting (Occam's razor) to eliminate multicollinearity in a simpler and more reliable way, while lowering the weight of TIPS/real_yield_10y to 0.65.
- **Dynamic entry threshold (top 5%):** Floating signal generation threshold calculated as the 95th percentile of the historical distribution of matrix signals. Guarantees that the system opens trades only on the strongest fundamental discrepancies.
- **G4 Liquidity Overlay (DISABLED):** Global liquidity overlay (Fed, ECB, and BOJ balance sheets). Disabled (`liq_weight = 0.00`) per Anti-Overfitting Mandate.

---

## 8. DISABLED MODULES

| Module | Reason for Disabling | Status |
|---|---|---|
| **COT Veto Filter** | Disabled in favor of manual entry control. Automatic vetoes often blocked profitable trades on strong fundamental trends. | Disabled (`ENABLE_COT_VETO=False`, diagnostic/info mode) |
| **Credit Factor (credit_z)** | Lack of high-quality, high-frequency credit spread data for all G10 currencies. | Excluded from scoring and database logs |
| **Curve Classifier Overlay** | Historical backtests showed deterioration of the Sharpe ratio in the out-of-sample period. | Disabled (`curve_regime_weight=0.0`) |
| **G4 Liquidity Overlay** | Deteriorated Sharpe ratio in the OOS period. Disabled per Anti-Overfitting Mandate. | Disabled (`liq_weight=0.00`) |
| **Inverse Correlation Weighting (ICW)** | Complex correlation math led to overfitting. Replaced by simple Equal Weighting (Occam's razor). | Disabled (replaced by simple average) |
| **Variance Mitigation** | Dispersion correction via `sqrt(n/11)` scaling. The workaround was removed after aligning currency coverage. | Disabled (`variance_penalty=1.0`) |
| **China Credit Impulse (AkShare)** | Toxic effect on performance (-0.08 to Sharpe). Data instability. | Disabled (`ENABLE_CHINA_V3=False`) |
| **ACLED Geopolitics** | API instability, inability to construct a fair point-in-time signal without lookahead bias. | Disabled (`ENABLE_GEOPOLITICS_V3=False`) |
| **Calendar Macro Surprise V3** | Complete duplication of CESI. Zero contribution to Sharpe. | Disabled (`ENABLE_CALENDAR_V3=False`, weight=0.00) |
| **TRADE_BALANCE (AUD)** | Disabled per Anti-Overfitting Mandate. A lookahead artifact was identified in the old lag calibration (+35d from observation date instead of +35d from end-of-month). After switching to the true point-in-time ABS release calendar (31–40d lag), the metric deteriorates Sharpe on active pairs (AUD/USD: -1.48). | Disabled (commented out in SURPRISE_EVENTS, weight=0.00) |

---

## 9. SYSTEM LIMITATIONS

- **Valid Out-of-Sample (OOS) test implemented:** The obsolete limitation regarding the absence of OOS validation has been resolved. The system now utilizes a walk-forward split: In-Sample (IS: 2020–2022) for calibration and Out-of-Sample (OOS: 2023–2026) for independent verification. A strict Anti-Overfitting Mandate is in place (any changes, such as Variant B, are accepted only if they yield $\Delta\text{Sharpe}_{\text{OOS}} \ge +0.05$).
- Dependence on manual data entry weekly (the `manual_alpha.json` and `manual_yields.json` files are critical for operation).
- The system does not automatically block trading in Risk-Off — it only adjusts weights; the decision to halt is made by the trader.
- Backtest IC (Information Coefficient — a measure of correlation between prediction and actual return) Decay is not representative of the live system, since the historical data before 2020 lacks CESI, Flash PMI, and rate expectations.
- The 2025 results are explained by low score variance (lack of macro trends) and the April Risk-Off episode, rather than a structural issue with the system.

---

## 10. QUICK CODE EXAMPLES

### Get scores and matrix:
```python
from modules.matrix.matrix_engine import MatrixEngine

engine = MatrixEngine(use_regime=True)  # z_window is read from config.yaml
engine.run()

scores = engine.get_scores()       # pd.Series: currency → composite score
matrix = engine.get_matrix()       # pd.DataFrame: 28 rows, signal, ticker, direction
signals = engine.get_top_signals() # Top signals above sigma_threshold
```

### Run a vector test:
```python
from modules.matrix.matrix_engine import MatrixEngine
from backtesting.vector_evaluator import VectorEvaluator

engine = MatrixEngine(start_date="2020-01-01", use_regime=False)
engine.run()

evaluator = VectorEvaluator(score_df=engine.get_score_series())
results = evaluator.run()
# results["21d"]["hr_>=1.0"] → Hit Rate at threshold 1.0 on a 21-day horizon
```

---

## 11. CALIBRATION DATA (current on dev/razor)

⚠️ The data is provided for informational purposes only. The only valid validation is via a forward test on real signals for at least 6 months.

Backtest results for the production strategy Variant B (No Regime Filter, no pair regime veto) with dynamic exit based on signal decay (Signal Decay / Entry Threshold 1.5σ / 1.4σ) for the OOS period 2023–2026:

| Metric | Value |
|---|---|
| Portfolio Sharpe Ratio | **+0.9770** |
| Overall Win Rate (Hit Rate) | 59.50% |
| Overall Profit Factor | 2.55 |
| Average Hold (days) | 14.9 |
| Number of Closed Trades (N) | 200 |

*(For comparison, a theoretical vector backtest with a fixed 63-day horizon without decay exits shows a Hit Rate of 69.62% and a Profit Factor of 3.95).*

Results by market regimes for Variant B (OOS 2023–2026):

| Regime | Trades (N) | Win Rate | Profit Factor |
|---|---|---|---|
| **RISK_ON** (regime_score > 0.3) | 166 | 62.7% | **3.57** |
| **NEUTRAL** (-0.3 <= regime_score <= 0.3) | 27 | 37.0% | **0.19** |
| **RISK_OFF** (regime_score < -0.3) | 7 | 71.4% | **1.18** |

### Additional stability metrics (dev/razor)
- Results are not dependent on outliers: the contribution of the top 10% of trades to total profit is less than 33% for all Tier 1 pairs (lowest contribution is CAD/NZD at 14.63%, highest is USD/JPY at 32.75%).
- Excluding the 5 best trades reduces the Profit Factor of GBP/JPY Risk-Off from 138.03 to 122.31 (-11.39%), and CAD/NZD Neutral from 326.18 to 295.51 (-9.40%), confirming the systematic nature of the returns.

---

## 12. TELEGRAM BOT ARCHITECTURE

The async Telegram bot (`scripts/telegram_bot.py`) is built on the `aiogram` (v3) framework and is fully integrated with the database and connectors of Deus Global Core.

### 12.1. Security and Data Storage
- **Whitelisting**: The bot filters all incoming messages and callback queries against the `ALLOWED_TELEGRAM_USERS` list in `.env`. Unauthorized users receive a concise `"❌ Access denied."` message.
- **Local Databases**:
  - `data/deus.db`: Read historical sessions, currency scores, active signals, and historical logs.
  - `data/manual_updates.db`: Log the history of manual parameter updates (who, when, and which value was updated).

### 12.2. Finite State Machine (FSM) Architecture
Dialogue scenarios and interactive parameter inputs are separated into independent FSM states:
1. **`ManualInputStates`** (Manual input of indicators):
   - Choose update category: `Alpha parameters` or `Yields/rates`.
   - Select G10 currency ➜ select specific metric.
   - Enter new numerical value with validation. When entering economic surprises (DIY Surprise), the FSM sequentially requests `Actual` and `Forecast` values and automatically calculates the delta.
   - The "🔄 Update COT (CFTC) reports" button is integrated directly into the manual alpha input interface.
2. **`CBReleaseStates`** (CB Press Release Analysis):
   - When clicking the `"📝 Analyze CB Release"` button, the bot requests the statement text.
   - The submitted text is sent to `GeminiSentimentConnector.analyze_text` with a deep macro analysis prompt and returned to the user as a structured table.
3. **`SizerStates`** (Step-by-step lot calculator):
   - The wizard step-by-step requests: currency pair ➜ account balance ➜ risk % ➜ ATR(14) value ➜ ATR multiplier.
   - Returns a detailed calculation (Pip Value, exact and working lots, actual risk).

### 12.3. Interactive Menu Sections
- **📊 Scoring & Signals**: Displays the G10 Scorecard and active signals from the database. Includes an inline button to launch the full calculation pipeline (`scripts/live_run.py`) in the background with interactive message status updates ("⏳ Calculating..." ➜ "✅ Done").
- **📰 Sentiment & Media**: Displays aggregated sentiment scores from the RSS feed. A popup system notification (toast) shows the timestamp of the last CSV news database update. The Gemini Research module shows a detailed text report with the run timestamp.
- **📜 Signal History**: Interactive menu with database record counters across 5 categories (`🟢 Active`, `🔴 Closed`, `⚫️ Blacklisted`, `⚪️ Regime Filtered`, `📋 Full History`). Supports nested navigation and return to categories.
- **📈 Macro Metrics**: Dynamically generates a Matplotlib chart of the composite score trend for the last 30 runs in a dark neon style. The image is sent to the user as a PNG.
