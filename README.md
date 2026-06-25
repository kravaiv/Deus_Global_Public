# Deus Global Core

**Deus Global Core** is a quantitative macroeconomic ranking system for G8 currencies (USD, EUR, GBP, JPY, CHF, AUD, CAD, NZD) and a generator of directional trading signals in the FX market.

The system analyzes fundamental discrepancies between countries, normalizes them via a rolling Z-score (a statistical measure showing the deviation of the current value from the historical average in standard deviations), and generates signals based on a matrix of 28 cross rates. It is a **pure macro oracle**: no stop-losses, position sizing, or PnL simulation — only a directional fundamental narrative.

---

## 🧠 Strategy Core

### 1. Normalization (Rolling Z-Score)
All macroeconomic metrics are normalized using a **180-day rolling window** (`z_score_window: 180` in `config.yaml`). For monthly series (CB rates, CPI, unemployment), the window is automatically scaled to 24 months, and for quarterly series — to 8 quarters.

### 2. Scorecard (Currency Scorer)
For each of the 8 currencies, a Composite Score (a weighted sum of all macroeconomic indicators of the currency reflecting its fundamental strength) is formed from 5 groups of macro factors:
- **Monetary Policy** (~30%): CB rate, 10Y real yield (TIPS for USD), yield curve slope (10Y−2Y), money supply (M2 YoY)
- **Growth / PMI** (~20%): PMI, employment, unemployment rate
- **Inflation** (~15%): CPI deviation from the CB target, core PCE
- **Terms of Trade** (~15%): Commodity currency exposure to oil, copper, wheat, iron ore
- **Macro Surprises** (~20%): CESI, interest rate expectations. *(COT is output for diagnostics only)*

When `Cycle Blender` is active, the weights of these groups adapt dynamically to the current phase of the cycle (Expansion, Slowdown, Contraction, Recovery) with smooth blending at the boundaries.

### 3. Entry Rules (Signal Generation)
An entry signal is generated when **four filters** are met simultaneously:
1. **Threshold:** The signal differential $|Score_A - Score_B| \ge 1.5\sigma$.
2. **Opposite Sides:** The long currency must have a positive score ($Score_{long} > 0$), and the short currency must have a negative score ($Score_{short} < 0$).
3. **Stability:** The signal must remain active in the same direction for **3+ consecutive days** (trend confirmation).
4. **Regime Universe filter:** The pair must be included in the active instruments list for the current market regime (see the "Regime Trading Universes" section).

### 4. Exit Rules
An open position is exited and the signal is closed when any of the following conditions are met:
1. **Signal Decay:** The absolute differential $|Score_A - Score_B|$ falls below the exit threshold of **$1.0\sigma$**.
2. **Signal Reversal:** The signs of the currency scores of the pair reverse (the long currency becomes $\le 0$, or the short currency becomes $\ge 0$).
3. **Forced Time Exit:** A forced exit after **63 trading days** of holding.

---

## 📂 Project Structure

```text
Deus_Global_v1/
├── PAIR_TIERS.md             # Classification of currency pairs by tiers and blacklist
├── backtesting/
│   ├── __init__.py           # G10, TICKER_MAP, _lookup_ticker
│   └── vector_evaluator.py   # Testing predictive power (Hit Rate / IC)
├── backtesting/scripts/      # Backtesting scripts
│   ├── isolated_signal_decay_backtest.py # Signal decay exit test (1.0σ / 0.75σ)
│   ├── isolated_stop_backtest.py         # Test of isolated stop-losses
│   └── isolated_trailing_backtest.py     # Test of trailing stop-losses
├── data/
│   ├── cache/                # Parquet cache of API requests and Gemini cache
│   ├── historical/           # Historical data
│   ├── manual_alpha.json     # Manual entry: Flash PMI, Economic Surprise, COT Index
│   └── manual_yields.json    # Manual entry: yields, unemployment CHF/NZD
├── docs/
│   ├── stop_loss_decay_backtest_report.md # Report on signal decay exit backtest
│   ├── stop_loss_backtest_report.md       # Report on hard stop-losses backtest
│   └── stop_loss_trailing_backtest_report.md # Report on trailing stop-losses
├── execution/
│   ├── veto_filters.py       # COT veto and signal filtering
│   └── pair_selector.py      # Selection of traded pairs and regime universes
├── modules/
│   ├── context/              # Regime Filter (VIX, HY Spread, Carry Unwind)
│   ├── data_ingestion/       # Connectors: FRED, Yahoo Finance, DBnomics, CFTC, RSS
│   │   └── gemini_sentiment_connector.py # AI sentiment + BeautifulSoup parsing
│   ├── matrix/               # MatrixEngine — ranking and pair matrix construction
│   ├── normalization/        # Z-score Engine
│   └── scoring/              # CurrencyScorer, MacroOverlay, CycleBlender
├── scripts/
│   ├── live_run.py           # Production script: daily signal generation
│   ├── index_run.py          # Pipeline script: daily equity index signals
│   ├── index_backtest.py     # Vector backtest script for equity indices
│   ├── telegram_bot.py       # Async Telegram bot (whitelisting, FSM, charts)
│   ├── position_sizer.py     # Interactive CLI position size (lot) calculator
│   ├── update_alpha.py       # Interactive entry for manual_alpha.json
│   ├── update_yields.py      # Interactive entry for manual_yields.json
│   ├── smoke_test.py         # Verification of connector functionality
│   ├── web_dashboard.py      # Web UI Dashboard (local port 8080)
│   ├── live_media_dashboard.py # Console media dashboard (Gemini / VADER)
│   └── run_gemini_sentiment.py # CLI for evaluating news/documents via Gemini
├── config.yaml               # FRED/Yahoo tickers, parameters
└── .env                      # API keys, tokens, and Telegram whitelist
```

