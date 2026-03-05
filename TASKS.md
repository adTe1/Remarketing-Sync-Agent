# TASKS — Remarketing Spy Agent

Central task tracker for the full project implementation.
Each task is actionable, testable, and has a clear "done" criteria.

---

## Phase 0 — Setup, Security, API Registrations

### Repo & Infrastructure
- [ ] Create directory structure: `agent/`, `frontend/`, `tests/`, `docs/`, `data/`
- [ ] `.gitignore`: `.env`, `__pycache__`, `node_modules`, `data/*.db`, `.venv`, `htmlcov/`
- [ ] `.env.example`: all env vars with placeholder values (no real secrets)
- [ ] `requirements.txt`: FastAPI, uvicorn, httpx, beautifulsoup4, playwright, cryptography, facebook-business, google-ads, anthropic, apscheduler, pydantic
- [ ] `requirements-dev.txt`: pytest, pytest-cov, pytest-asyncio, ruff, mypy
- [ ] `docs/ENV.md`: document every env var (what it is, where to get it, required/optional)

### Backend Skeleton
- [ ] `agent/main.py`: FastAPI app, CORS config, lifespan (startup/shutdown), mount API router
- [ ] `agent/config.py`: Pydantic Settings loading from `.env`, feature flags, defaults
- [ ] `agent/database.py`: SQLite connection, create_tables(), all schema from architecture.md
- [ ] `GET /api/v1/health`: health check endpoint (DB, tokens, agent state)
- [ ] Structured logging setup: JSON format, file + console output

### API Registrations (BLOCKERS — start immediately)
- [ ] **Meta**: Create Developer account at developers.facebook.com
- [ ] **Meta**: Identity verification at facebook.com/ID
- [ ] **Meta**: Create App (type: Business), add Marketing API product
- [ ] **Meta**: Submit App Review for `ads_read`, `ads_management`, `pages_read_engagement`, `pages_show_list` (with screen recording)
- [ ] **Google**: Create Google Cloud project, enable Google Ads API
- [ ] **Google**: Create OAuth 2.0 credentials (client ID + secret)
- [ ] **Google**: Apply for Google Ads Developer Token in Manager Account
- [ ] **Google**: Submit design document for Basic Access

### Security Foundation
- [ ] Token encryption utility: Fernet encrypt/decrypt functions
- [ ] `agent/auth/token_storage.py`: CRUD encrypted tokens in SQLite
- [ ] Session middleware: httpOnly secure cookies for session management
- [ ] `docs/ENV.md`: step-by-step guide for every env var (already written)

---

## Phase 1 — Core Agent Brain

### State Machine
- [ ] `agent/core/state_machine.py`: AgentState enum, transitions dict, StateMachine class
- [ ] Guards: prevent invalid transitions, timeout detection
- [ ] State persistence: save current state to SQLite on every transition
- [ ] Crash recovery: `recover_on_startup()` — reset non-IDLE states, log warning, notify
- [ ] Unit test: `test_state_machine.py` — all valid transitions, all invalid blocked, crash recovery

### Agent Loop
- [ ] `agent/core/agent_loop.py`: AgentLoop class with `run_cycle()` method
- [ ] Scheduler integration: APScheduler triggers `run_cycle()` every N hours
- [ ] Manual trigger: API endpoint `POST /api/v1/agent/run` starts cycle
- [ ] Concurrent scan prevention: if already SCANNING, reject new run
- [ ] `max_proposals_per_cycle` guard: only top N proposals created, rest deferred to next cycle
- [ ] Unit test: `test_agent_loop.py` — mock all collectors, verify flow, verify proposal cap

### Decision Engine
- [ ] `agent/core/decision_engine.py`: scoring rules, multipliers, thresholds
- [ ] `score()` method: takes list of Changes, returns ScoredItems
- [ ] LLM escalation: `ask_llm()` for score 40-60 (optional, graceful if no key)
- [ ] Configurable thresholds via config.yaml / env
- [ ] Unit test: `test_decision_engine.py` — scoring accuracy, threshold behavior

### Proposal Generator
- [ ] `agent/core/proposal_generator.py`: rule-based template generation
- [ ] Templates: mirror, intercept, counter strategies
- [ ] LLM-enhanced generation (optional): Claude/GPT for smart copy
- [ ] Default budget from memory preferences (or €20 fallback)
- [ ] Unit test: `test_proposal_generator.py` — each strategy, LLM vs rule-based

### Memory
- [ ] `agent/core/memory.py`: log_decision(), get_recent_decisions()
- [ ] Preference learning: analyze approval/rejection patterns (after 10+ decisions)
- [ ] `get_preference_summary()`: human-readable for LLM context
- [ ] Performance feedback: pull Meta stats after 24h/72h/7d (Phase 3 dependency)
- [ ] Unit test: `test_memory.py` — logging, preference extraction

