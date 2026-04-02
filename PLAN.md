# Performance Marketing Agent — Implementation Plan

## Context
Ambient.ai runs paid campaigns on LinkedIn and Google Ads via Metadata.io. Today, campaign analysis is manual (see Mar 20 HTML report). We're building an automated agent that pulls live data from Metadata via MCP, tracks progress toward quarterly MQL goals, and posts structured weekly reports + mid-week anomaly alerts to Slack. The agent maintains persistent memory via a state file in GitHub so the team can share context and decisions accumulate over time.

---

## Architecture

```
┌─────────────────────────────────────┐
│  Claude Code — Manual Execution     │
│  ┌───────────────────────────────┐  │
│  │ /ad-report (on-demand skill)  │  │
│  │  - Full weekly report         │  │
│  │  - /ad-report anomaly check   │  │
│  │  - /ad-report linkedin only   │  │
│  │  - /ad-report google only     │  │
│  │  - /ad-report last 14 days    │  │
│  └───────────────────────────────┘  │
└──────────┬──────────────┬───────────┘
           │              │
     ┌─────▼─────┐  ┌────▼────────────┐
     │ Metadata  │  │ Slack Bot       │
     │ MCP Tools │  │ (post-to-slack  │
     │           │  │  via Bot Token) │
     └─────┬─────┘  └────┬────────────┘
           │              │
     Pull campaign   Post reports as
     data (LI + G)   "Henry" bot to
           │          #marketing-team
     ┌─────▼──────────────▼───────┐
     │  campaign_agent_state.json │
     │  (GitHub repo — shared)    │
     └────────────────────────────┘
```

**Components:**
1. **On-demand skill (`/ad-report`)** — invoke from Claude Code to run reports and anomaly checks
2. **Metadata MCP** — data source for all campaign/experiment metrics
3. **Slack bot script** — posts as Henry (dedicated bot identity), not as a personal account
4. **State file in GitHub** — persistent memory (goals, snapshots, decisions, alerts)

> **Why manual instead of scheduled?** Scheduled remote triggers (RemoteTrigger) run in Anthropic's cloud and cannot access the local `.env` file with the Slack bot token. Session-scoped cron jobs (CronCreate) die when the session ends. Manual execution via `/ad-report` preserves the Henry bot identity in Slack while keeping full control over timing. This can be revisited if Anthropic adds secret storage for remote triggers.

---

## Step-by-step Implementation

### Step 1: Discover Slack channel ID
Call `slack_search_channels(query="marketing-team", channel_types="private_channel")` to get the channel ID. If the bot isn't in the channel yet, instruct Alberto to invite it.

### Step 2: Clone repo and set up structure
Clone `https://github.com/AF975/Ad-Peformance-Agent.git` into the working directory. Create:
- `campaign_agent_state.json` — initial state with Q1 goals, empty snapshots, thresholds
- `reports/` directory — where weekly HTML reports will be saved
- `templates/weekly_report.html` — the HTML template (styled like the Mar 20 analysis)

Push to main.

### Step 3: Create the HTML report template
Build a styled HTML template matching the Mar 20 report design (DM Sans, card layout, Chart.js). The template will have placeholder sections that the agent fills each week:
- Goal progress bars
- Metric cards with WoW deltas
- Experiment ranking cards with badges
- Channel breakdown
- Alert cards
- Recommendations
- Charts (spend allocation + MQL trend line)

### Step 4: Run initial data pull (manual, interactive)
Execute the data collection sequence once interactively to:
- Verify all Metadata MCP tools return expected data
- Discover actual campaign/experiment names
- Build initial CTA-to-experiment mapping
- Generate the first HTML report and Slack message as a test
- Post as a Slack draft first for Alberto to review

### Step 5: Create on-demand skill — `/ad-report` ✅
Create a Claude Code skill that can be invoked manually with `/ad-report`. This is the single entry point for all report types:

| Command | What it does |
|---|---|
| `/ad-report` | Full weekly report (Monday through today) |
| `/ad-report month` | Monthly roll-up (last 30 days from today) |
| `/ad-report quarter` | Quarterly report (last 90 days from today) |
| `/ad-report anomaly check` | Mid-week anomaly check (posts only if issues found) |
| `/ad-report linkedin only` | LinkedIn experiments only |
| `/ad-report google only` | Google Ads keyword data only |
| `/ad-report last 14 days` | Custom lookback window |