---

## 🚀 Execution Instructions

**Daily Signal Generation:**
```bash
python scripts/live_run.py
```
*Note: If the script is launched on a weekend (Saturday or Sunday), it automatically rolls back the calculation date to the last trading Friday to prevent calculations on incomplete or missing weekend data.*

**Dry Run (output to console without database logging):**
```bash
python scripts/live_run.py --dry-run
```

**Forced Refresh of Cached Data:**
```bash
python scripts/live_run.py --force-refresh
```

**Daily Signal Generation for Equity Indices:**
```bash
python scripts/index_run.py
```
*Note: Like the currency script, when run on weekends, it automatically rolls back the calculation date to the last Friday. Supports `--dry-run`, `--force-refresh`, and `--date YYYY-MM-DD` parameters.*

**Equity Index Backtest Execution:**
```bash
python scripts/index_backtest.py
```
*Supports `--start YYYY-MM` and `--threshold` (entry threshold, default 0.5) parameters.*

**Running the Telegram Bot:**
To run the bot, you need to configure the token and allowed IDs in the `.env` file:
```env
TELEGRAM_BOT_TOKEN="your_bot_token_here"
ALLOWED_TELEGRAM_USERS="your_telegram_ID_here"
```
Launch the bot:
```bash
python scripts/telegram_bot.py
```
*The bot supports whitelisting, interactive scorecard, sentiment monitoring with timestamped alerts, manual input (FSM) with a COT button and DIY surprises, composite score charts via Matplotlib, step-by-step lot calculation, and CB press release analysis.*

**Running the Position Sizer Calculator:**
```bash
python scripts/position_sizer.py
```
*The script requests the pair, account size, risk in %, current price, and ATR(14) value step-by-step to calculate the exact and operational lot size, taking into account pip value conversion.*

**Running the Local Web Dashboard (Web UI Dashboard):**
Launch the interactive portal with Gemini-based news scanning, CB statement analysis, and G8 scores tracking (runs locally on port 8080, does not require external libraries):
```bash
python scripts/web_dashboard.py
```

**Running the Console Media Dashboard:**
```bash
# Use AI analysis by default (Gemini 3.1 Flash Lite)
python scripts/live_media_dashboard.py

# Use VADER offline analysis (without API requests)
python scripts/live_media_dashboard.py --vader
```

**Console AI Sentiment Testing (CLI):**
```bash
# Evaluate news feed for a specific currency
python scripts/run_gemini_sentiment.py --currency EUR

# Evaluate CB statement text directly
python scripts/run_gemini_sentiment.py -c USD --text "FOMC minutes indicate sticky inflation..."

# Run parser simulation (without consuming API quota)
python scripts/run_gemini_sentiment.py --currency GBP --dry-run
```

**Running Tests:**
```bash
pytest tests/
```

**Updating Manual Macro Indicators (weekly):**
```bash
python scripts/update_alpha.py
python scripts/update_yields.py
```

---

## 🌐 AI Sentiment and External Connectors (API & Fallbacks)

