# Architecture — Remarketing Spy Agent

## Project

**Name**: Remarketing-Sync-Agent (SpyAgent)
**Type**: Autonomous Competitive Intelligence Agent + Dashboard
**Purpose**: Monitors competitors' ads across Meta/Google, discovers targeting patterns, proposes counter-campaigns, and publishes them on Meta with one-click approval.

---

## Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Backend | **FastAPI** + uvicorn | Async, auto OpenAPI docs, production-ready |
| Frontend | **Vite + React + TypeScript** | Fast build, type safety |
| UI | **Tailwind CSS + shadcn/ui** | Modern, consistent, minimal CSS |
| State (frontend) | **TanStack Query + Zustand** | API cache + global state, no Redux overhead |
| Database | **SQLite** (default) / Supabase (optional) | Zero infrastructure locally, upgrade path to cloud |
| Browser Automation | **Playwright Python** | Headless Chromium, production-ready scraping |
| HTML Parsing | **BeautifulSoup4** | Fast static page parsing |
| OAuth | **authlib + httpx** | Meta/Google OAuth 2.0 |
| Token Encryption | **cryptography** (Fernet) | Symmetric encryption for stored tokens |
| Meta APIs | **facebook-business** SDK + httpx | Ad Library (read) + Marketing API (write) |
| Google Ads API | **google-ads** Python client | Own account stats |
| AI (optional) | **anthropic** SDK / **openai** SDK | Enhanced targeting analysis, proposal copy |
| Scheduling | **APScheduler** (in-process) | No Redis/Celery needed for single-user |
| Testing | **pytest** + pytest-cov + pytest-asyncio | Full test pyramid |
| Deployment | **Docker** + **nginx** + **Let's Encrypt** | VPS-ready, HTTPS |

---

## Directory Structure

```
Remarketing-Sync-Agent/
├── agent/                          # Python backend + agent logic
│   ├── main.py                     # FastAPI app entry point
│   ├── config.py                   # Settings, env vars, feature flags
│   ├── database.py                 # SQLite connection, migrations
│   │
│   ├── core/                       # AGENT BRAIN
│   │   ├── agent_loop.py           # Main agent loop (scheduler → scan → decide → act)
│   │   ├── decision_engine.py      # Priority scoring + LLM escalation
│   │   ├── state_machine.py        # Agent states (idle/scanning/proposing/etc.)
│   │   ├── proposal_generator.py   # Creates campaign proposals from findings
│   │   └── memory.py              # History, preference learning, feedback
│   │
│   ├── collectors/                 # DATA COLLECTION TOOLS
│   │   ├── crawl.py               # Site crawler (BS4 + Playwright hybrid)
│   │   ├── meta_ad_library.py     # Meta Ad Library API client
│   │   ├── google_transparency.py # Google Ads Transparency adapter (Playwright + SerpApi)
│   │   ├── custom_sites.py        # Custom URL monitoring (Playwright + diff)
│   │   └── competitor_discovery.py # Find competitors from keywords/domain
│   │
│   ├── analyzers/                  # DATA ANALYSIS
│   │   ├── audience_recipes.py    # Extract targeting from ads (rule-based)
│   │   ├── ai_analyzer.py         # LLM-enhanced analysis (optional)
│   │   └── campaign_comparator.py # Compare current vs previous scan results
│   │
│   ├── publishers/                 # ACTION EXECUTION
│   │   ├── meta_campaign.py       # Create Campaign → AdSet → Ad on Meta
│   │   └── creative_handler.py    # Upload/manage ad creatives
│   │
│   ├── auth/                       # AUTHENTICATION
│   │   ├── meta_oauth.py          # Meta OAuth 2.0 flow
│   │   ├── google_oauth.py        # Google OAuth 2.0 flow
│   │   └── token_storage.py       # Encrypted token store (Fernet + SQLite)
│   │
│   ├── notifications/              # PROACTIVE ALERTS
│   │   ├── slack_notifier.py      # Slack messages via webhook/MCP
│   │   ├── in_app.py              # In-app notification store
│   │   └── email_notifier.py      # Email digest (future)
│   │
│   ├── api/                        # REST API ROUTES
│   │   ├── v1/
│   │   │   ├── scan.py            # POST /scan, GET /scan/:id
│   │   │   ├── competitors.py     # CRUD competitors
│   │   │   ├── campaigns.py       # Proposals + approve/reject
│   │   │   ├── custom_sites.py    # CRUD custom sites
│   │   │   ├── auth.py            # OAuth login/callback
│   │   │   ├── accounts.py        # My Meta / My Google stats
│   │   │   ├── health.py          # Health checks
│   │   │   ├── notifications.py   # Get pending notifications
│   │   │   └── agent_status.py    # Agent state, logs
│   │   └── deps.py                # Shared dependencies (DB session, auth)
│   │
│   └── models/                     # DATA MODELS
│       ├── competitor.py
│       ├── scan_result.py
│       ├── scraped_ad.py
│       ├── audience_recipe.py
│       ├── campaign_proposal.py
│       ├── notification.py
│       └── token.py
│
├── frontend/                       # React dashboard
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── api/                   # API client (auto-generated from OpenAPI)
│   │   ├── stores/                # Zustand stores (auth, selectedDomain)
│   │   ├── hooks/                 # TanStack Query hooks
│   │   ├── components/
│   │   │   ├── layout/            # Header, Sidebar, DomainInput
│   │   │   ├── tabs/              # Services, Meta, Google, Lists, Custom, Logs, Accounts
│   │   │   ├── campaigns/         # Similar campaigns, proposals, approval
│   │   │   ├── health/            # System health widget
│   │   │   └── notifications/     # Notification center, badges
│   │   └── pages/
│   │       ├── Dashboard.tsx
│   │       ├── Competitors.tsx
│   │       ├── Campaigns.tsx
│   │       └── Settings.tsx
│   ├── package.json
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   └── tsconfig.json
│
├── tests/                          # Full test pyramid
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   ├── health/
│   ├── fixtures/
│   └── conftest.py
│
├── docs/                           # Documentation (this folder)
│   ├── architecture.md            # This file
│   ├── system_overview.md
│   ├── api_integrations.md
│   ├── agent_logic.md
│   └── deployment.md
│
├── TASKS.md                        # Central task tracker
├── README.md                       # Project overview + quick start
├── LICENSE                         # MIT
├── .env.example                    # Template for env vars (no secrets)
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── requirements.txt                # Production dependencies
└── requirements-dev.txt            # Dev/test dependencies
```

