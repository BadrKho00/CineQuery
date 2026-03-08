# CineQuery — MovieLens × BigQuery

> **Live URL:** `https://YOUR-CLOUD-RUN-URL` ← update after deployment

A Streamlit movie search app backed by Google BigQuery and enriched with TMDB metadata.

---

## Project Structure

```
movie-app/
├── app.py              # Streamlit entry point
├── config.py           # All configuration & env vars
├── requirements.txt
├── Dockerfile
├── db/
│   ├── client.py       # BigQuery client (cached)
│   └── queries.py      # SQL builders & executors
├── api/
│   └── tmdb.py         # TMDB API helper
└── ui/
    ├── sidebar.py      # Filter sidebar component
    ├── results.py      # Movie card grid
    ├── movie_detail.py # Full detail view
    └── styles.css      # Custom dark theme
```

---

## Local Setup

### 1. Clone & install
```bash
git clone <your-repo>
cd movie-app
pip install -r requirements.txt
```

### 2. Authenticate with Google Cloud
```bash
gcloud auth application-default login
```

### 3. Set environment variables
```bash
export GCP_PROJECT_ID="your-gcp-project-id"
export BQ_DATASET="movielens"
export TMDB_API_KEY="your-tmdb-api-key"   # from https://www.themoviedb.org/settings/api
```

### 4. Run locally
```bash
streamlit run app.py
```

---

## BigQuery Setup

### 1. Create a dataset
```bash
bq mk --dataset your-gcp-project-id:movielens
```

### 2. Upload movies table
```bash
bq load \
  --autodetect \
  --source_format=CSV \
  movielens.movies \
  gs://your-bucket/movies.csv
```
*(Or use the BigQuery console — upload CSV, set table name `movies`)*

**Schema:** `movieId:INTEGER, title:STRING, genres:STRING, tmdbId:INTEGER, language:STRING, release_year:INTEGER, country:STRING`

### 3. Upload ratings table
Same process, table name `ratings`.

**Schema:** `userId:INTEGER, movieId:INTEGER, rating:FLOAT, timestamp:INTEGER`

---

## Docker

### Build locally
```bash
docker build -t cinequery .
docker run -p 8080:8080 \
  -e GCP_PROJECT_ID="your-project" \
  -e BQ_DATASET="movielens" \
  -e TMDB_API_KEY="your-key" \
  -v ~/.config/gcloud:/root/.config/gcloud \
  cinequery
```
Visit http://localhost:8080

---

## Deploy to Google Cloud Run

```bash
# 1. Set your project
gcloud config set project YOUR_PROJECT_ID

# 2. Enable APIs (one-time)
gcloud services enable run.googleapis.com cloudbuild.googleapis.com bigquery.googleapis.com

# 3. Build & push image
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/cinequery

# 4. Deploy
gcloud run deploy cinequery \
  --image gcr.io/YOUR_PROJECT_ID/cinequery \
  --platform managed \
  --region europe-west1 \
  --allow-unauthenticated \
  --set-env-vars GCP_PROJECT_ID=YOUR_PROJECT_ID,BQ_DATASET=movielens,TMDB_API_KEY=YOUR_TMDB_KEY
```

The deploy command prints a **Service URL** — paste it in your README above.

---

## Features

| Feature | Implementation |
|---|---|
| Title autocomplete | `LIKE LOWER(@title_pattern)` |
| Language filter | `WHERE language = @language` |
| Genre filter | `WHERE genres LIKE @genre_pattern` |
| Avg rating filter | `JOIN ratings … GROUP BY … HAVING AVG(rating) >= @min_rating` |
| Year range filter | `WHERE release_year BETWEEN @min_year AND @max_year` |
| Movie details | TMDB API — poster, cast, overview, runtime |
| SQL logging | All queries printed to terminal |