The skill runs locally, so it has access to the `.env` file and posts to Slack as Henry via the bot script.

**Suggested cadence (manual):**
- **Fridays ~4pm PT** — run `/ad-report` for the full weekly report
- **Wednesdays ~10am PT** — run `/ad-report anomaly check` for mid-week anomaly detection
- **End of month** — run `/ad-report month` for the monthly roll-up
- **End of quarter** — run `/ad-report quarter` for the quarterly report

### Step 6: Create monthly roll-up report ✅

**Trigger:** `/ad-report month` — covers the **last 30 days** from the day of invocation.

The monthly report provides a bigger-picture view than weekly:
- 30-day portfolio summary with daily/weekly averages
- Budget utilization by channel (spend split, CPL per channel)
- Weekly trend table from snapshots within the 30-day window
- Top 5 performers and bottom 5 underperformers
- CTA coverage analysis (which CTAs are missing campaigns?)
- Goal pacing against quarterly targets
- Strategic recommendations (bigger-picture than weekly tactical recs)

**Slack delivery:** Posts to both `#marketing-team` and `#ext_metadata_ambient`

**HTML report:** Saved as `reports/monthly/YYYY-MM-DD.html`
**HTML template:** `templates/monthly_report.html`

**Charts:** Spend allocation (horizontal bar), weekly spend trend by channel (stacked bar), MQL progress vs target pace (line)

### Step 7: Create quarterly report ✅

**Trigger:** `/ad-report quarter` — covers the **last 90 days** from the day of invocation.