---

## Module Dependency Graph

```
┌─────────────────────────────────────────────────────────────┐
│                    AGENT CORE (brain)                        │
│  agent_loop → decision_engine → proposal_generator          │
│       ↕              ↕                  ↕                   │
│  state_machine    memory          notifications             │
└──────────┬──────────┬──────────────────┬────────────────────┘
           │          │                  │
     ┌─────▼─────┐  ┌─▼──────────┐  ┌───▼──────────┐
     │ COLLECTORS │  │ ANALYZERS  │  │ PUBLISHERS   │
     │ crawl      │  │ recipes    │  │ meta_campaign│
     │ meta_ads   │  │ ai_analyze │  │ creative     │
     │ google_ads │  │ comparator │  └──────────────┘
     │ custom     │  └────────────┘
     │ discovery  │
     └─────┬──────┘
           │
     ┌─────▼──────┐
     │  DATABASE   │  ← SQLite (scan_results, competitors,
     │  + AUTH     │     tokens, proposals, memory, notifications)
     └─────┬──────┘
           │
     ┌─────▼──────┐
     │  REST API   │  ← /api/v1/* endpoints
     └─────┬──────┘
           │
     ┌─────▼──────┐
     │  DASHBOARD  │  ← React frontend
     └────────────┘
```

---

## Data Flow

```
1. SCHEDULED TRIGGER (every 6h or manual)
   │
2. AGENT LOOP starts
   │
3. FOR EACH competitor:
   ├── crawl.py         → site pages, services, forms
   ├── meta_ad_library  → active Meta/IG ads + EU demographics
   ├── google_transparency → active Google ads
   │
4. COMPARE with previous run (campaign_comparator.py)
   │ Output: new_ads[], changed_targeting[], stopped_campaigns[]
   │
5. DECISION ENGINE scores findings
   │ Score >= 60 → PROPOSE
   │ Score 40-60 + LLM key → ASK LLM
   │ Score < 40 → LOG ONLY
   │
6. PROPOSAL GENERATOR creates campaign proposals
   │ Rule-based: template targeting + copy
   │ LLM-enhanced: smart copy + refined targeting
   │
7. NOTIFICATION sent (Slack + in-app)
   │ "Found 3 new competitor ads. 2 proposals ready."
   │
8. STATE → AWAITING_APPROVAL
   │
9. USER reviews in dashboard → APPROVE / REJECT / EDIT
   │
10. IF APPROVED:
    ├── meta_campaign.py → Create Campaign → AdSet → Ad (PAUSED)
    ├── memory.py → Log decision + preferences
    └── STATE → IDLE

11. PERFORMANCE FEEDBACK (24h/72h/7d later)
    └── Pull stats → update memory → improve future proposals
```

---

## Database Schema (SQLite)

