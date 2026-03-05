# Agent Logic — Remarketing Spy Agent

## This Is What Makes It an Agent (Not Just an App)

An application waits for you to press buttons.
An agent **thinks, decides, and acts on its own** — you only approve.

This document describes the 5 components that make SpyAgent autonomous:
1. Agent Loop (the heartbeat)
2. Decision Engine (the brain)
3. Proposal Generator (the creative)
4. State Machine (the guardrails)
5. Memory & Learning (the wisdom)

---

## 1. Agent Loop

**File**: `agent/core/agent_loop.py`

The agent loop is the main cycle that runs continuously. It is triggered by:
- **Scheduler**: Every 6 hours (configurable via `AGENT_SCAN_INTERVAL_HOURS` env var)
- **Manual**: User clicks "Run Scan Now" in dashboard
- **Event**: New competitor added (triggers immediate first scan)

### Loop Pseudocode

```python
class AgentLoop:
    def __init__(self, collectors, analyzers, decision_engine, 
                 proposal_generator, publishers, notifier, memory):
        self.state = StateMachine()
        # ... store all dependencies

    async def run_cycle(self):
        """One complete agent cycle."""
        
        # 1. SCAN
        self.state.transition_to(AgentState.SCANNING)
        competitors = await self.db.get_active_competitors()
        all_findings = []
        
        for competitor in competitors:
            findings = await self._scan_competitor(competitor)
            all_findings.extend(findings)
        
        # 2. ANALYZE
        self.state.transition_to(AgentState.ANALYZING)
        changes = await self.analyzers.compare_with_previous(all_findings)
        recipes = await self.analyzers.extract_audience_recipes(all_findings)
        
        # 3. DECIDE
        scored_items = self.decision_engine.score(changes)
        actionable = [item for item in scored_items if item.score >= 40]
        
        if not actionable:
            self.state.transition_to(AgentState.IDLE)
            return  # Nothing interesting found
        
        # 4. PROPOSE (capped to max_proposals_per_cycle)
        self.state.transition_to(AgentState.PROPOSING)
        proposals = []
        max_proposals = self.config.max_proposals_per_cycle  # default: 5
        
        for item in actionable:
            if len(proposals) >= max_proposals:
                break  # Remaining items stay in log, picked up next cycle
            
            if item.score >= 60:
                proposal = await self.proposal_generator.create(item, recipes)
                proposals.append(proposal)
            elif item.score >= 40 and self.llm_available:
                should_act = await self.decision_engine.ask_llm(item)
                if should_act:
                    proposal = await self.proposal_generator.create(item, recipes)
                    proposals.append(proposal)
        
        # Log items that didn't make the cut (for next cycle)
        if len(actionable) > max_proposals:
            await self.db.save_deferred_findings(actionable[max_proposals:])
        
        if not proposals:
            self.state.transition_to(AgentState.IDLE)
            return
        
        # 5. SAVE & NOTIFY
        await self.db.save_proposals(proposals)
        await self.notifier.send(
            title=f"SpyAgent: {len(proposals)} new proposals",
            body=self._format_summary(proposals),
            channels=["slack", "in_app"]
        )
        
        self.state.transition_to(AgentState.AWAITING_APPROVAL)
        # Agent now waits. User approves/rejects via dashboard.

    async def on_approval(self, proposal_id: str):
        """Called when user approves a proposal."""
        self.state.transition_to(AgentState.EXECUTING)
        
        proposal = await self.db.get_proposal(proposal_id)
        result = await self.publishers.meta.create_campaign(proposal)
        
        proposal.status = "published"
        proposal.meta_campaign_id = result.campaign_id
        await self.db.update_proposal(proposal)
        
        # Log to memory
        await self.memory.log_decision("proposal_approved", {
            "proposal": proposal.to_dict(),
            "targeting": proposal.targeting,
            "budget": proposal.budget_cents
        })
        
        await self.notifier.send(
            title="Campaign published!",
            body=f"{proposal.campaign_name} is live on Meta (PAUSED)."
        )
        
        self.state.transition_to(AgentState.IDLE)

    async def on_rejection(self, proposal_id: str, reason: str = ""):
        """Called when user rejects a proposal."""
        proposal = await self.db.get_proposal(proposal_id)
        proposal.status = "rejected"
        await self.db.update_proposal(proposal)
        
        await self.memory.log_decision("proposal_rejected", {
            "proposal": proposal.to_dict(),
            "reason": reason
        })
        
        # Check if all proposals decided
        pending = await self.db.count_pending_proposals()
        if pending == 0:
            self.state.transition_to(AgentState.IDLE)

    async def _scan_competitor(self, competitor):
        """Scan a single competitor across all enabled platforms."""
        findings = []
        
        if "crawl" in competitor.platforms:
            pages = await self.collectors.crawl.scan(competitor.domain)
            findings.append(Finding(type="crawl", data=pages))
        
        if "meta" in competitor.platforms:
            ads = await self.collectors.meta.fetch_ads(competitor.brand_name)
            findings.append(Finding(type="meta_ads", data=ads))
        
        if "google" in competitor.platforms:
            ads = await self.collectors.google.fetch_ads(competitor.domain)
            findings.append(Finding(type="google_ads", data=ads))
        
        return findings
```

