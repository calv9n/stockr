# stockR - Stock Recommendation System

A data engineering project that generates daily BUY / HOLD / SELL recommendations for a watchlist of stocks and commodities, driven by two independent signals: **technical indicators** (price-based) and **news sentiment** (NLP-scored headlines). Built as a learning project to practise medallion architecture, dbt modelling, and Python-based data enrichment.

> **Disclaimer:** This project is for educational purposes only. Nothing produced by this system constitutes financial advice.

---

## How it works

Two signals are computed independently and combined into a single composite score per ticker per day:

- **Technical signal** — moving average crossovers, RSI, MACD, and price momentum, normalised to a −1/+1 scale
- **Sentiment signal** — rolling 7-day VADER sentiment scores from news headlines, normalised to a −1/+1 scale

```
composite_score = (tech_score × 0.6) + (sentiment_score × 0.4)

score > 0.2  → BUY
score < -0.2 → SELL
else         → HOLD
```

Both signal weights and recommendation thresholds are configurable via dbt variables in `dbt_project.yml`.

---

## Architecture

```
Data sources
  └── yfinance / Alpha Vantage (prices)
  └── NewsAPI / Alpaca News API (headlines)
  └── watchlist.csv (dbt seed)
        │
        ▼
Bronze — raw ingestion (Delta Lake, append-only)
  └── raw_stock_prices
  └── raw_news_headlines
        │
        ▼
Python enrichment
  └── score_sentiment.py (VADER) → raw_news_sentiment
        │
        ▼
Silver — dbt staging & intermediate
  └── stg_stock_prices, stg_news_sentiment, stg_watchlist
  └── int_price_indicators   (MA, RSI, MACD, momentum)
  └── int_sentiment_windows  (rolling 7d aggregations)
  └── int_technical_score    (normalised tech signal)
  └── int_sentiment_score    (normalised sentiment signal)
        │
        ▼
Gold — dbt marts
  └── mart_signal_detail      (all indicators, one row per ticker per day)
  └── mart_recommendations    (composite score + BUY/HOLD/SELL, latest day)
```

---

## Tech stack

| Layer | Technology |
|---|---|
| Data warehouse | Databricks Community Edition + Delta Lake |
| Transformation | dbt Core (`dbt-databricks` adapter) |
| Ingestion | Python (`yfinance`, `requests`, `pandas`) |
| Sentiment NLP | VADER (`vaderSentiment`) |
| Orchestration | Manual run order / local cron |
| Version control | Git + GitHub |

---

## Watchlist

| Ticker | Type | Sector |
|---|---|---|
| AAPL, MSFT, NVDA, GOOGL | Stock | Tech |
| JPM, GS, BAC | Stock | Finance |
| UAL, DAL | Stock | Aviation |
| GC=F | Commodity | Gold |
| CL=F | Commodity | Oil |
| BTC-USD | Crypto | Speculative |

To modify the watchlist, edit `seeds/watchlist.csv` and run `dbt seed`.

---

## Project structure

```
stock-recommendation-system/
├── ingestion/
│   ├── ingest_prices.py
│   ├── ingest_news.py
│   └── score_sentiment.py
├── dbt_project/
│   ├── dbt_project.yml
│   ├── seeds/
│   │   └── watchlist.csv
│   └── models/
│       ├── staging/
│       │   ├── stg_stock_prices.sql
│       │   ├── stg_news_sentiment.sql
│       │   └── stg_watchlist.sql
│       ├── intermediate/
│       │   ├── int_price_indicators.sql
│       │   ├── int_sentiment_windows.sql
│       │   ├── int_technical_score.sql
│       │   └── int_sentiment_score.sql
│       └── marts/
│           ├── mart_signal_detail.sql
│           └── mart_recommendations.sql
├── .env.example
├── requirements.txt
└── README.md
```

---

## Getting started

**1. Clone the repo and set up the Python environment**

```bash
git clone https://github.com/your-username/stock-recommendation-system.git
cd stock-recommendation-system
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

**2. Configure environment variables**

```bash
cp .env.example .env
# Add your API keys and Databricks connection details
```

Required variables:

```
ALPHA_VANTAGE_API_KEY=
NEWS_API_KEY=
DATABRICKS_HOST=
DATABRICKS_TOKEN=
DATABRICKS_HTTP_PATH=
```

**3. Connect dbt to Databricks**

Add to `~/.dbt/profiles.yml`:

```yaml
stock_recommendations:
  target: dev
  outputs:
    dev:
      type: databricks
      host: "{{ env_var('DATABRICKS_HOST') }}"
      http_path: "{{ env_var('DATABRICKS_HTTP_PATH') }}"
      token: "{{ env_var('DATABRICKS_TOKEN') }}"
      catalog: hive_metastore
      schema: silver
      threads: 4
```

Verify the connection:

```bash
cd dbt_project && dbt debug
```

**4. Run the pipeline**

```bash
# Ingest raw data
python ingestion/ingest_prices.py
python ingestion/ingest_news.py

# Score sentiment
python ingestion/score_sentiment.py

# Run dbt
cd dbt_project
dbt seed
dbt run
dbt test
```

**5. Query the output**

```sql
SELECT ticker, composite_score, recommendation, tech_score, sentiment_score
FROM gold.mart_recommendations
ORDER BY composite_score DESC;
```

---

## Key dbt models

| Model | Layer | Description |
|---|---|---|
| `stg_stock_prices` | Staging | Cleaned OHLCV data, deduplicated |
| `stg_news_sentiment` | Staging | Headlines with VADER scores, null-filtered |
| `int_price_indicators` | Intermediate | MA (20d, 50d), RSI, MACD, momentum |
| `int_sentiment_windows` | Intermediate | Rolling 7d sentiment aggregations per ticker |
| `int_technical_score` | Intermediate | Normalised technical signal (−1 to +1) |
| `int_sentiment_score` | Intermediate | Normalised sentiment signal (−1 to +1) |
| `mart_signal_detail` | Gold | All indicators per ticker per day — useful for debugging |
| `mart_recommendations` | Gold | Final output: composite score + BUY/HOLD/SELL, latest day |

---

## Configuration

Signal weights and thresholds are set as dbt variables in `dbt_project.yml`:

```yaml
vars:
  tech_weight: 0.6
  sentiment_weight: 0.4
  buy_threshold: 0.2
  sell_threshold: -0.2
```

Adjust these to experiment with different weighting strategies without touching model SQL.

---

## Data sources

- **Prices** — [yfinance](https://github.com/ranaroussi/yfinance) (no key) or [Alpha Vantage](https://www.alphavantage.co/) (free tier, 25 req/day)
- **News** — [NewsAPI](https://newsapi.org/) or [Alpaca News API](https://alpaca.markets/docs/api-references/market-data-api/news/) (free tiers)
- **Sentiment** — [VADER](https://github.com/cjhutto/vaderSentiment) (local, no API needed)

---

## Notes

- Built and tested on Databricks Community Edition — clusters auto-terminate after 2 hours of inactivity
- VADER is used for sentiment scoring by default. FinBERT (`ProsusAI/finbert`) can be swapped in `score_sentiment.py` for improved accuracy on financial language
- Never commit `.env` — use `.env.example` as the safe template
- Run `dbt docs generate && dbt docs serve` to explore the model lineage graph locally
