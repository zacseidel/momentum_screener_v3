# Momentum Screener: Architecture Overview

This project is a modular, database-backed momentum screener for U.S. equities, focused on weekly analysis and decision support. It draws on SP500/SP400 index membership, personal watchlists, portfolio holdings, and past report candidates to construct a pipeline that filters stocks, computes technical features, applies machine learning classifications, and generates HTML reports for investment consideration.

## 📐 System Design

### 🔁 Data Flow Pipeline

   SSGA & Polygon APIs
          │
          ▼
     Database Update
          │
          ▼
    Build Scan Set
          │
          ▼
 Filter to Focus Group
          │
          ▼
 Compute Features & States
          │
          ▼
 Generate Weekly Report
          │
   ┌──────┴──────┐
   ▼             ▼
GitHub Pages SendGrid Email


### 🔍 Scan Set vs Focus Group

- **Scan Set**: All stocks of interest  
  Includes:
  - SP500/SP400 constituents
  - Personal portfolio tickers
  - Watchlist tickers
  - Previous report members

- **Focus Group**: Filtered stocks for full analysis  
  - Index stocks must meet momentum or market cap thresholds
  - Watchlist/portfolio/previous-report stocks are auto-included

---

## 🧱 Module Structure
```text
momentum-screener/
│
├── data/               # External data acquisition
│   ├── prices.py
│   ├── index_membership.py
│   ├── allocations.py
│   ├── watchlist.py
│   ├── news.py
│   └── init_db.py
│
├── sets/               # Set construction and filtering
│   ├── scan_set.py
│   ├── focus_group.py
│   └── report_members.py
│
├── features/           # Feature engineering and labeling
│   ├── features.py
│   ├── states.py
│   └── forward_returns.py
│
├── analysis/           # Ranking and summary logic
│   ├── momentum.py
│   ├── tracker.py
│   └── summary_stats.py
│
├── report/             # Output generation
│   ├── report.py
│   ├── charts.py
│   ├── html_builder.py
│   ├── emailer.py
│   └── publisher.py
│
├── ml/                 # Model training and prediction
│   ├── train_model.py
│   ├── predict_model.py
│   └── model_utils.py
│
├── log/                # Manual decision tracking
│   ├── decision_log.py
│   └── review_log.py
│
├── scripts/            # Orchestration runners
│   ├── generate_report.py
│   ├── retrain_model.py
│   └── backfill_data.py
│
├── config.py           # Global paths, constants, keys
├── README.md
└── requirements.txt
---
```
## 🗃️ SQLite Schema (see ERD below)

Key tables:

- index_membership, watchlist
- price_history, momentum_snapshots
- features, ml_states, forward_returns
- company_metadata, news_articles
- decision_log

## 🧠 Machine Learning

A quarterly-trained classifier assigns each focus group stock a "state" based on technical features. States correspond to return behaviors. These states are tracked over time and used to contextualize weekly signals.

## 📤 Output & Publication

The weekly report includes:

Focus group table
Summary chart comparing relative returns
Individual cards: company metadata, chart, momentum/ML details, recent news
Publication via GitHub Pages + optional email delivery


## 🗂️ Directory Scaffold (Reproduced as Tree View)

```text
momentum-screener/
├── data/
│   ├── prices.py
│   ├── index_membership.py
│   ├── allocations.py
│   ├── watchlist.py
│   ├── news.py
│   └── init_db.py
├── sets/
│   ├── scan_set.py
│   ├── focus_group.py
│   └── report_members.py
├── features/
│   ├── features.py
│   ├── states.py
│   └── forward_returns.py
├── analysis/
│   ├── momentum.py
│   ├── tracker.py
│   └── summary_stats.py
├── report/
│   ├── report.py
│   ├── charts.py
│   ├── html_builder.py
│   ├── emailer.py
│   └── publisher.py
├── ml/
│   ├── train_model.py
│   ├── predict_model.py
│   └── model_utils.py
├── log/
│   ├── decision_log.py
│   └── review_log.py
├── scripts/
│   ├── generate_report.py
│   ├── retrain_model.py
│   └── backfill_data.py
├── config.py
├── README.md
└── requirements.txt
```