---

## 2. Decision Engine

**File**: `agent/core/decision_engine.py`

### Approach: Priority Scoring + LLM Escalation

The decision engine uses a **rule-based scoring system** as the primary mechanism, with optional **LLM escalation** for ambiguous cases.

### Scoring Rules

```python
SCORING_RULES = {
    # Finding type                    Base score    Multipliers
    "new_competitor_discovered":       90,          # Always high priority
    "new_ad_campaign":                 70,          # Competitor launched new ad
    "targeting_changed":               55,          # Existing ad changed targeting
    "creative_changed":                45,          # Ad copy/image changed
    "ad_stopped":                      20,          # Competitor stopped an ad
    "no_change":                       5,           # Nothing new
}

MULTIPLIERS = {
    "competitor_priority_high":        1.3,         # User marked as important
    "competitor_priority_low":         0.7,         # User marked as less important
    "multiple_new_ads":                1.2,         # 3+ new ads = more important
    "matches_user_niche":              1.4,         # Keywords match user's niche
    "first_scan":                      1.5,         # First time scanning = everything new
}
```

### Decision Thresholds

```
Score >= 60  →  AUTO-PROPOSE      (create proposal immediately)
Score 40-60  →  LLM ESCALATION    (ask LLM if available, otherwise auto-propose)
Score < 40   →  LOG ONLY          (save finding, no action)
```

### Scoring Function

```python
class DecisionEngine:
    def score(self, changes: list[Change]) -> list[ScoredItem]:
        scored = []
        for change in changes:
            base = SCORING_RULES.get(change.type, 10)
            multiplier = 1.0
            
            for condition, mult in MULTIPLIERS.items():
                if self._check_condition(change, condition):
                    multiplier *= mult
            
            final_score = min(int(base * multiplier), 100)
            scored.append(ScoredItem(change=change, score=final_score))
        
        return sorted(scored, key=lambda x: x.score, reverse=True)

    async def ask_llm(self, item: ScoredItem) -> bool:
        """Ask LLM for decision on ambiguous cases (score 40-60)."""
        if not self.llm_client:
            return True  # No LLM = default to proposing
        
        prompt = f"""You are a competitive intelligence agent. 
Based on this finding, should we create a counter-campaign proposal?

Competitor: {item.change.competitor}
Finding: {item.change.type}
Details: {item.change.summary}
Score: {item.score}/100

Previous user preferences:
{self.memory.get_preference_summary()}

Answer with JSON: {{"should_propose": true/false, "reason": "..."}}"""
        
        response = await self.llm_client.generate(prompt)
        return response.should_propose
```

### Configurable

Users can adjust thresholds and multipliers via dashboard Settings page or `config.yaml`:

```yaml
decision_engine:
  auto_propose_threshold: 60
  llm_escalation_threshold: 40
  log_only_threshold: 0
  competitor_priorities:
    mentorpro.gr: high
    example.com: low
```

---

## 3. Proposal Generator

**File**: `agent/core/proposal_generator.py`

### How Proposals Are Created

