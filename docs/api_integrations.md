# API Integrations — Remarketing Spy Agent

## Overview

The system integrates with 5 external APIs. Each integration is behind an **adapter interface** so it can be swapped, mocked, or disabled without affecting the rest of the system.

```
┌──────────────────────────────────────────────┐
│              ADAPTER LAYER                    │
│                                              │
│  MetaAdLibraryAdapter   ← reads competitor   │
│  GoogleTransparencyAdapter ← reads competitor│
│  MetaMarketingAdapter   ← reads own + writes │
│  GoogleAdsAdapter       ← reads own          │
│  SlackAdapter           ← sends alerts       │
└──────────────────────────────────────────────┘
```

---

## 1. Meta Ad Library API (READ competitor ads)

**Purpose**: Find competitor ads, get demographics, discover new advertisers by keyword.

**Base URL**: `https://graph.facebook.com/v25.0/ads_archive`

**Authentication**: User access token (requires identity verification at facebook.com/ID).

**Permissions needed**: None special (Ad Library is public data). Identity verification only.

### Key Endpoints

**Search ads by keyword or page name:**
```
GET /ads_archive
  ?search_terms=MentorPro
  &ad_reached_countries=['GR']
  &ad_active_status=ACTIVE
  &ad_type=ALL
  &fields=id,ad_creation_time,ad_creative_bodies,page_name,
          publisher_platforms,ad_snapshot_url,
          demographic_distribution,delivery_by_region
  &limit=100
  &access_token={TOKEN}
```

**Response fields we use:**

| Field | Type | What we do with it |
|-------|------|-------------------|
| `page_name` | string | Competitor identification |
| `ad_creative_bodies` | string[] | Ad copy analysis |
| `publisher_platforms` | string[] | Which platforms (FB/IG) |
| `demographic_distribution` | object[] | Age/gender targeting extraction |
| `delivery_by_region` | object[] | Geographic targeting extraction |
| `ad_creation_time` | timestamp | Detect new campaigns |
| `ad_snapshot_url` | string | Link to ad preview |

**Rate limits**: 200 calls/hour per token. Handled with exponential backoff.

**Error handling:**
- 190 (token expired) → trigger refresh flow
- 4 (rate limit) → backoff + retry after delay
- 100 (unknown error) → log + retry 3x → mark as failed

**Setup blockers:**
- Identity verification at facebook.com/ID (2-7 business days)
- Meta Developer App (type: Business)

---

## 2. Google Ads Transparency Center (READ competitor ads)

**Purpose**: Find competitor Google Search/Display ads.

**Important**: No official API exists. We use an **adapter pattern** with two backends.

### Backend A: Playwright (free, default)

**URL**: `https://adstransparency.google.com/?domain={domain}`

**Method**: Headless Chromium loads the page, waits for ad cards, extracts data.

```python
async with async_playwright() as p:
    browser = await p.chromium.launch(headless=True)
    page = await browser.new_page()
    await page.goto(f"https://adstransparency.google.com/?domain={domain}")
    await page.wait_for_selector(".ad-card", timeout=15000)
    cards = await page.query_selector_all(".ad-card")
    # Extract headline, description, format, region, dates
```

**Risks**: Google may change HTML structure, show CAPTCHAs, or block headless browsers.

### Backend B: SerpApi (paid fallback, ~$50/month)

**URL**: `https://serpapi.com/search?engine=google_ads_transparency_center`

**Method**: Structured API call, returns JSON.

```python
params = {
    "engine": "google_ads_transparency_center",
    "advertiser_id": domain,
    "region": "GR",
    "api_key": SERPAPI_KEY
}
response = httpx.get("https://serpapi.com/search", params=params)
```

### Adapter Interface

```python
class GoogleTransparencyAdapter:
    """Unified interface regardless of backend."""
    
    async def fetch_ads(self, domain: str) -> list[ScrapedAd]:
        try:
            return await self._playwright_fetch(domain)
        except (PlaywrightBlockedError, TimeoutError):
            if self.serpapi_key:
                return await self._serpapi_fetch(domain)
            raise GoogleAdsUnavailableError(domain)
```

Both backends return the same `ScrapedAd` model. The rest of the system doesn't know or care which backend was used.

---

## 3. Meta Marketing API (READ own stats + WRITE campaigns)

**Purpose**: Two functions:
1. **Read**: View your own campaigns, ad sets, ads, performance insights.
2. **Write**: Create campaigns when user approves a proposal.

**Base URL**: `https://graph.facebook.com/v25.0`

**Authentication**: OAuth 2.0 (user logs in via Facebook, grants permissions).

**Permissions needed:**
- `ads_read` — View own campaigns and insights
- `ads_management` — Create/update campaigns (requires App Review)
- `pages_read_engagement` — Dependency for ads_management
- `pages_show_list` — Dependency for ads_management

### READ: Own Account Insights

**Get ad accounts:**
```
GET /me/adaccounts
  ?fields=id,name,currency,account_status
  &access_token={TOKEN}
```

**Get campaign insights:**
```
GET /act_{AD_ACCOUNT_ID}/insights
  ?fields=campaign_name,impressions,clicks,spend,cpm,ctr
  &time_range={"since":"2026-02-01","until":"2026-03-03"}
  &level=campaign
  &access_token={TOKEN}
```

**Response:**
```json
[
  {
    "campaign_name": "Spring Promo",
    "impressions": "12500",
    "clicks": "340",
    "spend": "45.20",
    "cpm": "3.62",
    "ctr": "2.72"
  }
]
```

### WRITE: Create Campaign (approval flow)

**Step 1 — Campaign:**
```
POST /act_{AD_ACCOUNT_ID}/campaigns
  name=SpyAgent - Mirror MentorPro
  objective=OUTCOME_TRAFFIC
  status=PAUSED
  special_ad_categories=[]
```
Returns: `{"id": "campaign_123"}`