```sql
-- Tracked competitors
CREATE TABLE competitors (
    id INTEGER PRIMARY KEY,
    domain TEXT NOT NULL UNIQUE,
    brand_name TEXT,
    platforms TEXT DEFAULT '["meta","google"]',  -- JSON array
    is_active INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Raw scan results per run
CREATE TABLE scan_results (
    id INTEGER PRIMARY KEY,
    competitor_id INTEGER REFERENCES competitors(id),
    scan_type TEXT NOT NULL,  -- 'crawl', 'meta_ads', 'google_ads'
    data TEXT NOT NULL,        -- JSON blob
    scanned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Parsed ads from any platform
CREATE TABLE scraped_ads (
    id INTEGER PRIMARY KEY,
    competitor_id INTEGER REFERENCES competitors(id),
    platform TEXT NOT NULL,    -- 'meta', 'google', 'linkedin'
    ad_external_id TEXT,
    headline TEXT,
    body TEXT,
    creative_url TEXT,
    platforms TEXT,             -- JSON: ["facebook","instagram"]
    demographics TEXT,          -- JSON: age, gender, region
    first_seen_at TIMESTAMP,
    last_seen_at TIMESTAMP,
    is_active INTEGER DEFAULT 1
);

-- Generated audience recipes
CREATE TABLE audience_recipes (
    id INTEGER PRIMARY KEY,
    competitor_id INTEGER REFERENCES competitors(id),
    strategy TEXT NOT NULL,     -- 'mirror', 'intercept', 'counter', 'expand'
    name TEXT NOT NULL,
    age_min INTEGER,
    age_max INTEGER,
    gender TEXT,
    locations TEXT,             -- JSON
    interests TEXT,             -- JSON
    keywords TEXT,              -- JSON
    estimated_size TEXT,
    confidence_score REAL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Campaign proposals (agent-generated)
CREATE TABLE campaign_proposals (
    id INTEGER PRIMARY KEY,
    based_on_recipe_id INTEGER REFERENCES audience_recipes(id),
    competitor_id INTEGER REFERENCES competitors(id),
    status TEXT DEFAULT 'pending',  -- pending/approved/rejected/published/failed
    campaign_name TEXT,
    headline TEXT,
    body_text TEXT,
    landing_page TEXT,
    image_url TEXT,
    budget_cents INTEGER,
    targeting TEXT,              -- JSON (full targeting spec)
    platform TEXT DEFAULT 'meta',
    priority_score INTEGER,
    meta_campaign_id TEXT,       -- Set after publish
    meta_adset_id TEXT,
    meta_ad_id TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    decided_at TIMESTAMP,
    published_at TIMESTAMP
);

-- Custom site monitoring
CREATE TABLE custom_sites (
    id INTEGER PRIMARY KEY,
    url TEXT NOT NULL,
    css_selector TEXT,
    check_interval_hours INTEGER DEFAULT 24,
    last_content_hash TEXT,
    last_checked_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Encrypted OAuth tokens
CREATE TABLE tokens (
    id INTEGER PRIMARY KEY,
    provider TEXT NOT NULL UNIQUE,  -- 'meta', 'google'
    encrypted_access_token TEXT,
    encrypted_refresh_token TEXT,
    expires_at TIMESTAMP,
    scopes TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Agent memory (decisions + preferences)
CREATE TABLE agent_memory (
    id INTEGER PRIMARY KEY,
    event_type TEXT NOT NULL,  -- 'proposal_approved', 'proposal_rejected', 'campaign_performance'
    data TEXT NOT NULL,         -- JSON: what was proposed, what was decided, why
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- In-app notifications
CREATE TABLE notifications (
    id INTEGER PRIMARY KEY,
    type TEXT NOT NULL,         -- 'new_ads', 'proposal_ready', 'campaign_published', 'alert'
    title TEXT NOT NULL,
    body TEXT,
    link TEXT,                  -- Dashboard deep link
    is_read INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Audit log
CREATE TABLE audit_log (
    id INTEGER PRIMARY KEY,
    module TEXT NOT NULL,
    action TEXT NOT NULL,
    domain TEXT,
    details TEXT,               -- JSON
    level TEXT DEFAULT 'info',  -- info/warn/error
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Security Architecture

```
┌──────────────┐     HTTPS (TLS)     ┌──────────────┐
│   Browser    │◄───────────────────►│    nginx     │
│  (React app) │                     │  reverse     │
└──────────────┘                     │  proxy       │
       │                             └──────┬───────┘
       │ httpOnly cookie                     │
       │ (session ID only)                   │
       │                             ┌───────▼──────┐
       └─────────────────────────────│   FastAPI    │
                                     │   backend    │
                                     └───────┬──────┘
                                             │
                              ┌──────────────┼──────────────┐
                              │              │              │
                        ┌─────▼────┐  ┌──────▼─────┐ ┌─────▼────┐
                        │ SQLite   │  │ Meta API   │ │ Google   │
                        │ (tokens  │  │ (server    │ │ Ads API  │
                        │ encrypted│  │  side only)│ │ (server  │
                        │ Fernet)  │  └────────────┘ │ side)    │
                        └──────────┘                 └──────────┘
```

**Rules:**
1. Frontend NEVER has access tokens — only httpOnly session cookie.
2. All external API calls from backend only.
3. Tokens encrypted at rest (Fernet) in SQLite.
4. Secrets only in env vars / .env (never committed).
5. HTTPS mandatory in production (Let's Encrypt).