### 1. Transition to the `gemini-3.1-flash-lite` Model
Following the audit of Google AI Studio free API limits, the `gemini-2.5-flash` model is heavily rate-limited by region to **20 requests per day** (triggering `429 Rate Limit Exceeded` errors).
Consequently, all system components have been transitioned to **`gemini-3.1-flash-lite`**, which has a limit of **1500 requests per day**. This ensures stable news parsing without hitting rate limits. The API key is set in the `.env` file under the `GEMINI_API_KEY` variable.

### 2. Fault Tolerance of Web UI and VADER Fallback
The local server in [web_dashboard.py](scripts/web_dashboard.py) and the console [live_media_dashboard.py](scripts/live_media_dashboard.py) are equipped with a **Graceful Degradation** mechanism:
- When Gemini API limits are exhausted (HTTP 429) or in case of network failures, the handler automatically switches to the offline **VADER** library for local sentiment calculation.
- The API returns the sentiment with a `"fallback": "VADER"` flag, and the UI displays a warning while continuing to function stably.
- State preservation is implemented in the Web UI: when switching currency tabs in the sidebar, loaded news, descriptions, and Z-score weights are not reset or re-requested, but are displayed instantly from the browser cache.

### 3. G8 External Data Connectors Audit

Within the macro data integration, an audit of 5 external API connectors was performed:
1. **BIS REER (`bis_data.py`):** Bank for International Settlements SDMX API. Works stably (data lag ~40-70 days). **Important:** when requesting the entire `WS_EER` database, the API times out. Filtering by a list of countries (e.g. `countries=['US']`) is mandatory.
2. **BoC BCPI (`boc_data.py`):** Bank of Canada Valet API.
   - Annual series (`A.BCPI`) have a massive lag of **1.4 years**.
   - To fetch fresh data (~41 days delay), monthly series codes with the `M.` prefix must be used (`M.BCPI`, `M.ENER`, `M.MTLS`, `M.AGRI`).
   - Overnight Rate Target returns data stably (~41 days lag).
3. **RBA (`rba_data.py`):** Reserve Bank of Australia API. Blocked entirely by Akamai CDN on the RBA side for cloud IP addresses. Integration is impossible without residential proxies or a VPN.
4. **SNB (`snb_data.py`):** Swiss National Bank API.
   - The `snbgwdzid` series (Policy Rate, SARON) works stably (daily frequency).
   - Balance sheet and sight deposits tables (`snbvsaktv`, `snbvsakv`, `snbdvsicht`) return a `404 Table not found` error.
   - During the sitemap audit of the SNB site, active working tables were found:
     - `snbmonagg` (Monetary aggregates M1-M3, monthly)
     - `snbimfra` (IMF Reserve assets, monthly)
     - `snbcurrc` (Current account / balance of payments, quarterly)
     - `snbgwdmigirow` / `snbmoba` (Weekly sight deposits / monetary base volume in CHF billions).
5. **OECD Real GDP (`oecd_data.py`):** Requires the use of a 13-dimensional key (e.g. `Q.Y.USA.S1.S1.B1GQ._Z._Z._Z.XDC.L.N.T0102` for the US) and the installation of the `defusedxml` package to parse XML responses. Without an explicit `*` in the filters, the SDMX v2 API returns a 422 error.

## 📊 Calibration and Metrics (dev/razor)

Within the backtest on the OOS period 2023–2026, the system demonstrates the following metrics:
- **Total Hit Rate (63-day):** `69.62%`
- **Total Profit Factor:** `3.95`

### Currency Pair Tier Table (Out-of-Sample Backtest Results 2023–2026)

A detailed report on tiers, win rates, profit factors, and the blacklist is in the [PAIR_TIERS.md](PAIR_TIERS.md) file.

| Tier | Pair | Hit Rate | Profit Factor | Description |
|---|---|:---:|:---:|---|
| **Tier 1** | GBP/JPY | 66.67% | **8.04** | High signal quality |
| (PF ≥ 3.0) | AUD/JPY | 66.67% | **6.76** | confirmed on OOS |
| | USD/JPY | 76.19% | **6.40** | walk-forward test. |
| | EUR/JPY | 66.67% | **4.85** | |
| **Tier 2** | AUD/CHF | 61.11% | **2.90** | Good signal quality, |
| (PF ≥ 1.4) | NZD/JPY | 50.00% | **1.96** | stable mathematical |
| | JPY/CAD | 52.17% | **1.95** | expectation. |
| | GBP/USD | 53.33% | **1.94** | |
| | NZD/USD | 53.85% | **1.54** | |
| **Tier 3** | AUD/USD | 57.14% | **1.23** | Boundary results. |
| (PF ≥ 1.0) | USD/CAD | 54.17% | **1.16** | Reduced lot size allowed. |

