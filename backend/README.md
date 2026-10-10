# CreatorLens Backend

FastAPI service behind CreatorLens. It finds influencers from a plain-English prompt, enriches and analyses their profiles, generates AI insights, and produces downloadable PDF reports.

Interactive API docs are served at `/docs` once the server is running.

## What it does

- **Dynamic search**: turns a natural-language prompt into a search, asking a few optional follow-up questions to fill in missing details (pasting an Instagram profile link or username skips them). Influencers are discovered with SerpAPI and profile data is scraped with BrightData and Selenium.
- **Analysis**: engagement and audience metrics, post-level analysis, and comparison against industry standards and peer creators.
- **AI insights**: summaries and recommendations generated with OpenAI and Google Gemini.
- **PDF reports**: branded reports built with ReportLab, with Matplotlib charts.
- **Agent assistant**: a chat endpoint the frontend uses for its assistant.
- **Metrics**: per-search-session performance metrics and system vitals.
- **Background jobs**: Celery workers (Redis broker) refresh stale influencer data.

## Tech stack

| Area | Tools |
| --- | --- |
| Framework | FastAPI, Uvicorn, Pydantic v2 |
| Database | SQLAlchemy (async). SQLite for development, PostgreSQL for production |
| Queue and cache | Celery, Redis |
| AI | OpenAI, Google Gemini |
| Data collection | SerpAPI, BrightData, Selenium (Chrome) |
| Reporting | ReportLab, Matplotlib |

## Requirements

