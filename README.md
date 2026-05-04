# Job Agent — Job Scraping & Ingestion Service

## 1. Project Overview

Job Agent automates the tedious process of monitoring job boards. It:

- **Scrapes** public job listing pages (LinkedIn-style) without requiring a login, using Playwright's headless Chromium browser.
- **Extracts** structured data: job title, company, location, URL, and full description (from the detail page).
- **Deduplicates** listings at the DB level using `source_url` as the unique key.
- **Stores** everything in PostgreSQL with a clean relational schema.
- **Exposes** the data via a FastAPI REST API, with an endpoint to trigger scraping on demand.

### Tech Stack

| Layer              | Technology                          |
| ------------------ | ----------------------------------- |
| Web Framework      | FastAPI 0.111                       |
| Database           | PostgreSQL 16+                      |
| ORM                | SQLAlchemy 2.0 (async, declarative) |
| Async DB Driver    | asyncpg                             |
| Browser Automation | Playwright (async Chromium)         |
| Validation         | Pydantic v2                         |
| Config             | pydantic-settings + python-dotenv   |
| Testing            | pytest + pytest-asyncio + aiosqlite |

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│                   FastAPI App                       │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ GET /jobs  │  │ POST /jobs/  │  │ POST /appli-│  │
│  │            │  │   collect    │  │  cations    │  │
│  └─────┬──────┘  └──────┬───────┘  └──────┬──────┘  │
│        │                │                  │        │
│        ▼                ▼                  ▼        │
│  ┌─────────────────────────────────────────────────┐│
│  │              Service Layer                      ││
│  │  job_service.py          scraper.py             ││
│  │  (CRUD logic)       (Playwright + DB writes)    ││
│  └──────────────────────────┬──────────────────────┘│
│                             │                       │
│                             ▼                       │
│  ┌─────────────────────────────────────────────────┐│
│  │           SQLAlchemy 2.0 Async ORM              ││
│  │  Company ──< Job ──< Application                ││
│  │                 └── JobScore (1:1)              ││
│  │                          Resume                 ││
│  └──────────────────────────┬──────────────────────┘│
└─────────────────────────────│───────────────────────┘
                              │ asyncpg
                              ▼
                    ┌──────────────────┐
                    │   PostgreSQL 16  │
                    └──────────────────┘
```

### Data Flow (Scraper)

```
LinkedIn public /jobs/search page
        │
        │ Playwright (headless Chromium)
        ▼
┌─────────────────────┐
│  Job listing cards  │  ← parsed with CSS selectors
│  (title, company,   │
│   location, url)    │
└────────┬────────────┘
         │ for each card
         ▼
┌─────────────────────┐
│  Job detail page    │  ← navigate to card URL
│  (full description, │
│   seniority, type)  │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Dedup check        │  ← SELECT WHERE source_url = ?
│  (skip if exists)   │
└────────┬────────────┘
         │ new only
         ▼
┌─────────────────────┐
│  Upsert Company     │  ← get-or-create by name
│  Insert Job         │  ← linked to company.id
└─────────────────────┘
```

### Database Schema

```mermaid

erDiagram

    COMPANIES {
        int id PK
        string name
        string website
        string industry
        string description
        string logo_url
        string headquarters
        datetime created_at
        datetime updated_at
    }

    JOBS {
        int id PK
        int company_id FK
        string title
        string location
        string job_type
        string seniority
        string description
        string source_url
        string source_platform
        boolean is_active
        datetime created_at
        datetime updated_at
    }

    APPLICATIONS {
        int id PK
        int job_id FK
        int resume_id FK
        string status
        string notes
        string cover_letter
        datetime applied_at
        datetime created_at
        datetime updated_at
    }

    RESUMES {
        int id PK
        string label
        string content
        string file_path
        boolean is_default
        datetime created_at
        datetime updated_at
    }

    JOB_SCORES {
        int id PK
        int job_id FK
        float score
        float skill_match
        float experience_match
        float location_match
        string reasoning
        datetime created_at
        datetime updated_at
    }

    COMPANIES ||--o{ JOBS : has
    JOBS ||--o{ APPLICATIONS : receives
    RESUMES ||--o{ APPLICATIONS : uses
    JOBS ||--|| JOB_SCORES : scores
```

## 3. Project Structure

```
job-agent/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── jobs.py
│   │   └── applications.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── scraper.py
│   │   └── job_service.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── company.py
│   │   ├── job.py
│   │   ├── application.py
│   │   ├── resume.py
│   │   └── job_score.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── company.py
│   │   ├── job.py
│   │   └── application.py
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   └── session.py
│   │
│   └── utils/
│       ├── __init__.py
│       └── logger.py
│
├── scripts/
│   ├── run_scraper.py
│   └── init_db.py
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_models.py
│   └── test_services.py
│
├── .env.example
├── pytest.ini
├── requirements.txt
└── README.md
```

## 4. Prerequisites

| Requirement | Version |
| ----------- | ------- |
| Python      | 3.11+   |
| PostgreSQL  | 16+     |
| pip         | Latest  |

---

## 5. Setup Instructions

### Step 1 — Clone the repository

```bash
git clone https://github.com/your-org/job-agent.git
cd job-agent
```

### Step 2 — Create a virtual environment

```bash
python -m venv .venv

# Activate it:
# macOS / Linux:
source .venv/bin/activate

# Windows (PowerShell):
.venv\Scripts\Activate.ps1
```

### Step 3 — Install Python dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### Step 4 — Install Playwright browsers

Playwright bundles its own browser binaries. Install Chromium (the only browser we use):

```bash
playwright install chromium
```

To also install the browser system dependencies (needed on Linux CI / Docker):

```bash
playwright install-deps chromium
```

Verify the installation:

```bash
python -c "from playwright.sync_api import sync_playwright; print('Playwright OK')"
```

### Step 5 — Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and update at minimum:

```dotenv
DATABASE_URL=postgresql+asyncpg://postgres:yourpassword@localhost:5432/jobagent
```

---

## 14. Ethical Scraping Guidelines

This project is built for **educational and personal use**. Before deploying in any capacity:

1. **Read the robots.txt** — Check `https://www.linkedin.com/robots.txt` (or your target site's robots.txt) and honour the `Disallow` rules.
2. **Read the Terms of Service** — Many job platforms prohibit automated scraping. Review their ToS before use.
3. **Respect rate limits** — The `SCRAPER_DELAY_MS` setting exists for this reason. Do not set it to 0.
4. **Do not scrape personal data** — Only extract publicly listed job information (title, company, description). Do not scrape recruiter contact details or user profiles.
5. **Do not bypass authentication** — This project intentionally only scrapes public pages (no login). Do not modify it to log in and scrape private content.
6. **Use a descriptive User-Agent** — In production, identify your bot in the User-Agent string.

---

---

## License

MIT — see [LICENSE](LICENSE) for details.