```python
class ProposalGenerator:
    async def create(self, item: ScoredItem, recipes: list[AudienceRecipe]) -> CampaignProposal:
        """Generate a campaign proposal based on findings and recipes."""
        
        best_recipe = self._pick_best_recipe(item, recipes)
        
        if self.llm_client:
            proposal = await self._llm_generate(item, best_recipe)
        else:
            proposal = self._rule_based_generate(item, best_recipe)
        
        proposal.priority_score = item.score
        return proposal
```

### Rule-Based Generation (always available)

```python
TEMPLATES = {
    "mirror": {
        "name": "Mirror - {competitor}",
        "headline": "{service} - Better Rates, Better Service",
        "body": "Looking for {service}? Compare before you decide. {cta}",
    },
    "intercept": {
        "name": "Intercept - {competitor} Keywords",
        "headline": "{keyword} - Find the Best Deal",
        "body": "Searching for {keyword}? See why customers choose us. {cta}",
    },
    "counter": {
        "name": "Counter - {competitor}",
        "headline": "{service} Without the Wait",
        "body": "Get {service} faster and at better prices. {cta}",
    },
}

def _rule_based_generate(self, item, recipe) -> CampaignProposal:
    strategy = self._pick_strategy(item)  # mirror/intercept/counter
    template = TEMPLATES[strategy]
    
    return CampaignProposal(
        campaign_name=template["name"].format(competitor=item.change.competitor),
        headline=template["headline"].format(
            service=recipe.primary_interest,
            keyword=recipe.top_keyword
        ),
        body_text=template["body"].format(
            service=recipe.primary_interest,
            keyword=recipe.top_keyword,
            cta="Learn more →"
        ),
        landing_page="",  # User must fill in
        image_url="",     # User must upload
        budget_cents=self.memory.get_preferred_budget() or 2000,  # €20 default
        targeting={
            "age_min": recipe.age_min,
            "age_max": recipe.age_max,
            "gender": recipe.gender,
            "geo_locations": recipe.locations,
            "interests": recipe.interests,
        },
        platform="meta",
        status="pending",
    )
```

### LLM-Enhanced Generation (optional)

```python
async def _llm_generate(self, item, recipe) -> CampaignProposal:
    prompt = f"""Create a Meta Ads campaign proposal to counter this competitor activity.

Competitor: {item.change.competitor}
Their ad: {item.change.ad_headline} — {item.change.ad_body}
Their targeting: {recipe.to_summary()}

User preferences (from past approvals):
{self.memory.get_preference_summary()}

Generate a JSON campaign proposal with:
- campaign_name
- headline (max 40 chars)
- body_text (max 125 chars)
- suggested_budget_eur_per_day
- targeting (age_min, age_max, gender, locations, interests)
- strategy_reasoning (why this approach)"""
    
    response = await self.llm_client.generate(prompt, response_format="json")
    return CampaignProposal.from_llm_response(response, recipe)
```

### What the User Sees in Dashboard

```
┌─────────────────────────────────────────────────────┐
│  PROPOSAL: Mirror - MentorPro                        │
│  ─────────────────────────────────────────────────── │
│  Strategy: Mirror their audience                     │
│  Score: 85/100 (high priority)                       │
│                                                      │
│  Headline: [Ανακαίνιση Σπιτιού - Καλύτερες Τιμές  ]│
│  Body:     [Ψάχνετε ανακαίνιση; Δείτε γιατί μας  ] │
│            [επιλέγουν. Μάθετε περισσότερα →        ] │
│  Landing:  [https://mysite.gr/renovation           ] │
│  Image:    [📎 Upload your image]                    │
│  Budget:   [€15/day        ]                         │
│  Audience: 30-55, Attica, Home Renovation            │
│                                                      │
│  [✏️ Edit]  [❌ Reject]  [✅ Approve & Run on Meta]  │
└─────────────────────────────────────────────────────┘
```

User can edit any field before approving. Landing page and image are REQUIRED.

---

## 4. State Machine

**File**: `agent/core/state_machine.py`

### States

```
IDLE ──► SCANNING ──► ANALYZING ──► PROPOSING ──► AWAITING_APPROVAL ──► EXECUTING ──► IDLE
  ▲                                                      │                    │
  │                                                      ▼                    │
  └──────────────────── (no findings) ◄─────── (all rejected) ◄──────────────┘
```