The quarterly report is the most comprehensive:
- Quarter scorecard with verdict (GOAL MET / PARTIAL / MISSED)
- MQL attainment table by channel and CTA with percentage attainment
- 90-day portfolio summary with monthly averages
- Budget utilization by channel
- Monthly trend table
- Experiment lifecycle summary (all experiments with status, spend, leads, MQLs, CPL)
- Top 5 performers
- Key learnings (what worked, what didn't, audience/channel/CTA insights)
- Strategic recommendations for next quarter (goal setting, budget, experiment strategy)
- Decision log highlights

**Slack delivery:**
Posts to both `#marketing-team` and `#ext_metadata_ambient`

**HTML report:** Saved as `reports/quarterly/YYYY-MM-DD.html`
**HTML template:** `templates/quarterly_report.html`

**Charts:** Monthly spend by channel (stacked bar), MQL progress vs target pace (line), CTA effectiveness (grouped bar: actual vs target)

### Step 8: Verify
- Test `/ad-report` for full weekly report
- Test `/ad-report anomaly check` for mid-week check
- Check `#marketing-team` channel access for bot
- Verify HTML reports in repo under `reports/`, `reports/monthly/`, `reports/quarterly/`

---

## State File Schema

```json
{
  "version": "1.0",
  "lastUpdated": "ISO timestamp",
  "goals": {
    "quarter": "Q1_FY27",
    "dateRange": { "start": "2026-02-01", "end": "2026-04-30" },
    "totalMQLTarget": 18,
    "channelTargets": {
      "LINKEDIN": {
        "total": 9,
        "byCTA": { "demo_cto": 4, "white_paper_download": 2, "webinar_on_demand_view": 3 }
      },
      "GOOGLE_ADS": {
        "total": 9,
        "byCTA": { "demo_cto": 4, "white_paper_download": 2, "webinar_on_demand_view": 3 }
      }
    }
  },
  "snapshots": [
    {
      "weekEnding": "2026-03-20",
      "portfolio": { "totalSpend": 0, "totalLeads": 0, "totalMQLs": 0, "avgCPL": 0 },
      "byChannel": { "LINKEDIN": {...}, "GOOGLE_ADS": {...} },
      "mqlProgress": {
        "LINKEDIN": { "total": 0, "demo_cto": 0, "white_paper_download": 0, "webinar_on_demand_view": 0 },
        "GOOGLE_ADS": { "total": 0, "demo_cto": 0, "white_paper_download": 0, "webinar_on_demand_view": 0 }
      },
      "topExperiments": [],
      "watchList": []
    }
  ],
  "decisionLog": [
    { "date": "ISO", "action": "description", "reason": "why", "outcome": "pending|done|skipped" }
  ],
  "alertHistory": [
    { "id": "type-entity-week", "date": "ISO", "type": "cpl_spike|lead_drought|...", "entity": "experiment-name", "message": "description", "acknowledged": false }
  ],
  "experimentTracking": {
    "knownExperiments": {
      "exp-id": {
        "name": "...", "firstSeen": "ISO", "channel": "LINKEDIN|GOOGLE_ADS",
        "status": "Active", "cta": "demo_cto|white_paper_download|webinar_on_demand_view|other",
        "lifetimeSpend": 0, "lifetimeLeads": 0, "lifetimeMQLs": 0, "lastWeekCPL": 0
      }
    }
  },
  "config": {
    "slackChannelId": "CXXXXXXXX",
    "slackEstaffChannelId": "CYYYYYYYY",
    "anomalyThresholds": {
      "cplSpikePercent": 30,
      "zeroLeadSpendThreshold": 300,
      "spendDropPercent": 50,
      "creditBalanceLowPercent": 20
    }
  }
}
```

**Key design decisions:**
- Snapshots capped at 13 (one quarter) — enough for trends, keeps file small
- Alert dedup by `{type}-{entity}` within 5 days — prevents repeat noise
- CTA mapping stored per experiment in `experimentTracking` — classify once, reuse
- Thresholds in config — tunable without editing the scheduled task

---

## Friday Report: Data Flow

```
1. Read state file from disk
2. get_credit_balance()
3. experiment_performance_stats(LINKEDIN, this week)     ─┐
4. experiment_performance_stats(GOOGLE_ADS, this week)    │ Current week
5. experiment_performance_stats(LINKEDIN, QTD)            │ Quarter-to-date
6. experiment_performance_stats(GOOGLE_ADS, QTD)         ─┘
7. account_funnel_reports(MQL, QTD)                       → MQL attribution
8. performance_metrics(ads, this week, LINKEDIN)          → Ad-level detail
9. performance_metrics(offer, QTD)                        → Offer/CTA mapping
10. search_experiments(Active)                            → Experiment roster
```

Note: `performance_metrics` only supports FACEBOOK/LINKEDIN channels. All Google data comes from `experiment_performance_stats`.

---

## Slack Message Format (single compact message)

One message to the channel (~1500-2000 chars). Full details live in the HTML report.

```
:dart: *Q1 FY27 MQL Goal Tracker*
*Overall: 6 / 18 MQLs (33%) · 58% of quarter elapsed*
Pacing: BEHIND — need to accelerate

LI: 4/9 · Google: 2/9

| CTA | LI | Google |
|-----|-----|--------|
| Demo CTO | 2/4 | 1/4 |
| White Paper | 1/2 | 0/2 |
| Webinar OD | 1/3 | 1/3 |

---
:bar_chart: *This Week* (Mar 21-27)
Spend: $1,460 (+12% WoW) · Leads: 11 (+22%) · MQLs: 4 (+100%)
CPL: $133 (-15% WoW) · Credits: 8,420 remaining (66% used)

:trophy: *Top:* Blueprint WP → Campus Police — $39 CPL, 100% MQL rate
:warning: *Watch:* Five Tenets REC — $428 spent, 0 MQLs (flagged 2 wks ago)
:warning: *Watch:* Google Ads — $900, 101 clicks, 0 leads (LP issue)

:bulb: *Rec:* Pause non-Blueprint offers on Enterprise segment — $900+ spent, 0 leads
:bulb: *Rec:* Build dedicated PPC landing pages for Google traffic

:link: *Full report:* github.com/AF975/Ad-Peformance-Agent/blob/main/reports/2026-03-27.html
_Reply in thread to ask questions · Next report: Apr 3_
```

## HTML Report (saved in `reports/YYYY-MM-DD.html`)

Full styled report matching the Mar 20 analysis format. Structure:

1. **Header** — "Weekly Performance Report · Ambient.ai · Week of {date}"
2. **Goal Tracker** — Visual progress bars for overall, by channel, and by CTA
3. **Portfolio Summary** — 4 metric cards (spend, leads, MQLs, CPL) with WoW deltas
4. **Experiment Rankings** — Cards for each experiment, ranked by CPL, with badges (Top Performer / Converting / Watch / Alert)
5. **Channel Breakdown** — LinkedIn section + Google Ads section with separate metrics
6. **Issues Flagged** — Red alert cards for problem areas
7. **Recommendations** — Numbered action items with data support
8. **Decision Log** — Timeline of actions and observations this week
9. **Spend Allocation Chart** — Chart.js horizontal bar (same as Mar 20 report)
10. **Quarter Trend** — Line chart showing weekly MQL accumulation vs. linear target pace

The HTML template will be stored in the repo as `templates/weekly_report.html` — the agent fills in data each week. Uses the same DM Sans / clean card design as the Mar 20 report.

---

## Wednesday Anomaly Check: Logic

```
1. Read state file
2. Pull Mon-Wed experiment stats (both channels)
3. Get credit balance
4. Search for Paused/Failed/WithoutSpend experiments
5. Compare against last Friday snapshot (pro-rate by 3/5)
6. Check each anomaly against alertHistory for dedup
7. IF anomalies found → post one Slack message
8. IF nothing → stay silent
9. Update state file, commit + push
```

**Anomaly triggers:**
- CPL on any experiment > 30% above last week
- Experiment with > $300 spend and 0 leads this week
- Total spend pacing < 50% of expected
- Credit balance < 20% remaining
- Experiments changed from Active to Paused/Failed
- MQL run rate projects to < 60% of quarterly target

---

## CTA Mapping Strategy

Experiments and offers are mapped to CTA types by name pattern matching:
- Contains "demo", "cto", or "bofu" → `demo_cto`
- Contains "white paper", "whitepaper", or "wp" → `white_paper_download`
- Contains "webinar", "on demand", or "mofu" → `webinar_on_demand_view`
- No match → `other` (flagged in report for manual classification)

Once classified, the CTA is stored in `experimentTracking.knownExperiments` and reused on subsequent runs. The agent only re-classifies new experiments.

---

## Memory / Decision Tracking

The `decisionLog` accumulates over time:
- New experiment detected → logged
- Experiment paused/completed → logged
- Anomaly detected → logged with suggested action
- Goal pacing status changes → logged
- Recommendations generated → logged

The Friday report includes the last 7 days of entries. Entries older than 90 days are pruned. This gives the agent context like: "We flagged this experiment 2 weeks ago and it still hasn't improved — escalating recommendation to pause."

---

## Edge Cases

| Risk | Mitigation |
|------|------------|
| Metadata returns empty data | Report notes "No active experiments" for that channel |
| CTA can't be classified | Logged as "other", flagged for human input |
| State file parse error | Start fresh with defaults, note reset in Slack |
| Git push fails | Continue report, note state save failure in Slack |
| Slack message > 5000 chars | Truncate watch list to top 3, trim recommendations |
| Not all CTAs have campaigns | Report shows 0/target and recommends launching |

---

## Report Archiving — Google Drive

The Google Drive MCP connector is read-only (search + fetch only) — it cannot create or upload files. To save weekly reports to the target Drive folder (`11RwRRTOmK7deOg82s2s6sFK8KMhaFpI5`), we have three options:

**Option A: HTML reports in Git repo + Slack (Recommended)**
- Generate a styled HTML report (similar to the Mar 20 analysis) each week
- Save it in the `Ad-Peformance-Agent` repo under `reports/YYYY-MM-DD.html`
- Post the GitHub link in the Slack message
- Team accesses reports from GitHub; no Drive dependency
- Pro: Fully automated, version-controlled. Con: Not in Drive.

**Option B: Google Apps Script webhook**
- Create a small Apps Script in the Drive folder that accepts POST requests with HTML content and creates a Google Doc
- The scheduled task calls this webhook via `curl` after generating the report
- Pro: Reports land in Drive as Google Docs. Con: Requires one-time Apps Script setup.

**Option C: Gmail attachment**
- Use the Gmail MCP `gmail_create_draft` / send to email the HTML report to a distribution list or shared mailbox
- Team members can save to Drive manually or via a Gmail filter
- Pro: No extra setup. Con: Not directly in Drive folder.

---

## Resolved Decisions
- **GitHub repo**: `https://github.com/AF975/Ad-Peformance-Agent.git` (existing)
- **Wednesday anomaly check**: 10am PT
- **Friday weekly report**: 4pm PT
- **Slack channels**: #marketing-team and #ext_metadata_ambient (all report types). Both private — will discover channel IDs and verify bot access during Step 1
- **Report archiving**: Option A — HTML reports in git repo under `reports/`, link shared in Slack
- **Slack format**: Compact single message with key metrics + link to full HTML report