- Python 3.11 or newer (the Docker image uses 3.11)
- Google Chrome or Chromium, for the Selenium scraper
- Redis, for the Celery worker. The API itself starts without it and the Redis cache fails safe if it is unavailable
- API keys: OpenAI, Google Gemini, SerpAPI and BrightData (see [Configuration](#configuration))

## Quick start

```bash
cd backend

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env             # then fill in your keys, see Configuration

uvicorn app.main:app --reload --port 8000
```

- API: http://localhost:8000
- Docs: http://localhost:8000/docs
- Health check: http://localhost:8000/api/health

Tables are created automatically on startup. To create them manually, run `python -m scripts.init_db`.

### Background worker

Run this in a second terminal (Redis must be running):

```bash
celery -A app.config.celery_app worker --loglevel=info
```

### Docker

```bash
cp .env.example .env             # fill in your keys first
docker compose up --build
```

The container serves the API on port 8000 and installs Chrome for scraping. `storage/` is mounted from the host so generated reports and results persist. `docker-compose.yml` also mounts `./data` and the two `influencer_master_data` files, so create them or remove those mounts if you don't use them. The compose file does not start Redis.

## Configuration

Settings are read from `.env` (see `app/config/settings.py`). The app refuses to start unless the required variables below are set.

**Required**

| Variable | Purpose |
| --- | --- |
| `HARD_CODED_USERNAME`, `HARD_CODED_PASSWORD` | Credentials for the single-user login (`POST /api/auth/login`) |
| `OPENAI_API_KEY` | OpenAI access |
| `GEMINI_API_KEY` | Google Gemini access |
| `SERP_API_KEY` | SerpAPI, used for influencer discovery |
| `BRIGHTDATA_API_KEY`, `BRIGHTDATA_DATASET_ID` | BrightData profile scraping |
| `SPREADD_URL` | Spreadd.io URL, used for fake-follower checks |
| `INFLUENCER_DATABASE` | Default influencer dataset CSV filename |
| `ENDPOINT_URL` | Webhook URL that search results are posted to. Set it to an empty value (or a non-HTTP value) to skip the callback |

`.env.example` does not yet list `HARD_CODED_USERNAME`, `HARD_CODED_PASSWORD`, `INFLUENCER_DATABASE` or `ENDPOINT_URL`. Add them to your `.env` yourself.

**Common optional settings**

| Variable | Default | Purpose |
| --- | --- | --- |
| `ENV` | `development` | Set to `production` to disable the `dev-bypass` token |
| `DATABASE_URL` | empty (SQLite) | Leave empty for a local `creatos_connect.db`. For PostgreSQL use `postgresql+asyncpg://user:pass@host:5432/db` |
| `REDIS_URL` | `redis://localhost:6379/0` | Celery broker and cache |
| `OPENAI_MODEL` | `gpt-4o-mini` | OpenAI model |
| `GEMINI_MODEL` | `gemini-2.5-flash-lite` | Gemini model |
| `CORS_ORIGINS` | `https://creators-connect.vercel.app`, `http://localhost:3000` | JSON list or comma-separated origins allowed to call the API |
| `RATE_LIMIT_REQUESTS`, `RATE_LIMIT_WINDOW_SEC` | `60`, `60` | Per-client rate limit |
| `WEBHOOK_SECRET` | empty | If set, the internal webhook requires an `X-Webhook-Secret` header |

## API overview

Routes are defined in `app/config/api_routes.py` and registered in `app/endpoints/`.

| Area | Endpoints |
| --- | --- |
| Auth and system | `POST /api/auth/login`, `GET /api/health` |
| Search | `GET /api/dynamic-search/defaults`, `POST /api/dynamic-search/prompt` |
| Analysis | `GET /api/results/dynamic/prompt`, `GET /api/results/dynamic/prompt/all`, `GET /api/analysis/{conversation_id}`, `GET /api/results/profession-peers/status`, `POST /api/industry-comparison/insights`, `POST /api/internal-comparison/insights`, `POST /api/ai-insight` |
| Reports | `POST /api/report/generate/dynamic`, `GET /api/report/download/{report_name}` |
| Metrics | `GET /api/metrics/system/vitals`, `GET /api/metrics/{conversation_id}`, `GET /api/metrics/{conversation_id}/analysis`, plus admin routes for listing, export and comparison |
| Agent | `POST /api/agent/chat`, `GET /api/agent/health` |
| Utility | `GET /api/proxy/image?url=...` (image proxy used by the frontend) |

A typical flow: log in, send a prompt to dynamic search, fetch the results and analysis for the conversation, generate AI insights, then generate and download the PDF report.

### Authentication

`POST /api/auth/login` checks the username and password against `HARD_CODED_USERNAME` and `HARD_CODED_PASSWORD` and returns a token that the frontend stores and sends back. This is a simple single-user gate, not a full JWT system, so don't treat it as production-grade authentication. In non-production environments the token `dev-bypass` is also accepted.

## Project structure

```
backend/
├── app/
│   ├── main.py             # FastAPI app, middleware, login and health routes
│   ├── config/             # Settings, database, Celery app, route constants
│   ├── endpoints/          # Route registration per area
│   ├── controllers/        # Request handling
│   ├── services/
│   │   ├── core/           # Search pipeline, discovery, persistence
│   │   ├── analysis/       # Influencer metrics, post and industry analysis
│   │   ├── llm/            # OpenAI and Gemini clients, agent service
│   │   ├── scrapers/       # SerpAPI, BrightData, Spreadd scrapers
│   │   ├── reporting/      # PDF generation and monitoring
│   │   └── data/           # Redis, in-memory and temp-session stores
│   ├── middleware/         # Error handling, request logging, rate limiting
│   ├── models/             # SQLAlchemy models and Pydantic schemas
│   ├── worker/             # Celery tasks
│   └── utils/              # Helpers, parsing, charts, logging
├── scripts/
│   ├── init_db.py          # Create database tables
│   └── health_check.py     # Check the database, Redis and the API
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── .env.example
```

Files written at runtime (uploads, results, reports) go in `storage/` and are not committed.

## Useful commands

```bash
python -m scripts.init_db                                   # create tables
python -m scripts.health_check                              # check DB and Redis
python -m scripts.health_check --api http://localhost:8000  # also ping the API
python -m scripts.health_check --skip-redis                 # skip the Redis check
```

## Troubleshooting

- **Startup fails with a validation error**: a required variable is missing from `.env`. The error names the field.
- **Login returns 401**: check `HARD_CODED_USERNAME` and `HARD_CODED_PASSWORD`.
- **Browser calls are blocked by CORS**: add the frontend's origin to `CORS_ORIGINS`.
- **Fake-follower checks fail**: make sure Chrome or Chromium is installed (it is included in the Docker image).
- **Background refresh never runs**: start Redis and the Celery worker.