---

## Phase 1b — Data Collectors

### Site Crawler
- [ ] `agent/collectors/crawl.py`: hybrid BS4 + Playwright crawler
- [ ] HTTP-first with Playwright fallback (for JS-heavy sites)
- [ ] Max depth 3, max pages 200, robots.txt respect, 10s timeout/page
- [ ] Page classification: landing, form, service, blog, other
- [ ] Output: list of PageInfo (url, title, description, has_form, category)
- [ ] Unit test: `test_crawl.py` — sample HTML fixtures, limits, robots.txt

### Meta Ad Library
- [ ] `agent/collectors/meta_ad_library.py`: MetaAdLibraryClient class
- [ ] `fetch_ads(search_terms, country)` → list of ScrapedAd
- [ ] Pagination handling (follow `paging.next`)
- [ ] EU demographics parsing (age, gender, region)
- [ ] Rate limit handling: 200 calls/hour, exponential backoff
- [ ] Unit test: `test_meta_ad_library.py` — fixture responses, empty, malformed

### Google Ads Transparency
- [ ] `agent/collectors/google_transparency.py`: GoogleTransparencyAdapter
- [ ] Playwright backend: load page, extract ad cards
- [ ] SerpApi backend (fallback): structured JSON API
- [ ] Adapter pattern: same output regardless of backend
- [ ] Unit test: `test_google_transparency.py` — both backends, fallback behavior

### Competitor Discovery
- [ ] `agent/collectors/competitor_discovery.py`: find competitors from keywords
- [ ] Meta Ad Library keyword search → extract unique page_names
- [ ] Deduplication logic
- [ ] Unit test: `test_competitor_discovery.py` — dedup, empty results

### Custom Sites
- [ ] `agent/collectors/custom_sites.py`: Playwright + CSS selector + diff
- [ ] Content hash comparison for change detection
- [ ] Unit test: `test_custom_sites.py` — change detection, selector extraction

### Analyzers
- [ ] `agent/analyzers/audience_recipes.py`: extract targeting from ads
- [ ] `agent/analyzers/campaign_comparator.py`: diff current vs previous scan
- [ ] `agent/analyzers/ai_analyzer.py`: LLM-enhanced analysis (optional)
- [ ] Unit test for each analyzer

---

## Phase 1c — Testing Infrastructure

- [ ] `tests/conftest.py`: shared fixtures, mock HTTP server, test DB
- [ ] `tests/fixtures/sample_meta_response.json`
- [ ] `tests/fixtures/sample_google_response.json`
- [ ] `tests/fixtures/sample_crawl_html/` (2-3 sample pages)
- [ ] All unit tests from Phase 1 + 1b passing
- [ ] `pytest tests/unit/ -v` → all green
- [ ] Coverage report: `pytest --cov=agent --cov-report=html`

---

## Phase 2 — REST API

### Endpoints
- [ ] `POST /api/v1/scan` → trigger scan for domain, return job_id
- [ ] `GET /api/v1/scan/{job_id}/status` → scan progress
- [ ] `GET /api/v1/scan/{job_id}/results` → scan results (services, ads, recipes)
- [ ] `GET /api/v1/competitors` → list all competitors
- [ ] `POST /api/v1/competitors` → add competitor
- [ ] `DELETE /api/v1/competitors/{id}` → remove competitor
- [ ] `GET /api/v1/campaigns/proposals` → list pending proposals
- [ ] `POST /api/v1/campaigns/{id}/approve` → approve and create on Meta
- [ ] `POST /api/v1/campaigns/{id}/reject` → reject proposal
- [ ] `GET /api/v1/custom-sites` → list custom sites
- [ ] `POST /api/v1/custom-sites` → add custom site
- [ ] `DELETE /api/v1/custom-sites/{id}` → remove custom site
- [ ] `GET /api/v1/agent/status` → current state, last run, pending proposals
- [ ] `POST /api/v1/agent/run` → manual trigger
- [ ] `GET /api/v1/agent/logs` → recent logs (paginated)
- [ ] `GET /api/v1/notifications` → unread notifications
- [ ] `POST /api/v1/notifications/{id}/read` → mark as read

### Integration Tests
- [ ] `tests/integration/test_api_endpoints.py` — every endpoint: 200, 400, 401, 404
- [ ] `tests/integration/test_database.py` — CRUD all tables

---

## Phase 3 — Dashboard (React)

### Setup
- [ ] `frontend/`: Vite + React + TypeScript + Tailwind + shadcn/ui
- [ ] TanStack Query setup (API hooks)
- [ ] Zustand stores: auth state, selected domain, agent status
- [ ] API client: auto-generated from FastAPI OpenAPI schema (optional) or manual hooks

