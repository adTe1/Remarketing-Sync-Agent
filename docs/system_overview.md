# System Overview — Remarketing Spy Agent

## What This System Does

SpyAgent is an **autonomous competitive intelligence agent** that:

1. **Discovers** your competitors (from keywords or your domain)
2. **Monitors** their ads across Meta Ad Library and Google Ads Transparency
3. **Analyzes** their targeting patterns (demographics, interests, regions)
4. **Proposes** counter-campaigns with audience recipes and ad copy
5. **Publishes** approved campaigns on Meta Ads with one click
6. **Learns** from your decisions to improve future proposals

You only do ONE thing: **approve or reject**. The agent handles the rest.

---

## System Components

### 1. Agent Core (the brain)

The autonomous loop that runs continuously without human intervention.

**Agent Loop** (`agent/core/agent_loop.py`)
- Triggered by scheduler (default: every 6 hours) or manually via dashboard.
- Orchestrates: scan → compare → decide → propose → notify → wait → execute.
- Runs as a background task inside the FastAPI process (APScheduler).

**Decision Engine** (`agent/core/decision_engine.py`)
- Receives scan findings (new ads, changed targeting, stopped campaigns).
- Assigns priority score (0-100) per finding based on configurable rules.
- Score >= 60 → auto-generate proposal.
- Score 40-60 → ask LLM for opinion (if LLM key exists), otherwise auto-generate.
- Score < 40 → log only, no action.

**State Machine** (`agent/core/state_machine.py`)
- Tracks agent state: `IDLE → SCANNING → ANALYZING → PROPOSING → AWAITING_APPROVAL → EXECUTING → MONITORING → IDLE`.
- Guards prevent invalid transitions (cannot execute without approval).
- Dashboard shows current state in real-time.

**Proposal Generator** (`agent/core/proposal_generator.py`)
- Takes audience recipe + competitor ads as input.
- Rule-based mode: template headlines/copy + targeting from recipe.
- LLM mode (optional): Claude/GPT generates smart copy, refined targeting, better headlines.
- Output: `CampaignProposal` object ready for approval.

**Memory** (`agent/core/memory.py`)
- **History**: Every scan, proposal, approval/rejection stored in SQLite.
- **Preference Learning**: After 10+ decisions, detects patterns (preferred budgets, targeting styles, copy tone).
- **Performance Feedback**: After campaign runs 24h/72h/7d, pulls stats and logs what worked.

### 2. Data Collectors (hands)

Modules that gather information from the outside world.

| Collector | Source | Method | Cost |
|-----------|--------|--------|------|
| `crawl.py` | Any website | BeautifulSoup + Playwright | Free |
| `meta_ad_library.py` | Meta Ad Library | Official Graph API | Free |
| `google_transparency.py` | Google Ads Transparency | Playwright + SerpApi fallback | Free / ~$50/mo |
| `custom_sites.py` | Custom URLs | Playwright + CSS selector | Free |
| `competitor_discovery.py` | Meta + Google | Keyword search in ad libraries | Free |

### 3. Analyzers (thinking)

Modules that extract insights from raw data.

| Analyzer | Input | Output |
|----------|-------|--------|
| `audience_recipes.py` | Scraped ads + demographics | Audience definitions (age, gender, location, interests) |
| `ai_analyzer.py` | Ads + context | Enhanced targeting patterns, competitive insights |
| `campaign_comparator.py` | Current scan + previous scan | Diff: new ads, changed targeting, stopped campaigns |

### 4. Publishers (actions)

Modules that act on the real world.

| Publisher | Action | API |
|-----------|--------|-----|
| `meta_campaign.py` | Create Campaign → AdSet → Ad | Meta Marketing API |
| `creative_handler.py` | Upload images/video for ads | Meta Marketing API |

### 5. Notifications (voice)

How the agent communicates with you.

| Channel | When | Priority |
|---------|------|----------|
| **Slack** | New proposals ready, campaign published, errors | Urgent |
| **In-App** | Badge count, notification center | Normal |
| **Email** | Weekly digest (future) | Low |

### 6. REST API (backbone)

How the dashboard talks to the backend.

| Endpoint Group | Purpose |
|----------------|---------|
| `/api/v1/scan/*` | Trigger scans, get results |
| `/api/v1/competitors/*` | CRUD competitors |
| `/api/v1/campaigns/*` | View proposals, approve/reject |
| `/api/v1/custom-sites/*` | CRUD custom monitoring |
| `/api/v1/auth/*` | OAuth login/callback (Meta, Google) |
| `/api/v1/accounts/*` | Own account stats (Meta, Google Ads) |
| `/api/v1/health` | System health checks |
| `/api/v1/notifications/*` | Get/mark-read notifications |
| `/api/v1/agent/*` | Agent state, logs, config |

### 7. Dashboard (face)

React app that provides the user interface.