**Step 2 — Ad Set:**
```
POST /act_{AD_ACCOUNT_ID}/adsets
  campaign_id=campaign_123
  name=Mirror Audience 30-55 Attica
  daily_budget=2000              # €20/day in cents
  billing_event=IMPRESSIONS
  optimization_goal=LINK_CLICKS
  targeting={"geo_locations":{"countries":["GR"]},"age_min":30,"age_max":55}
  start_time=2026-03-05T00:00:00
```
Returns: `{"id": "adset_456"}`

**Step 3 — Ad:**
```
POST /act_{AD_ACCOUNT_ID}/ads
  adset_id=adset_456
  name=Renovation Services Ad
  creative={"title":"...","body":"...","link_url":"...","image_hash":"..."}
  status=PAUSED
```
Returns: `{"id": "ad_789"}`

**All campaigns created as PAUSED** for safety. User activates in Meta Ads Manager or via future "Activate" button.

**Setup blockers:**
- Meta App (Business type) + App Review (2-7 business days)
- Screen recording of app functionality required for review
- Business verification if managing other accounts

---

## 4. Google Ads API (READ own stats)

**Purpose**: View your own Google Ads campaigns and performance metrics.

**Authentication**: OAuth 2.0 (Google Sign-In) + Developer Token.

**Required:**
- Google Ads Manager Account
- Developer Token (application + review process)
- OAuth 2.0 client credentials (Google Cloud Console)

### Query Own Campaigns

```python
from google.ads.googleads.client import GoogleAdsClient

client = GoogleAdsClient.load_from_dict({
    "developer_token": GOOGLE_ADS_DEV_TOKEN,
    "client_id": GOOGLE_CLIENT_ID,
    "client_secret": GOOGLE_CLIENT_SECRET,
    "refresh_token": stored_refresh_token,
    "login_customer_id": customer_id
})

query = """
    SELECT
        campaign.name,
        campaign.status,
        metrics.impressions,
        metrics.clicks,
        metrics.cost_micros,
        metrics.ctr,
        metrics.average_cpc
    FROM campaign
    WHERE segments.date DURING LAST_30_DAYS
    ORDER BY metrics.impressions DESC
"""

ga_service = client.get_service("GoogleAdsService")
stream = ga_service.search_stream(customer_id=customer_id, query=query)

for batch in stream:
    for row in batch.results:
        campaign = {
            "name": row.campaign.name,
            "status": row.campaign.status.name,
            "impressions": row.metrics.impressions,
            "clicks": row.metrics.clicks,
            "cost": row.metrics.cost_micros / 1_000_000,
            "ctr": row.metrics.ctr,
        }
```

**Setup blockers:**
- Developer Token application (2-10 business days)
- Design document required for Basic/Standard access
- Test Account Access granted immediately (test accounts only)

---

## 5. Slack API (SEND notifications)

**Purpose**: Proactive alerts when agent finds something or has proposals ready.

**Method**: Slack Incoming Webhook (simplest) or MCP Slack (already configured).

### Webhook (simple)

```python
import httpx

async def send_slack_alert(title: str, body: str, link: str):
    payload = {
        "blocks": [
            {"type": "header", "text": {"type": "plain_text", "text": title}},
            {"type": "section", "text": {"type": "mrkdwn", "text": body}},
            {"type": "actions", "elements": [
                {"type": "button", "text": {"type": "plain_text", "text": "Open Dashboard"}, "url": link}
            ]}
        ]
    }
    await httpx.post(SLACK_WEBHOOK_URL, json=payload)
```

### MCP Slack (already configured)

The project has MCP Slack server configured. Can be used for richer interactions.

**No setup blockers** — just needs a webhook URL or MCP configuration.

---

## Integration Status Matrix

| Integration | Read/Write | Free? | Blocker | Timeline |
|------------|-----------|-------|---------|----------|
| Meta Ad Library | Read | Yes | Identity verification | 2-7 days |
| Google Transparency | Read | Yes (Playwright) / ~$50/mo (SerpApi) | None | Immediate |
| Meta Marketing API | Read + Write | Yes | App Review | 2-7 days |
| Google Ads API | Read | Yes | Developer Token | 2-10 days |
| Slack | Write | Yes | Webhook URL | Immediate |

---

## Environment Variables Required

```bash
# Meta Ad Library (competitor ads - read)
META_AD_LIBRARY_TOKEN=         # User access token

# Meta Marketing API (own account + campaign creation)
META_APP_ID=                   # From developers.facebook.com
META_APP_SECRET=               # From developers.facebook.com
META_REDIRECT_URI=             # OAuth callback URL

# Google Ads API (own account stats)
GOOGLE_ADS_DEV_TOKEN=          # Developer token
GOOGLE_CLIENT_ID=              # From Google Cloud Console
GOOGLE_CLIENT_SECRET=          # From Google Cloud Console
GOOGLE_REDIRECT_URI=           # OAuth callback URL

# Google Ads Transparency (fallback)
SERPAPI_KEY=                   # Optional, for SerpApi fallback

# AI / LLM (optional)
ANTHROPIC_API_KEY=             # For Claude-powered analysis
OPENAI_API_KEY=                # Alternative: OpenAI

# Slack
SLACK_WEBHOOK_URL=             # Incoming webhook URL

# Security
TOKEN_ENCRYPTION_KEY=          # Fernet key for encrypting stored tokens
SESSION_SECRET=                # Secret for session cookies

# Database
DATABASE_URL=sqlite:///./spyagent.db  # Default SQLite path
```

All stored in `.env` file (never committed). Template provided in `.env.example`.