### Layout & Navigation
- [ ] Header: domain input, agent status indicator, notification badge
- [ ] Sidebar/tabs: Services, Meta, Google, Lists, Campaigns, Custom, Accounts, Logs

### Tabs
- [ ] **Services**: table of crawled pages (url, title, has_form, category)
- [ ] **Meta Ads**: scraped Meta ads with demographics, filters (competitor, date)
- [ ] **Google Ads**: scraped Google ads with keywords/regions
- [ ] **Audience Lists**: recipes with export CSV/JSON buttons
- [ ] **Campaigns**: proposals list with approve/reject/edit + published status
- [ ] **Custom Sites**: CRUD custom URLs, change history
- [ ] **My Accounts**: Meta + Google stats (after OAuth, Phase 4)
- [ ] **Settings**: scan interval, thresholds, budget, notifications, competitor priorities, feature flags
- [ ] **Logs**: real-time agent log viewer

### Components
- [ ] System Health widget: green/yellow/red per check
- [ ] Notification center: dropdown with unread notifications
- [ ] Proposal card: editable fields, approve/reject buttons
- [ ] Loading/error/empty states for every data-dependent component

---

## Phase 4 — OAuth Login & Own Account Stats

### Meta OAuth
- [ ] `agent/auth/meta_oauth.py`: login redirect, callback, token exchange
- [ ] `GET /api/v1/auth/meta/login` → redirect to Facebook
- [ ] `GET /api/v1/auth/meta/callback` → exchange code, encrypt token, set session
- [ ] Token refresh flow (before expiry)
- [ ] Integration test: `test_meta_oauth_flow.py` (mocked Facebook)

### Google OAuth
- [ ] `agent/auth/google_oauth.py`: login redirect, callback, token exchange
- [ ] `GET /api/v1/auth/google/login` → redirect to Google
- [ ] `GET /api/v1/auth/google/callback` → exchange code, encrypt token, set session
- [ ] Token refresh flow
- [ ] Integration test: `test_google_oauth_flow.py` (mocked Google)

### Own Account Stats
- [ ] `GET /api/v1/accounts/meta` → own Meta campaigns, insights
- [ ] `GET /api/v1/accounts/google` → own Google Ads campaigns, metrics
- [ ] Dashboard: "My Accounts" tab with real data
- [ ] "Connect Facebook" / "Connect Google Ads" buttons

---

## Phase 5 — Campaign Publishing (Approval Flow)

### Meta Campaign Creator
- [ ] `agent/publishers/meta_campaign.py`: create Campaign → AdSet → Ad
- [ ] All campaigns created as PAUSED
- [ ] Validation: creative required, budget > 0, targeting non-empty
- [ ] Error handling: parse Meta error messages, show to user
- [ ] Integration test: `test_meta_campaign_create.py` (mocked Meta API)

### Creative Handling
- [ ] `agent/publishers/creative_handler.py`: upload images to Meta
- [ ] Dashboard: image upload in proposal card
- [ ] Image hash retrieval for ad creation

### E2E Test
- [ ] `tests/e2e/test_approval_flow.py`: propose → approve → (mock) create → verify status

---

## Phase 6 — Notifications

### Slack
- [ ] `agent/notifications/slack_notifier.py`: send alerts via webhook
- [ ] Trigger on: new proposals, campaign published, health degraded, scan error
- [ ] Message format: title + body + "Open Dashboard" button

### In-App
- [ ] `agent/notifications/in_app.py`: CRUD notifications in SQLite
- [ ] Dashboard: notification badge (count), dropdown list
- [ ] Mark as read on click

---

## Phase 7 — VPS & Production

- [ ] `Dockerfile`: multi-stage (backend + frontend build)
- [ ] `docker-compose.yml`: app service, volumes, health check
- [ ] `nginx.conf`: HTTPS, reverse proxy, static files, WebSocket
- [ ] Deployment docs: `docs/deployment.md` (already written)
- [ ] Backup script: daily SQLite backup via cron
- [ ] Auto-renew SSL: certbot cron job

---

## Phase 8 — Open Source & Polish

- [ ] `LICENSE`: MIT
- [ ] `README.md`: project description, features, quick start, screenshots
- [ ] `CONTRIBUTING.md`: how to contribute, code style, PR process
- [ ] API versioning confirmed: all routes under `/api/v1/`
- [ ] Feature flags: LinkedIn disabled by default
- [ ] Final test run: all tests passing, coverage report

---

## Done Criteria

The project is **complete** when:
1. `pytest tests/ -v` → all green
2. Docker builds and runs locally
3. Dashboard loads, all tabs work with real backend data
4. Agent runs autonomously: scan → analyze → propose → notify
5. User can approve proposal → campaign created on Meta (PAUSED)
6. Health endpoint returns "healthy"
7. Documentation complete (this file + docs/)