### Allowed Transitions

```python
TRANSITIONS = {
    AgentState.IDLE: [AgentState.SCANNING],
    AgentState.SCANNING: [AgentState.ANALYZING, AgentState.IDLE],
    AgentState.ANALYZING: [AgentState.PROPOSING, AgentState.IDLE],
    AgentState.PROPOSING: [AgentState.AWAITING_APPROVAL, AgentState.IDLE],
    AgentState.AWAITING_APPROVAL: [AgentState.EXECUTING, AgentState.IDLE],
    AgentState.EXECUTING: [AgentState.IDLE],
}
```

### Implementation

```python
from enum import Enum
from datetime import datetime

class AgentState(str, Enum):
    IDLE = "idle"
    SCANNING = "scanning"
    ANALYZING = "analyzing"
    PROPOSING = "proposing"
    AWAITING_APPROVAL = "awaiting_approval"
    EXECUTING = "executing"

class StateMachine:
    def __init__(self):
        self.state = AgentState.IDLE
        self.state_since = datetime.utcnow()
        self.history: list[tuple[AgentState, datetime]] = []

    def transition_to(self, new_state: AgentState):
        if new_state not in TRANSITIONS.get(self.state, []):
            raise InvalidTransitionError(
                f"Cannot go from {self.state} to {new_state}. "
                f"Allowed: {TRANSITIONS[self.state]}"
            )
        self.history.append((self.state, self.state_since))
        self.state = new_state
        self.state_since = datetime.utcnow()

    def to_dict(self) -> dict:
        return {
            "state": self.state.value,
            "since": self.state_since.isoformat(),
            "duration_seconds": (datetime.utcnow() - self.state_since).total_seconds(),
        }
```

### Guards

- Cannot EXECUTE without going through AWAITING_APPROVAL (prevents accidental campaign creation).
- Cannot SCAN while already SCANNING (prevents duplicate runs).
- Timeout: If stuck in AWAITING_APPROVAL for > 48h, agent sends reminder notification.
- If EXECUTING fails, state goes back to IDLE (not AWAITING_APPROVAL) and error is logged.

### Crash Recovery

If the agent process crashes mid-cycle (VPS restart, OOM kill, etc.), state may be stuck in a non-IDLE value. On startup, the agent must recover:

```python
class StateMachine:
    def recover_on_startup(self):
        """Called once when the backend process starts."""
        persisted = self.db.get("agent_state")  # Read from SQLite
        
        if persisted and persisted != AgentState.IDLE:
            logger.warning(
                f"Agent was in state '{persisted}' when process stopped. "
                f"Resetting to IDLE. Last cycle may be incomplete."
            )
            self.state = AgentState.IDLE
            self.state_since = datetime.utcnow()
            self.db.set("agent_state", AgentState.IDLE)
            
            # Notify about incomplete cycle
            self.notifier.send(
                title="Agent recovered from crash",
                body=f"Was in '{persisted}' state. Reset to IDLE. Check logs.",
                channels=["slack", "in_app"]
            )
        else:
            self.state = AgentState.IDLE
            self.state_since = datetime.utcnow()

    def transition_to(self, new_state: AgentState):
        # ... existing validation ...
        # Persist to DB so we can recover
        self.db.set("agent_state", new_state.value)
```

State is persisted to SQLite on every transition. On startup, `recover_on_startup()` is called in FastAPI's lifespan. Any non-IDLE state is reset with a warning log and notification.

---

## 5. Memory & Learning

**File**: `agent/core/memory.py`

### Phase 1: Simple History (from day 1)

Every event stored in `agent_memory` table:

```python
class Memory:
    async def log_decision(self, event_type: str, data: dict):
        await self.db.insert("agent_memory", {
            "event_type": event_type,
            "data": json.dumps(data),
            "created_at": datetime.utcnow()
        })
    
    async def get_recent_decisions(self, limit: int = 20) -> list[dict]:
        return await self.db.query(
            "SELECT * FROM agent_memory ORDER BY created_at DESC LIMIT ?", 
            [limit]
        )
```