## Entity-Relationship Diagram (ERD)
```mermaid

index_membership {
  TEXT ticker
  TEXT index_name
  TEXT sector
  TEXT subindustry
  REAL market_cap
  REAL weight
  TEXT last_updated
}

price_history {
  TEXT ticker
  TEXT date
  REAL open
  REAL high
  REAL low
  REAL close
  INTEGER volume
}

momentum_snapshots {
  TEXT date
  TEXT ticker
  REAL return_12m
  INTEGER return_rank
  TEXT source_universe
}

features {
  TEXT ticker
  TEXT date
  REAL pct_from_52w_high
  REAL pct_from_52w_low
  REAL slope_200d
  BOOLEAN above_10d
  BOOLEAN above_150d
  BOOLEAN above_200d
  INTEGER high_volume_days
  REAL trend_strength
}

ml_states {
  TEXT ticker
  TEXT date
  INTEGER state_id
  TEXT state_label
}

forward_returns {
  TEXT ticker
  TEXT date
  REAL return_1w
  REAL excess_return_1w
}

company_metadata {
  TEXT ticker
  TEXT name
  TEXT description
  TEXT sector
  TEXT subindustry
}

news_articles {
  TEXT ticker
  TEXT date
  TEXT headline
  TEXT url
  TEXT source
  REAL sentiment_score
}

watchlist {
  TEXT ticker
  TEXT tag
  TEXT added_on
}

decision_log {
  TEXT ticker
  TEXT date
  TEXT action
  TEXT notes
}
```

## Module Map & dependencies

### data/

#### Prices.py
def update_grouped_prices(tickers: list[str], start_date: str, end_date: str):
    """Fetch and update grouped price data for given tickers over the date range."""

def update_full_history(ticker: str):
    """Download and store full 2-year price history for a single ticker."""

def update_incremental_price_data(tickers: list[str], days: int = 5):
    """Download incremental daily prices for past N days using grouped API."""

#### index_membership.py
def update_index_membership():
    """Download and store latest SP500 and SP400 constituent lists and weights."""

def get_index_tickers(index_name: str) -> list[str]:
    """Return current list of tickers in specified index (e.g., 'SP500')."""

#### allocations.py
def fetch_etf_weights(etf: str) -> pd.DataFrame:
    """Fetch and return SPY/MDY weightings from SSGA Excel source."""

#### watchlist.py
def add_to_watchlist(ticker: str, tag: str = ""):
    """Add ticker to watchlist with optional tag."""

def remove_from_watchlist(ticker: str):
    """Remove ticker from watchlist."""

def get_watchlist() -> list[str]:
    """Return all tickers currently in watchlist."""

#### news.py
def update_news(tickers: list[str]):
    """Fetch and store recent headlines and metadata for each ticker."""

def get_recent_articles(ticker: str) -> pd.DataFrame:
    """Return recent news articles for a given ticker."""

#### init_db.py
def initialize_database():
    """Create all required tables if they do not exist."""

### sets/
### scan_set.py
def build_scan_set(as_of_date: str) -> set[str]:
    """Construct scan set from index tickers, watchlist, portfolio, prior reports."""

def cache_scan_set(tickers: set[str], as_of_date: str):
    """Store list of scan set tickers for this report date."""

#### focus_group.py
def filter_focus_group(scan_set: set[str], as_of_date: str) -> set[str]:
    """Apply inclusion rules to select focus group from scan set."""

def cache_focus_group(tickers: set[str], as_of_date: str):
    """Store list of focus group tickers for this report date."""

#### report_members.py
def filter_focus_group(scan_set: set[str], as_of_date: str) -> set[str]:
    """Apply inclusion rules to select focus group from scan set."""

def cache_focus_group(tickers: set[str], as_of_date: str):
    """Store list of focus group tickers for this report date."""

### features/
#### features.py
def compute_features(ticker: str, as_of_date: str):
    """Compute and store technical indicators for a given ticker."""

