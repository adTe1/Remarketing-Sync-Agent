# Environment Variables — Remarketing Spy Agent

Every env var the system uses, what it is, where to get it, and whether it's required.

All values go in `.env` file at the project root (never committed to git).
Generate a template with `cp .env.example .env`.

---

## Meta Ad Library (competitor ad scraping)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `META_AD_LIBRARY_TOKEN` | **Yes** (for Meta features) | User access token for the Graph API `/ads_archive` endpoint | 1. Go to [developers.facebook.com](https://developers.facebook.com) → create account. 2. Complete identity verification at [facebook.com/ID](https://www.facebook.com/ID) (takes 2-7 days). 3. Create App (type: Business). 4. Go to Graph API Explorer → select your app → generate User Token with no special permissions needed. 5. For long-lived token: exchange short-lived token via `/oauth/access_token` endpoint. |

## Meta Marketing API (own account stats + campaign creation)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `META_APP_ID` | **Yes** (for OAuth login) | Your Meta App's ID | Found in [App Dashboard](https://developers.facebook.com/apps/) → your app → Settings → Basic → App ID. |
| `META_APP_SECRET` | **Yes** (for OAuth login) | Your Meta App's secret key | Same page as App ID → App Secret (click "Show"). **Never commit this.** |
| `META_REDIRECT_URI` | **Yes** (for OAuth login) | OAuth callback URL | Set in App Dashboard → Facebook Login → Settings → Valid OAuth Redirect URIs. Local: `http://localhost:8000/api/v1/auth/meta/callback`. Production: `https://yourdomain.com/api/v1/auth/meta/callback`. |

**App Review required for**: `ads_read`, `ads_management`, `pages_read_engagement`, `pages_show_list`. Submit with screen recording showing how the app uses these permissions. Timeline: 2-7 business days.

## Google Ads API (own account stats)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `GOOGLE_ADS_DEV_TOKEN` | **Yes** (for Google Ads features) | Developer token for Google Ads API | 1. Create or use a [Google Ads Manager Account](https://ads.google.com/home/tools/manager-accounts/). 2. Go to Admin → API Center → apply for Developer Token. 3. Initially get Test Account access (test accounts only). 4. Apply for Basic Access: fill out the form + submit a design document (Google provides template). Timeline: 2-10 business days. |
| `GOOGLE_CLIENT_ID` | **Yes** (for OAuth login) | OAuth 2.0 client ID | 1. Go to [Google Cloud Console](https://console.cloud.google.com). 2. Create project (or use existing). 3. Enable "Google Ads API". 4. Go to APIs & Services → Credentials → Create Credentials → OAuth client ID. 5. Application type: Web application. 6. Add authorized redirect URI (see below). |
| `GOOGLE_CLIENT_SECRET` | **Yes** (for OAuth login) | OAuth 2.0 client secret | Same screen as Client ID. Click download JSON or copy the secret. **Never commit this.** |
| `GOOGLE_REDIRECT_URI` | **Yes** (for OAuth login) | OAuth callback URL | Set in Google Cloud Console → Credentials → your OAuth client → Authorized redirect URIs. Local: `http://localhost:8000/api/v1/auth/google/callback`. Production: `https://yourdomain.com/api/v1/auth/google/callback`. |

## Google Ads Transparency (competitor ads — fallback)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `SERPAPI_KEY` | **No** (optional paid fallback) | SerpApi API key for Google Ads Transparency scraping | 1. Sign up at [serpapi.com](https://serpapi.com). 2. Free tier: 100 searches/month. Paid: from ~$50/month. 3. Copy API key from dashboard. Only needed if Playwright scraping gets blocked by Google. |

## AI / LLM (optional — agent works without it)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `ANTHROPIC_API_KEY` | **No** (optional) | Claude API key for enhanced analysis and proposal copy | 1. Sign up at [console.anthropic.com](https://console.anthropic.com). 2. Create API key. Cost: ~$0.01-0.10 per agent decision. |
| `OPENAI_API_KEY` | **No** (optional, alternative to Anthropic) | OpenAI API key | 1. Sign up at [platform.openai.com](https://platform.openai.com). 2. Create API key. |

Only one LLM key needed (Anthropic OR OpenAI). If neither is set, agent uses rule-based analysis (works fine, just less "creative" proposals).

## Slack (notifications)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `SLACK_WEBHOOK_URL` | **No** (optional, for Slack alerts) | Incoming webhook URL for a Slack channel | 1. Go to [api.slack.com/apps](https://api.slack.com/apps) → Create New App. 2. Incoming Webhooks → Activate → Add New Webhook to Workspace. 3. Select channel → copy webhook URL. |

## Security (required for production)

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `TOKEN_ENCRYPTION_KEY` | **Yes** | Fernet key for encrypting OAuth tokens stored in SQLite | Generate with: `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `SESSION_SECRET` | **Yes** | Secret for signing session cookies | Generate with: `python -c "import secrets; print(secrets.token_hex(32))"` |

## Database

| Variable | Required | What it is | How to get it |
|----------|----------|-----------|---------------|
| `DATABASE_URL` | **No** (has default) | SQLite database file path | Default: `sqlite:///./data/spyagent.db`. Change if you want a different location. |

## Agent Behavior (all optional — have sane defaults)

| Variable | Required | Default | What it does |
|----------|----------|---------|-------------|
| `AGENT_SCAN_INTERVAL_HOURS` | No | `6` | How often the agent runs its scan cycle |
| `AGENT_MAX_PROPOSALS_PER_CYCLE` | No | `5` | Maximum proposals generated per cycle (top N by score) |
| `AGENT_AUTO_PROPOSE_THRESHOLD` | No | `60` | Score >= this → auto-create proposal |
| `AGENT_LLM_ESCALATION_THRESHOLD` | No | `40` | Score 40-60 → ask LLM (if key exists) |
| `AGENT_DEFAULT_BUDGET_CENTS` | No | `2000` | Default campaign budget in cents (2000 = €20/day) |
| `AGENT_CAMPAIGN_STATUS` | No | `PAUSED` | Initial status of created campaigns (PAUSED or ACTIVE) |

---

## Minimum Viable Setup

To run the agent with basic functionality (crawl + Meta ad scraping):

```
TOKEN_ENCRYPTION_KEY=<generate>
SESSION_SECRET=<generate>
META_AD_LIBRARY_TOKEN=<your token>
```

To add own account stats and campaign creation:

```
META_APP_ID=<your app id>
META_APP_SECRET=<your app secret>
META_REDIRECT_URI=http://localhost:8000/api/v1/auth/meta/callback
```

To add Google Ads:

```
GOOGLE_ADS_DEV_TOKEN=<your token>
GOOGLE_CLIENT_ID=<your client id>
GOOGLE_CLIENT_SECRET=<your client secret>
GOOGLE_REDIRECT_URI=http://localhost:8000/api/v1/auth/google/callback
```

Everything else is optional and has sensible defaults.
