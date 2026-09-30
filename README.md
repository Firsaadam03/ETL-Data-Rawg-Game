# ETL-Data-Rawg-Game

# 🎮 Game Discovery Analytics — RAWG ETL Pipeline

An end-to-end data engineering project that extracts game catalog data from the public [RAWG Video Games Database API](https://rawg.io/apidocs), transforms it with Python and pandas, and loads it into **Google BigQuery** as a lightweight star schema, ready to be queried from a BI tool such as Looker Studio.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![pandas](https://img.shields.io/badge/pandas-data%20transformation-150458)
![BigQuery](https://img.shields.io/badge/Google-BigQuery-4285F4)

---

## 🔎 Overview

This project implements a full **Extract → Transform → Load → Analytics** workflow for a game discovery use case:

1. **Extract** game, genre, and platform data from the RAWG REST API.
2. **Transform** the nested JSON into clean, analysis-ready tables and add derived features.
3. **Load** the tables into a BigQuery dataset (`RAWG`).
4. **Analyze** the warehouse with SQL designed to plug directly into Looker Studio.

Everything runs from a single Jupyter notebook, and no secrets are hard-coded: the API key, project ID, and credential path are read from a local `.env` file.

## ❓ Business Questions

The pipeline is designed to answer product/marketing-style questions that a game discovery platform would care about:

- Which genres are released most often, and which rate the highest?
- Which platforms dominate the market by title count?
- Is there a correlation between critic scores (Metacritic) and player ratings?
- Which developers consistently ship highly rated games?
- How have release trends shifted over the years?

## 🏗 Architecture

```
RAWG.io API  (free API key)
      │
      │  Extract
      │  • /games      → core catalog data (rating, metacritic, playtime, ...)
      │  • /genres     → genre reference data
      │  • /platforms  → platform reference data
      ▼
Python ETL  (requests + pandas)
      │
      │  Transform: clean, flatten nested JSON, feature engineering, quality checks
      ▼
Google BigQuery  — dataset: RAWG
      ├── fact_games       (fact table, one row per game)
      ├── genres_raw       (game ↔ genre junction table)
      ├── platforms_raw    (game ↔ platform junction table)
      ├── genre            (genre dimension)
      └── platform         (platform dimension)
                │
                ▼
        Looker Studio dashboard (Custom SQL)
```

## 🧰 Tech Stack

| Layer | Tools |
|---|---|
| Extraction | Python, `requests`, RAWG REST API |
| Transformation | `pandas`, `numpy` |
| Warehouse | Google BigQuery, `pandas-gbq` |
| Config & credentials | `python-dotenv`, Google service account |
| Consumption layer | Looker Studio (Custom SQL) |
| Environment | Jupyter Notebook |

## 🗄 Data Model

The warehouse follows a lightweight star schema. One game can belong to many genres and many platforms, so those relationships are stored as junction tables instead of being flattened away.

| Table | Type | Description |
|---|---|---|
| `fact_games` | Fact | One row per game: rating, Metacritic score, playtime, review counts, ESRB rating, release date, plus derived features |
| `genres_raw` | Junction | Links `game_id` to `genre_id` (many-to-many) |
| `platforms_raw` | Junction | Links `game_id` to `platform_id` (many-to-many) |
| `genre` | Dimension | Genre reference data (`id`, `name`, `slug`, `games_count`, ...) |
| `platform` | Dimension | Platform reference data (`id`, `name`, `slug`, `games_count`, `year_start`, ...) |

### Derived features in `fact_games`

| Feature | Description |
|---|---|
| `release_year` | Year extracted from `released` (null for unreleased titles) |
| `release_decade` | Decade bucket, e.g. `2010s` |
| `rating_category` | Player rating bucket: Exceptional (≥ 4.5), Recommended (≥ 3.5), Meh (≥ 2.5), Skip |
| `metacritic_category` | Critic score bucket: Must Play (≥ 90), Great (≥ 75), Good (≥ 60), Mixed |
| `has_metacritic` | Boolean flag for whether a Metacritic score exists |
| `popularity_score` | Blend of player rating and critic score: `0.6 × (rating × 20) + 0.4 × metacritic`; falls back to `rating × 20` when no Metacritic score exists |

## ⚙️ Pipeline Details

### 1. Extract

| RAWG parameter | Detail |
|---|---|
| Base URL | `https://api.rawg.io/api/` |
| Auth | API key as a query parameter (`?key=YOUR_KEY`) |
| Pagination | Cursor-style, via the `next` field in each response |

- `get_data()` is a thin wrapper around `requests.get` that injects the API key and raises an error on non-200 responses.
- `fetch_pages()` pulls paginated results from `/games`. The volume is controlled by the `page_size` parameter and the `max_page` argument (the notebook uses `page_size=40` and `max_page=5`).
- Nested `genres` and `platforms` fields are kept intact at this stage and flattened during transformation.
- Unreleased titles (`tba=True` or `released=None`) are intentionally kept, since "not yet released" is a useful signal for trend analysis.
- `/genres` and `/platforms` are small reference endpoints fetched in a single call each.

### 2. Transform

1. Drop fields with no analytical value (`slug`, `stores`, `tags`, `clip`, color fields, etc.).
2. Cast `released` and `updated` to `datetime`.
3. Explode `genres` and `platforms` into long-format junction tables (`genres_raw`, `platforms_raw`).
4. Flatten `esrb_rating.name` into a plain string column.
5. Engineer the derived features listed above.
6. Run data-quality checks before loading:
   - deduplicate by game `id`
   - `rating` must be between 0 and 5
   - `ratings_count` and `playtime` must be non-negative

Intermediate checkpoints are exported as CSV (`raw_game.csv`, `genres_raw.csv`, `platforms_raw.csv`) so the cleaned tables can be inspected without re-calling the API.

### 3. Load

Tables are written to the BigQuery dataset `RAWG` using `pandas_gbq.to_gbq()` with a service-account credential. The current load strategy is a **full refresh** (`if_exists="replace"`).

## 📁 Project Structure

```
ETL-Data-Rawg-Game/
├── game_database_etl_pipeline.ipynb   # Full pipeline: extract → transform → load → SQL
├── raw_game.csv                       # Checkpoint: cleaned games table
├── genres_raw.csv                     # Checkpoint: game ↔ genre junction table
├── platforms_raw.csv                  # Checkpoint: game ↔ platform junction table
├── .env.example                       # Template for required environment variables
├── .gitignore                         # Keeps secrets out of version control
└── README.md
```

> **Not committed (kept local):** `.env` and the Google service account JSON key.

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- A free RAWG API key: <https://rawg.io/apidocs>
- A Google Cloud project with the BigQuery API enabled
- A Google Cloud **service account** with permission to create and write BigQuery tables (for example, `BigQuery Data Editor` and `BigQuery Job User`), and its JSON key file

### 1. Clone the repository

```bash
git clone https://github.com/Firsaadam03/ETL-Data-Rawg-Game.git
cd ETL-Data-Rawg-Game
```

### 2. Install dependencies

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install pandas numpy requests pandas-gbq google-auth python-dotenv jupyter
```

### 3. Configure environment variables

Create a `.env` file in the project root (see `.env.example`):

```env
API_KEY=your_rawg_api_key
PROJECT_ID=your_gcp_project_id
CREDENTIAL_PATH=path/to/your-service-account-key.json
```

Make sure secrets are never committed:

```gitignore
.env
*.json
```

### 4. Run the pipeline

```bash
jupyter notebook game_database_etl_pipeline.ipynb
```

Run the cells top to bottom. When finished, the five tables will be available in the `RAWG` dataset in BigQuery.

## 📊 Analytics Layer

The last section of the notebook contains Custom SQL queries designed for Looker Studio:

| Query | Chart type | Purpose |
|---|---|---|
| KPI Scorecard Pack | Scorecards | Total games, average rating, average Metacritic, % of games with a critic score |
| Release trend by genre | Time series | Number of releases per year, broken down by genre |
| Rating vs. Metacritic | Scatter plot | Player rating vs. critic score, bubble size = playtime |
| Rating category × genre | Heatmap | Distribution of rating categories across genres |

See section 5 of the notebook for the full SQL.

## 🔭 Possible Extensions

- Incremental (delta) loads instead of full refreshes
- Orchestration with Airflow / Cloud Composer for scheduled runs
- A `dim_date` table and slowly changing dimension handling for genre/platform reference data
- Unit tests for the transform functions (`rating_category`, `metacritic_category`, `popularity_score`)

## 👤 Author

**FIrsa Adam** — [GitHub](https://github.com/Firsaadam03)

---

*Data provided by [RAWG](https://rawg.io/). This project is for educational and portfolio purposes.*