**Tabs:**
- **Services**: Crawled pages from competitor site (landing pages, forms, promotions)
- **Meta Ads**: Scraped Meta/Instagram ads with demographics
- **Google Ads**: Scraped Google ads with keywords/regions
- **Audience Lists**: Generated audience recipes with export (CSV/JSON)
- **Campaigns**: Proposed campaigns + approve/reject/edit + published status
- **Custom Sites**: Monitored URLs + change alerts
- **My Accounts**: Own Meta/Google Ads stats (after OAuth login)
- **Logs**: Real-time agent activity log
- **Health**: System status widget (green/yellow/red per component)
- **Settings**: Agent configuration (see below)

**Settings Page — What the User Can Configure:**

The Settings page exposes agent behavior controls without requiring env var changes or restarts:

| Setting | Default | What it controls |
|---------|---------|-----------------|
| Scan interval | 6 hours | How often the agent auto-scans competitors |
| Max proposals per cycle | 5 | Cap on proposals generated per run (top N by priority score) |
| Auto-propose threshold | 60 | Score >= this → auto-create proposal without asking |
| LLM escalation threshold | 40 | Scores in this "grey zone" (40-60) get LLM review if key exists |
| Default budget | €20/day | Pre-filled budget on new proposals |
| Campaign initial status | PAUSED | Whether created campaigns start PAUSED or ACTIVE |
| Notification channels | Slack + In-App | Toggle which channels receive alerts |
| Competitor priorities | per competitor | Mark competitors as high/normal/low priority (affects scoring multiplier) |
| Feature flags | LinkedIn OFF | Enable/disable optional features like LinkedIn scraping |

Settings are stored in SQLite (not env vars) so they persist and can be changed at runtime from the dashboard without restarting the backend. Env vars serve as initial defaults on first run.

---

## How It All Works Together — End-to-End Flow

```
HOUR 0: You add competitor "mentorpro.gr" in the dashboard
   └─► POST /api/v1/competitors {"domain": "mentorpro.gr"}
   └─► Competitor stored in SQLite

HOUR 0: Agent starts first scan (manual trigger or scheduled)
   └─► State: IDLE → SCANNING
   └─► crawl.py: Crawls mentorpro.gr, finds 7 service pages
   └─► meta_ad_library.py: Finds 4 active Facebook/Instagram ads
   └─► google_transparency.py: Finds 3 active Google Search ads
   └─► All stored in scraped_ads table

HOUR 0 + 30s: Agent analyzes
   └─► State: SCANNING → ANALYZING
   └─► campaign_comparator.py: First run = all ads are "new"
   └─► audience_recipes.py: Extracts targeting → "Age 30-55, Female, Attica, Home Renovation"
   └─► decision_engine.py: Score = 85 (new competitor, many ads) → PROPOSE

HOUR 0 + 45s: Agent generates proposals
   └─► State: ANALYZING → PROPOSING
   └─► proposal_generator.py: Creates 2 campaign proposals
       ├─ Mirror Campaign: Same targeting, your creative
       └─ Intercept Campaign: Same keywords, your landing page
   └─► Proposals stored in campaign_proposals table

HOUR 0 + 50s: Agent notifies you
   └─► State: PROPOSING → AWAITING_APPROVAL
   └─► Slack: "SpyAgent found 7 ads from mentorpro.gr. 2 proposals ready."
   └─► In-App: Badge "2 pending"

HOUR 1: You open dashboard, review proposals
   └─► Edit budget (€20/day → €15/day)
   └─► Upload your own image
   └─► Click "Approve & Run on Meta"

HOUR 1 + 5s: Agent executes
   └─► State: AWAITING_APPROVAL → EXECUTING
   └─► meta_campaign.py: POST /campaigns (PAUSED) → POST /adsets → POST /ads
   └─► Campaign created on Meta (status: PAUSED for your review)
   └─► memory.py: Logs approval + preferences (budget €15, targeting style)
   └─► State: EXECUTING → MONITORING

HOUR 6: Agent runs scheduled scan
   └─► Same flow, but campaign_comparator.py compares with previous run
   └─► Only NEW or CHANGED ads trigger proposals

DAY 2: Performance feedback
   └─► Agent pulls campaign stats from Meta (impressions, clicks, cost)
   └─► memory.py: Logs performance → improves future proposals

DAY 7+: Agent is smarter
   └─► Knows you prefer €15 budgets, tech-focused copy, 25-45 age range
   └─► Proposals match your style → higher approval rate
```

---

## What Works Without Configuration

Even without any API keys, the agent can:
- Crawl any website (find pages, forms, services)
- Compare scan results over time
- Generate rule-based audience recipes
- Serve the dashboard with all UI features

**With Meta Ad Library token** (free, requires identity verification):
- Scrape competitor ads from Meta/Instagram
- Get EU demographics data

**With Meta Marketing API** (free, requires App Review):
- View own campaigns/stats
- Create campaigns via one-click approval

**With Google Ads Developer Token** (free, requires application):
- View own Google Ads campaigns/stats

**With LLM API key** (paid, optional):
- Smarter proposal generation (better copy, refined targeting)
- AI-powered competitive analysis
- LLM escalation for ambiguous decisions
