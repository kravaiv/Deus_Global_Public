# Statistics and Volume of the Deus Global v1 Project

**Document Creation Date:** 2026-06-25

## 1. General Project Volume Metrics
* **Total Number of Files:** 3,828
* **Total Lines of Code (Lines of Code):** 1,414,584
* **Total Source Code Size:** 60.89 MB (excluding the virtual environment `.venv`, `.git` directory, `data/cache` cache, and historical logs)

## 2. Programming Languages and File Types Used
* **C++ (`.cpp` / `.h`):** 1,983 files, 430,597 lines. Used in the research block (FinceptTerminal QT modules) for high-performance calculations.
* **TypeScript (`.ts`):** 11 files, 453,808 lines. Responsible for analytical dashboards and frontend interfaces.
* **Python (`.py`):** 1,151 files, 415,004 lines. The main language of the core: model calculations, Z-score, data ingestion, Telegram bot, web dashboard.
* **Markdown (`.md`):** 376 files, 62,309 lines. Documentation, macro-indicator specifications, and architectural design documents.
* **JSON (`.json`):** 84 files, 13,674 lines. Configuration files and weekly manual entry snapshots.

## 3. Core Program Modules of the Kernel (Python)
* **`scripts/telegram_bot.py` (2,470 lines, 108.58 KB):** An interactive bot for displaying signals, entering data, and administration.
* **`modules/scoring/currency_scorer.py` (1,756 lines, 98.41 KB):** Calculation of the Composite Score, adaptive Z-score, inflation de-trending, and toxic yield filtering.
* **`scripts/web_dashboard.py` (1,224 lines, 48.35 KB):** Web analytical dashboard.
* **`modules/context/regime_filter.py` (996 lines, 46.97 KB):** Determination of the market regime (Risk-On/Off) based on VIX and credit spreads.
* **`scripts/live_run.py` (924 lines, 38.58 KB):** Daily pipeline for data collection and signal generation.
* **`modules/matrix/matrix_engine.py` (765 lines, 35.41 KB):** Construction of the G10 currency pair trading matrix (28 pairs).
* **`modules/db.py` (719 lines, 29.91 KB):** SQLite database management (`deus.db`), storing execution sessions and signals history.

## 4. Architectural Highlights of the Project
* **Business Cycle Phase Smoothing (Cycle Blender):** Dynamic interpolation of weights based on PMI.
* **Adaptive Volatility Z-score:** Calculations adapt dynamically to changes in the VIX.
* **Built-in Toxic Yield Protector (Toxic Yield Filter):** Protection against rising interest rates on the background of sovereign risk during market panics.
* **Multi-level Macro Indicator Overlay (Macro Overlay):** Integration of household debt vulnerability and news sentiment metrics.
