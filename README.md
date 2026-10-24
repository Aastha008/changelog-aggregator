# Changelog Aggregator

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![CI](https://img.shields.io/badge/CI-Passing-success?logo=github-actions&logoColor=white)](#automated-testing)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Code Style: Ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://github.com/astral-sh/ruff)
[![FastAPI](https://img.shields.io/badge/API-FastAPI-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![SQLite](https://img.shields.io/badge/Database-SQLite-003B57?logo=sqlite&logoColor=white)](https://sqlite.org/)

An automated release monitoring and aggregation engine designed to track, scrape, sanitize, and unify software release notes across heterogeneous sources (GitHub releases, Atom/RSS feeds, and static HTML pages). 

Changelog Aggregator standardizes unstructured changelogs into a strict Semantic Versioning (`SemVer`) domain model, flags breaking changes and security vulnerabilities (CVEs), persists history in an embedded SQLite database, and exposes consolidated updates through a Rich terminal dashboard, a FastAPI REST service, and outgoing webhooks.

---

## Problem Statement

Engineering teams operate against hundreds of direct and transitive open-source dependencies, microservices, and internal developer platforms. Tracking these updates manually introduces critical liabilities:

- **Fragmented Visibility**: Release notes are dispersed across GitHub Release feeds, Atom/RSS endpoints, and custom HTML documentation portals with conflicting markup standards.
- **Silent Breakages**: Major version bumps and deprecation notices are frequently buried within dense, unstructured markdown or raw HTML tables.
- **Delayed Security Remediation**: Critical security patches and CVE advisories are lost in general maintenance updates, slowing down patching cycles.

**Target Audience:** Platform Engineers, Site Reliability Engineers (SREs), Tech Leads, and Security/DevOps teams requiring a centralized, automated pipeline to ingest, audit, and triage third-party dependency updates.

---

## Key Features

- **Multi-Source Scheduled Polling**: Automated polling engine powered by `APScheduler` capable of orchestrating recurring sync jobs for GitHub API endpoints, Atom/RSS XML feeds, and raw HTML release pages.
- **DOM Normalization & Sanitization**: Robust DOM scraping pipeline using `BeautifulSoup4` and `Requests` that isolates version tags, markdown-to-plaintext release bodies, author metadata, and publication timestamps while stripping unsafe HTML elements.
- **Semantic Versioning & Threat Classification**: Built-in regex and heuristic parsers running on `Pydantic` models to extract SemVer metadata (`MAJOR`, `MINOR`, `PATCH`), tag breaking changes (`BREAKING CHANGE`, deprecation warnings), and identify referenced security advisories (`CVE-XXXX-YYYY`).
- **Embedded Persistence**: Zero-overhead data tier built on SQLite with relational mappings for feed sources, scraped releases, and notification delivery state.
- **Dual Consumption Interface**:
  - **Rich Terminal UI**: Terminal dashboard with real-time log streaming, ANSI-colored severity badges, and dynamic filtering by tag, date range, or update classification.
  - **FastAPI Management Layer**: RESTful endpoints to register new watch targets, query aggregated feeds, and manually trigger on-demand sync jobs.
- **Outbound Webhook Dispatcher**: Automatic JSON payload dispatching to configured endpoints (e.g., Slack, Discord, or automation endpoints) upon detection of breaking changes or high-severity CVEs.

---

## Architecture & System Design

```
                     +---------------------------------------+
                     |         Feed Sources Ingestion        |
                     |  - GitHub Releases (API / HTML)       |
                     |  - Atom / RSS Feeds (XML)             |
                     |  - Custom Documentation (Raw HTML)    |
                     +---------------------------------------+
                                         |
                                         v
                     +---------------------------------------+
                     |             src/core.py               |
                     |  APScheduler Periodic Job Executor   |
                     |  BeautifulSoup4 Sanitizer / Extractor |
                     |  Regex CVE / Breaking Change Heuristics|
                     +---------------------------------------+
                                         |
                                         v
                     +---------------------------------------+
                     |            src/models.py              |
                     |  Pydantic Validation & Domain Mapping |
                     |  SemVer Classification Engine         |
                     +---------------------------------------+
                                         |
                                         v
                     +---------------------------------------+
                     |          Persistence Layer            |
                     |           SQLite Database             |
                     +---------------------------------------+
                                  |             |
                 +----------------+             +----------------+
                 |                                               |
                 v                                               v
+----------------------------------+           +----------------------------------+
|            src/api.py            |           |           src/main.py            |
|       FastAPI REST Engine        |           |      Rich CLI Dashboard          |
|  - Ingest Configuration          |           |  - Semantic Version Highlighting |
|  - Aggregated Log Queries        |           |  - Real-time Log Filtering       |
|  - Manual Sync Triggers          |           |  - Outbound Webhook Dispatcher   |
+----------------------------------+           +----------------------------------+
```

### Module Responsibilities

| Module | Core Purpose |
| :--- | :--- |
| `src/models.py` | Pydantic domain models enforcing valid SemVer representations, source configurations, persistence schemas, and JSON serialization. |
| `src/core.py` | Primary business logic: scheduler execution, HTTP fetching with exponential backoff, HTML/XML DOM extraction, regex-based breaking change/CVE heuristics, and webhook dispatching. |
| `src/api.py` | FastAPI application exposing programmatic interfaces to manage feed targets, query changelogs, and invoke scraping cycles. |
| `src/main.py` | Application bootstrapping layer: initializes the SQLite schema, triggers `APScheduler`, and launches the process in either CLI Dashboard mode or API server mode. |

---

## Technology Stack

- **Runtime**: Python 3.11+
- **HTTP Engine**: `Requests` (with custom timeout and retry session adapters)
- **DOM Scraping & Extraction**: `BeautifulSoup4` with `lxml` parser
- **Data Validation & Schemas**: `Pydantic v2`
- **Task Scheduling**: `APScheduler 3.x`
- **Terminal Visualization**: `Rich`
- **REST Framework**: `FastAPI` / `Uvicorn`
- **Persistence**: `SQLite3`
- **Test Framework**: `pytest`, `pytest-mock`, `pytest-asyncio`

---

## Installation & Setup

### Prerequisites

- Python 3.11 or higher
- Git
- SQLite 3.3x+

### 1. Clone Repository

```bash
git clone https://github.com/your-org/changelog-aggregator.git
cd changelog-aggregator
```

### 2. Set Up Virtual Environment

```bash
# Create virtual environment
python3 -m venv .venv

# Activate environment
# Linux/macOS:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate
```

### 3. Install Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

---

## Environment Variables Configuration

Copy the example configuration to initialize your local `.env`:

```bash
cp .env.example .env
```

### Configuration Options

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `DATABASE_URL` | String | `sqlite:///./changelogs.db` | SQLite connection string for persistent storage. |
| `POLL_INTERVAL_MINUTES` | Integer | `60` | Execution frequency for background scheduled polling tasks. |
| `HTTP_TIMEOUT_SECONDS` | Integer | `15` | Network request timeout for feed and HTML DOM scraping. |
| `GITHUB_TOKEN` | String | `""` | Optional personal access token to prevent GitHub API rate limits. |
| `WEBHOOK_URL` | String | `""` | Target webhook URL (Slack, Discord, internal endpoint) for alerts. |
| `API_HOST` | String | `0.0.0.0` | Host interface binding for the FastAPI server. |
| `API_PORT` | Integer | `8000` | Port allocation for the FastAPI server. |
| `LOG_LEVEL` | String | `INFO` | Application log verbosity (`DEBUG`, `INFO`, `WARNING`, `ERROR`). |

---

## Usage

### 1. Interactive Terminal Dashboard

Launch the Rich-powered terminal dashboard to inspect consolidated changelogs:

```bash
# Run default dashboard
python -m src.main --mode cli

# Run with custom filtering
python -m src.main --mode cli --filter-type major --days 14
```

### 2. Running the REST API Server

Launch the FastAPI application to manage feeds and query historical releases:

```bash
python -m src.main --mode api
```

The service initializes at `http://0.0.0.0:8000`. Interactive OpenAPI documentation is accessible at `http://localhost:8000/docs`.

### API Endpoints Reference

#### Register a Monitored Source
```bash
curl -X POST "http://localhost:8000/api/v1/sources" \
     -H "Content-Type: application/json" \
     -d '{
       "name": "FastAPI",
       "url": "https://github.com/tiangolo/fastapi/releases.atom",
       "source_type": "atom"
     }'
```

#### Trigger an On-Demand Poll
```bash
curl -X POST "http://localhost:8000/api/v1/sources/sync"
```

#### Retrieve Consolidated Releases
```bash
# Query all breaking changes within the last 7 days
curl -X GET "http://localhost:8000/api/v1/changelogs?update_type=MAJOR&has_cve=true"
```

#### Example Output Payload
```json
[
  {
    "id": 104,
    "source_name": "FastAPI",
    "version": "0.110.0",
    "semver": {
      "major": 0,
      "minor": 110,
      "patch": 0,
      "raw": "0.110.0"
    },
    "update_type": "MINOR",
    "published_at": "2024-03-03T18:24:00Z",
    "has_breaking_changes": false,
    "detected_cves": [],
    "summary": "Support Python 3.12 syntax, fix dependency overrides for sub-dependencies.",
    "source_url": "https://github.com/tiangolo/fastapi/releases/tag/0.110.0"
  }
]
```

---

## Automated Testing

The suite uses `pytest` with mocked network boundaries to validate HTML DOM parsing, SemVer extraction algorithms, security heuristic detection, and FastAPI endpoints.

Run the test suite:

```bash
# Run all tests
pytest

# Run tests with detailed output and coverage reporting
pytest -v --cov=src --cov-report=term-missing

# Target specific test suites
pytest tests/test_core.py -v
pytest tests/test_api.py -v
```

---

## Project Structure

```
changelog-aggregator/
├── .env.example              # Template environment variables
├── .gitignore                # Git ignore rules for Python/virtualenvs
├── requirements.txt          # Production and testing runtime dependencies
├── README.md                 # Project architecture and technical documentation
├── src/
│   ├── __init__.py           # Package namespace initialization
│   ├── models.py             # Pydantic schemas and database persistence models
│   ├── core.py               # DOM parsing, SemVer logic, scheduler, webhooks
│   ├── api.py                # FastAPI HTTP routing and request controllers
│   └── main.py               # Entrypoint for CLI Dashboard or API execution
└── tests/
    ├── __init__.py           # Test suite namespace
    ├── test_core.py          # Unit tests for core scraper, SemVer, and CVE heuristics
    └── test_api.py           # Integration tests for FastAPI endpoints
```

---

## Roadmap

- [ ] **Multi-Target Webhook Router**: Per-source webhook routing allowing specific team channels to receive notifications only for assigned packages.
- [ ] **LLM Summarization Worker**: Optional integration with lightweight local models (e.g., via Ollama) to generate 2-sentence executive digests of monolithic changelogs.
- [ ] **Package Registry Native Adapters**: Native integration with PyPI JSON API, npm Registry API, and Crates.io API to bypass raw HTML parsing where structured metadata is present.
- [ ] **Grafana / Prometheus Exporter**: Expose scrape success rates, change frequency metrics, and CVE detection counters via a `/metrics` Prometheus endpoint.

---

## License

This project is licensed under the terms of the **MIT License**. See the [LICENSE](LICENSE) file for complete terms.