### Phase 2: Preference Learning (after 10+ decisions)

```python
class PreferenceLearner:
    async def analyze_preferences(self) -> UserPreferences:
        """Analyze approval/rejection history to detect patterns."""
        decisions = await self.memory.get_all_decisions()
        
        approved = [d for d in decisions if d["event_type"] == "proposal_approved"]
        rejected = [d for d in decisions if d["event_type"] == "proposal_rejected"]
        
        if len(approved) + len(rejected) < 10:
            return UserPreferences.default()  # Not enough data
        
        return UserPreferences(
            preferred_budget_range=self._extract_budget_range(approved),
            preferred_age_range=self._extract_age_range(approved),
            preferred_locations=self._extract_locations(approved),
            avoided_strategies=self._extract_strategies(rejected),
            approval_rate=len(approved) / (len(approved) + len(rejected)),
        )
    
    def get_preference_summary(self) -> str:
        """Human-readable summary for LLM context."""
        prefs = self.cached_preferences
        if not prefs:
            return "No preference data yet."
        return (
            f"Preferred budget: €{prefs.preferred_budget_range[0]}-"
            f"€{prefs.preferred_budget_range[1]}/day. "
            f"Age range: {prefs.preferred_age_range}. "
            f"Locations: {', '.join(prefs.preferred_locations)}. "
            f"Avoided strategies: {', '.join(prefs.avoided_strategies)}. "
            f"Approval rate: {prefs.approval_rate:.0%}."
        )
```

### Phase 3: Performance Feedback (after first published campaign)

```python
class PerformanceFeedback:
    async def collect_feedback(self):
        """Pull stats for published campaigns and learn from results."""
        published = await self.db.get_published_campaigns(min_age_hours=24)
        
        for campaign in published:
            if campaign.meta_campaign_id:
                stats = await self.meta_api.get_campaign_insights(
                    campaign.meta_campaign_id,
                    fields=["impressions", "clicks", "spend", "ctr"]
                )
                
                await self.memory.log_decision("campaign_performance", {
                    "proposal_id": campaign.id,
                    "targeting": campaign.targeting,
                    "budget": campaign.budget_cents,
                    "stats": stats,
                    "performance_rating": self._rate_performance(stats),
                })
    
    def _rate_performance(self, stats: dict) -> str:
        """Simple rating based on CTR benchmarks."""
        ctr = float(stats.get("ctr", 0))
        if ctr > 3.0:
            return "excellent"
        elif ctr > 1.5:
            return "good"
        elif ctr > 0.5:
            return "average"
        return "poor"
```

### How Memory Improves Proposals Over Time

```
Week 1:  Agent uses default templates + budget €20/day
         User approves 2, rejects 1 (budget too high)

Week 2:  Agent notices pattern: user prefers €15 budget
         Proposals now default to €15/day
         User approves 3 out of 3

Week 4:  Agent has performance data from Meta
         Mirror campaigns CTR: 2.5% (good)
         Intercept campaigns CTR: 0.8% (poor)
         Agent prioritizes Mirror strategy in future proposals

Week 8:  Agent knows:
         - Budget: €10-20/day sweet spot
         - Targeting: 25-45, urban areas
         - Strategy: Mirror > Counter > Intercept
         - Copy style: direct, with price mention
         Proposals match user preferences = high approval rate
```

---

## Configuration

All agent behavior is configurable via environment variables or `config.yaml`:

```yaml
agent:
  scan_interval_hours: 6
  max_proposals_per_cycle: 5
  auto_propose_threshold: 60
  llm_escalation_threshold: 40
  default_budget_cents: 2000
  campaign_initial_status: "PAUSED"  # PAUSED or ACTIVE

  notifications:
    slack_enabled: true
    in_app_enabled: true
    email_enabled: false

  crawl:
    max_depth: 3
    max_pages: 200
    timeout_seconds: 10
    respect_robots_txt: true

  memory:
    preference_learning_min_decisions: 10
    performance_feedback_after_hours: 24
    
  features:
    linkedin_enabled: false  # Disabled by default (ToS risk)
    ai_analysis_enabled: true  # Uses LLM if key exists
```