*Note: Pairs with CHF (except AUD/CHF), as well as pairs with insufficient sample size (e.g. CAD/NZD, USD/EUR) are completely blocked and blacklisted (see [PAIR_TIERS.md](PAIR_TIERS.md)).*

---

## 🌐 Regime Trading Universes (Regime Universes)

To reduce noise in market stress and sideways phases, the active trading universe is strictly divided by regimes in accordance with [pair_selector.py](execution/pair_selector.py):

1. **Risk-On Universe (Risk Appetite):**
   * `GBP/JPY`, `EUR/JPY`, `USD/JPY`, `USD/GBP`, `USD/AUD`, `JPY/NZD`, `JPY/AUD`, `USD/CAD`, `JPY/CAD`, `CHF/AUD`
2. **Neutral Universe (Normal State):**
   * `EUR/JPY`, `USD/GBP`, `USD/NZD`, `USD/AUD`, `JPY/NZD`, `JPY/AUD`, `CHF/AUD`, `USD/CAD`
3. **Risk-Off Universe (Flight to Safety / Panic):**
   * `GBP/JPY`, `EUR/JPY`, `USD/JPY`, `USD/GBP`, `USD/NZD`, `JPY/NZD`, `JPY/AUD`, `USD/CAD`, `JPY/CAD`

*Signals for pairs not included in the active universe of the current regime are automatically marked as `[REGIME FILTERED]` in `live_run.py` and excluded from trading.*

---

## ⚖️ Risk Management and Position Sizing

For calculating transaction volumes and managing risk, the project uses the interactive calculator [position_sizer.py](scripts/position_sizer.py).

### Position Sizing Formula (Lots)
The position size is calculated based on the daily volatility of the instrument using the formula:
$$Lots = \frac{Risk (\$)}{ATR(14) \times Pip\_Value\_Per\_Lot}$$

where:
- **Risk ($)**: The specified risk per trade in dollars, calculated from the trading account size (e.g., a 0.5% risk on a $5,000 account equals $25).
- **ATR(14)**: Daily Average True Range from TradingView/platform (in absolute terms, e.g., 0.00546).
- **Pip Value Per Lot**: Calculated dynamically based on the quote currency of the pair and the current exchange rate to USD.

Running the calculator:
```bash
python scripts/position_sizer.py
```

---

## 📉 Equity Index Signals

In addition to the G8 FX market, the system supports generating signals for **equity indices** of G8 countries via a separate pipeline [index_run.py](scripts/index_run.py).

### 1. Equity Index Strategy Core
The composite signal for each index is calculated based on 3 factors:
- **OECD CLI Momentum (50%):** economic health leading indicators momentum (MoM change in OECD CLI, normalized via a 3-year Z-window).
- **Yield Curve Slope (30%):** yield curve slope 10Y-2Y (normalized Z-score signaling recession/expansion phases).
- **HY Credit Spread (20%):** monthly momentum of the corporate high-yield credit spread (High Yield OAS), normalized and **inverted** (narrowing spread = positive factor = rising equities).

For instruments of commodity-exporting countries (e.g., ASX200 for Australia), a commodity overlay of 10% is integrated into the scoring (CLI 40% / YC 30% / HY 20% / Commodity 10%), but it is currently disabled (commodity tickers are set to None).

### 2. Traded Instruments
- 🇺🇸 **SP500** (USA, ticker `^GSPC`, base score USD)
- 🇩🇪 **DAX40** (Germany, ticker `^GDAXI`, base score EUR)
- 🇯🇵 **JPN225** (Japan, ticker `^N225`, base score JPY)
- 🇬🇧 **UK100** (UK, ticker `^FTSE`, base score GBP)
- 🇦🇺 **ASXAUD** (Australia, ticker `^AXJO`, base score AUD)

### 3. Filters and Rules
- **Entry:** A LONG signal is generated when the composite score $\ge 0.5\sigma$. A SHORT signal is generated when the composite score $\le -0.5\sigma$ (in the absence of a VIX veto).
- **VIX Short Veto:** Opening new short positions (SHORT) is automatically blocked if the fear index VIX $\ge 22.0$ at the time of pipeline execution.
- **Exit:** Position closing in daily mode (`index_run.py`) occurs upon a **sign reversal** of the composite or upon reaching a time limit of **63 trading days** (signal decay is used only in the monthly backtester).