def update_features(tickers: list[str], as_of_date: str):
    """Compute features in batch for multiple tickers."""

#### states.py
def update_states(tickers: list[str], as_of_date: str):
    """Load model and assign ML state to each ticker."""

def predict_states(feature_df: pd.DataFrame) -> pd.DataFrame:
    """Return predicted states from features using stored model."""

#### forward_returns.py
def compute_forward_returns(ticker: str, as_of_date: str):
    """Calculate and store 1-week forward returns for ticker."""

def update_forward_returns(tickers: list[str], as_of_date: str):
    """Batch forward return computation."""


### analysis/
#### momentum.py
def compute_12mo_return(ticker: str, as_of_date: str) -> float:
    """Calculate 12-month trailing return."""

def rank_momentum(tickers: list[str], as_of_date: str) -> pd.DataFrame:
    """Rank all tickers by 12-month return."""

#### tracker.py
def get_missing_history_tickers(as_of_date: str) -> list[str]:
    """Identify tickers missing full history for current analysis."""

def get_new_focus_group_entries(as_of_date: str) -> list[str]:
    """Identify newly promoted focus group members this week."""

#### summary_stats.py
def summarize_sector_performance(as_of_date: str) -> pd.DataFrame:
    """Return summary stats (avg return, state counts) by sector."""

def state_transition_summary(as_of_date: str) -> pd.DataFrame:
    """Report changes in ML state week-over-week."""



### report/
#### report.py
def gather_company_info(ticker: str) -> dict:
    """Return company metadata for the report card."""

def get_card_data(ticker: str, as_of_date: str) -> dict:
    """Return full data for rendering one ticker's report card."""

#### charts.py
def generate_relative_return_chart(tickers: list[str], benchmark: str, as_of_date: str):
    """Create summary chart of all tickers vs benchmark."""

def generate_stock_chart(ticker: str, as_of_date: str):
    """Create 12-month price chart for individual stock."""

#### html_builder.py
def render_report(focus_group_data: list[dict], as_of_date: str) -> str:
    """Render HTML content using templates and ticker data."""

def write_report_to_disk(html: str, output_path: str):
    """Save rendered HTML file to specified path."""

#### emailer.py
def send_email(subject: str, html_content: str):
    """Send the rendered report using SendGrid."""

#### publisher.py
def push_to_github_pages(report_path: str):
    """Deploy HTML report to GitHub Pages site."""

def publish_to_substack(subject: str, html: str):  # optional future
    """Optional: Send to Substack audience."""

def tweet_summary(summary_text: str):  # optional future
    """Optional: Tweet highlight from report."""


### ml/
#### train_model.py
def train_classifier(X: pd.DataFrame, y: pd.Series):
    """Train model from features and target labels."""

def save_model(model, path: str):
    """Save trained model to disk."""

#### predict_model.py
def load_model(path: str):
    """Load stored classifier from disk."""

def predict_states(X: pd.DataFrame) -> pd.DataFrame:
    """Predict state for input features."""

#### model_utils.py
def evaluate_model(model, X_test, y_test):
    """Print AUC, accuracy, etc."""

def plot_confusion_matrix(model, X_test, y_test):
    """Generate and display confusion matrix."""

def model_summary(model):
    """Print model parameters, feature importances, etc."""

### log/ 
#### decision_log.py
def log_decision(ticker: str, date: str, action: str, notes: str):
    """Record user decision in markdown-compatible format."""

def get_decisions(date_range: tuple[str, str]) -> pd.DataFrame:
    """Return decision log entries in date range."""

#### review_log.py
def compare_past_decisions_to_outcome():
    """Analyze decision accuracy based on forward return."""

def summarize_performance_by_state():
    """Aggregate return statistics grouped by ML state."""


### scripts
#### generate_report.py
def run_pipeline(as_of_date: str):
    """Main weekly orchestration function."""

#### retrain_model.py
def retrain_and_save():
    """Retrain the ML model on most recent features and outcomes."""
    
##### backfill_data.py
def backfill_historical_prices(start_year: int):
    """Pull old price history for all tickers as needed."""
