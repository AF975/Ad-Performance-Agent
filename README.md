# Henry — The Ad Performance Agent

Henry is an AI-powered performance marketing agent built on Claude Code. He pulls live campaign data from Metadata.io, analyzes it against quarterly MQL goals, generates styled HTML reports, and posts summaries to Slack — all automatically.

Henry runs as a Claude Code skill (`/ad-report`) and can also be triggered by scheduled tasks on a weekly cadence.

---

## Table of Contents

- [What Henry Does](#what-henry-does)
- [Architecture Overview](#architecture-overview)
- [How to Run Henry](#how-to-run-henry)
- [What Gets Produced](#what-gets-produced)
- [Repository Structure](#repository-structure)
- [The State File](#the-state-file)
- [The Execution Pipeline](#the-execution-pipeline)
- [Data Sources](#data-sources)
- [CTA Classification](#cta-classification)
- [Anomaly Detection](#anomaly-detection)
- [Slack Delivery](#slack-delivery)
- [HTML Reports](#html-reports)
- [Scheduled Tasks](#scheduled-tasks)
- [Configuration & Thresholds](#configuration--thresholds)
- [Troubleshooting](#troubleshooting)

---

## What Henry Does

Every week (or on demand), Henry:

1. **Pulls live data** from Metadata.io — LinkedIn experiment stats, Google Ads keyword stats, credit balance, active experiment roster, and offer-level performance
2. **Classifies experiments** by CTA type (Demo, White Paper, Webinar) using offer name pattern matching
3. **Computes analytics** — week-over-week deltas, goal pacing, top performers, watch list, CTA coverage gaps
4. **Generates an HTML report** — a styled, interactive page with progress bars, experiment cards, charts, alerts, and recommendations
5. **Posts a Slack summary** to #marketing-team as "Henry — The Ad Performance Agent" (a dedicated Slack bot, not a personal account)
6. **Updates persistent memory** — snapshots, experiment tracking, decision log, alert history
7. **Commits everything to GitHub** — report and state file are versioned and pushed

---

## Architecture Overview

```
                        ┌──────────────────────┐
                        │   Claude Code         │
                        │                       │
                        │  /ad-report skill     │
                        │  (manual execution)   │
                        └───┬──────────┬────────┘
                            │          │
                  ┌─────────▼──┐  ┌────▼────────────┐
                  │ Metadata   │  │ Slack Bot        │
                  │ MCP Server │  │ (post-to-slack.sh│
                  │            │  │  via Bot Token)  │
                  └─────┬──────┘  └────┬────────────┘
                        │              │
                  Pull campaign   Post summaries to
                  data (LI + G)   #marketing-team
                        │              │
                  ┌─────▼──────────────▼─────────┐
                  │  campaign_agent_state.json    │
                  │  (GitHub repo — versioned)    │
                  └──────────────────────────────┘
                        │
                  ┌─────▼──────────────────────┐
                  │  reports/YYYY-MM-DD.html    │
                  │  (GitHub Pages — viewable)  │
                  └────────────────────────────┘
```

**Key components:**

| Component | What it is | Where it lives |
|---|---|---|
| **Skill definition** | The instructions that tell Claude how to run the report | `~/.claude/skills/ad-report/skill.md` |
| **State file** | Persistent memory — goals, snapshots, experiment tracking, decisions | `campaign_agent_state.json` (repo root) |
| **HTML template** | The report layout with `{{PLACEHOLDER}}` tokens | `templates/weekly_report.html` |
| **Generated reports** | Completed HTML reports, one per run | `reports/YYYY-MM-DD.html` |
| **Slack bot script** | Shell script that posts to Slack via the bot token | `scripts/post-to-slack.sh` |
| **Bot credentials** | Slack bot OAuth token | `.env` (gitignored, never committed) |

---

## How to Run Henry

Henry runs manually via the `/ad-report` skill in Claude Code. Type the command and Henry handles the rest.

| Command | What it does |
|---|---|
| `/ad-report` | Full weekly report (Monday through today) |
| `/ad-report month` | Monthly roll-up (last 30 days from today) |
| `/ad-report quarter` | Quarterly report (last 90 days from today) |
| `/ad-report anomaly check` | Mid-week anomaly check (posts only if issues found) |
| `/ad-report linkedin only` | LinkedIn experiments only |
| `/ad-report google only` | Google Ads keyword data only |
| `/ad-report last 14 days` | Custom lookback window instead of current week |

### Suggested cadence

| Report | When to run | Command |
|---|---|---|
| Weekly report | Fridays ~4pm PT | `/ad-report` |
| Mid-week anomaly check | Wednesdays ~10am PT | `/ad-report anomaly check` |
| Monthly roll-up | End of month | `/ad-report month` |
| Quarterly report | End of quarter | `/ad-report quarter` |

> **Why manual?** Henry posts to Slack as a dedicated bot (not your personal account). This requires a local bot token stored in `.env`, which isn't accessible to remote scheduled agents. Manual execution preserves Henry's bot identity.

---

## What Gets Produced

Each run generates three outputs:

### 1. Slack Message (in #marketing-team)

A structured text summary posted by "Henry — The Ad Performance Agent" containing:
- Key metrics (spend, leads, MQLs, CPL)
- Goal pacing status with channel and CTA breakdown
- Top performer highlight
- Critical issues
- Top recommendations
- Link to the full HTML report

### 2. HTML Report (on GitHub Pages)

A full interactive report viewable at:
```
https://af975.github.io/Ad-Peformance-Agent/reports/YYYY-MM-DD.html
```

Contains: goal progress bars, metric cards with WoW deltas, experiment ranking cards with badges, alert cards, recommendation cards, spend allocation chart, MQL trend chart, and decision log.

### 3. Updated State File

`campaign_agent_state.json` is updated with a new snapshot, experiment tracking updates, and decision log entries, then committed and pushed to GitHub.

---

## Repository Structure

```
Ad-Peformance-Agent/
├── .env                          # Slack bot token (gitignored)
├── .gitignore                    # Keeps .env out of git
├── campaign_agent_state.json     # Persistent memory (the brain)
├── PLAN.md                       # Original implementation plan
├── README.md                     # This file
├── scripts/
│   └── post-to-slack.sh          # Slack bot posting script
├── templates/
│   ├── weekly_report.html        # Weekly HTML report template
│   ├── monthly_report.html       # Monthly HTML report template (last 30 days)
│   └── quarterly_report.html     # Quarterly HTML report template (last 90 days)
└── reports/
    ├── 2026-03-24.html           # Weekly reports
    ├── monthly/                  # Monthly roll-ups (/ad-report month)
    └── quarterly/                # Quarterly reports (/ad-report quarter)
```

---

## The State File

`campaign_agent_state.json` is Henry's persistent memory. It's what allows him to compare this week to last week, track experiments over time, and remember past decisions.

### Schema

```json
{
  "version": "1.0",
  "lastUpdated": "ISO timestamp",

  "goals": {
    "quarter": "Q1_FY27",
    "dateRange": { "start": "2026-02-01", "end": "2026-04-30" },
    "totalMQLTarget": 18,
    "channelTargets": {
      "LINKEDIN": { "total": 9, "byCTA": { "demo_cto": 4, "white_paper_download": 2, "webinar_on_demand_view": 3 } },
      "GOOGLE_ADS": { "total": 9, "byCTA": { "demo_cto": 4, "white_paper_download": 2, "webinar_on_demand_view": 3 } }
    }
  },

  "snapshots": [
    {
      "weekEnding": "YYYY-MM-DD",
      "portfolio": { "totalSpend": 0, "totalLeads": 0, "totalMQLs": 0, "avgCPL": 0 },
      "byChannel": { "LINKEDIN": { ... }, "GOOGLE_ADS": { ... } },
      "mqlProgress": { "LINKEDIN": { "total": 0, "demo_cto": 0, ... }, "GOOGLE_ADS": { ... } },
      "topExperiments": [ ... ],
      "watchList": [ ... ]
    }
  ],

  "experimentTracking": {
    "knownExperiments": {
      "<experiment-id>": {
        "name": "Blueprint WP → Enterprise v6",
        "firstSeen": "2026-03-24",
        "channel": "LINKEDIN",
        "status": "Active",
        "cta": "white_paper_download",
        "lifetimeSpend": 151,
        "lifetimeLeads": 3,
        "lifetimeMQLs": 1,
        "lastWeekCPL": 50
      }
    }
  },

  "decisionLog": [
    { "date": "ISO", "action": "description", "reason": "why", "outcome": "pending|done|skipped" }
  ],

  "alertHistory": [
    { "id": "type-entity-week", "date": "ISO", "type": "cpl_spike|cta_gap|...", "entity": "name", "message": "...", "acknowledged": false }
  ],

  "config": {
    "slackChannelId": "C08UY0AF5FA",
    "anomalyThresholds": { "cplSpikePercent": 30, "zeroLeadSpendThreshold": 300, ... }
  }
}
```

### Key design decisions

- **Snapshots** are capped at 13 (one quarter of weeks). Enough for trend charts, keeps the file small.
- **Experiment tracking** stores CTA classification per experiment — classify once on first sight, reuse forever.
- **Decision log** entries older than 90 days are pruned. Alert history entries older than 30 days are pruned.
- **Config thresholds** are tunable without editing the skill definition.

---

## The Execution Pipeline

When Henry runs, he follows these 9 steps in order:

### Step 1: Read State File
Load `campaign_agent_state.json`. Extract goals, previous snapshots (for WoW comparison), known experiments, and config. If the file is missing, start fresh with default goals.

### Step 2: Determine Date Ranges
Calculate: current week (Monday–today), quarter-to-date (Feb 1–today), time progress (% of quarter elapsed). If the user passed `last N days`, override the week range.

### Step 3: Pull Data from Metadata
Make 6 parallel MCP calls to Metadata.io:

| Call | What it gets |
|---|---|
| LinkedIn stats (current week) | This week's experiment performance, ranked by CPL |
| LinkedIn stats (QTD) | Quarter-to-date totals for goal tracking |
| Google Ads keywords (QTD) | Keyword-level data (Google doesn't use the experiments endpoint) |
| Active experiments | Full roster of currently running experiments |
| Credit balance | Remaining Metadata credits |
| Offer-level performance | Offer names for CTA classification |

### Step 4: Classify Experiments by CTA
Each experiment is mapped to a CTA type (`demo_cto`, `white_paper_download`, `webinar_on_demand_view`, or `other`) based on its offer name. Previously classified experiments are reused from the state file.

### Step 5: Compute Metrics
- **Portfolio summary**: total spend, leads, MQLs, CPL for the week
- **WoW comparison**: delta percentages vs. last week's snapshot
- **Goal pacing**: actual MQLs vs. expected at this point in the quarter (ON TRACK / BEHIND / AT RISK)
- **Top performers**: top 3 experiments by CPL (with at least 1 lead)
- **Watch list**: experiments with high spend + zero leads, CPL spikes, or extreme CPL
- **CTA coverage gaps**: which CTA types have zero MQLs
- **Recommendations**: 2–4 specific, data-backed action items

### Step 6: Generate HTML Report
Read the template, replace all `{{PLACEHOLDER}}` tokens with computed values, save to `reports/YYYY-MM-DD.html`.

### Step 7: Post to Slack
Write the message to a temp file, call `scripts/post-to-slack.sh` to post as the bot identity.

### Step 8: Update State File
Add new snapshot, update experiment tracking, append to decision log, prune old entries, write to disk.

### Step 9: Git Commit and Push
Stage the report and state file, commit with message `Weekly report YYYY-MM-DD`, push to GitHub.

---

## Data Sources

Henry pulls all data from **Metadata.io** via MCP (Model Context Protocol) tools.

### LinkedIn Data
Pulled via `experiment_performance_stats`. Each "experiment" in Metadata is a LinkedIn campaign with a specific audience + offer + ad creative combination. Returns: spend, impressions, clicks, leads, MQLs, CPL, CTR.

### Google Ads Data
Google Ads experiments don't appear in the experiment stats endpoint. Instead, Henry uses `experiments_keywords_stats` which returns keyword-level data. He aggregates across all keywords to get Google channel totals.

### Credit Balance
Metadata uses a credit system. Henry tracks remaining credits and flags when they're low or denied.

### Offer-Level Data
The `performance_metrics` endpoint (offer view) provides offer names, which Henry uses to classify experiments by CTA type.

---

## CTA Classification

Henry maps every experiment to one of four CTA (call-to-action) types. This drives the goal tracking breakdown.

| CTA Type | Offer name patterns (case-insensitive) |
|---|---|
| `demo_cto` | "demo", "cto", "bofu", "request a demo" |
| `white_paper_download` | "white paper", "whitepaper", "wp", "blueprint", "pulsar", "tenets", "agentic" |
| `webinar_on_demand_view` | "webinar", "on demand", "mofu", "btb" |
| `other` | Anything that doesn't match — flagged for manual classification |

Classification is stored in `experimentTracking.knownExperiments` and reused on subsequent runs. Henry only classifies new experiments.

---

## Anomaly Detection

Henry flags issues using configurable thresholds:

| Anomaly | Threshold | Example |
|---|---|---|
| Zero-lead spend | Experiment spent > $300 with 0 leads | "Five Tenets → Higher Ed: $428 spent, 0 leads" |
| CPL spike | CPL increased > 30% vs. last week | "Experiment X CPL up 45% WoW" |
| Extreme CPL | CPL > 2x portfolio average | "Experiment Y at $600 CPL vs $300 avg" |
| CTA gap | A CTA type has 0 campaigns running | "Demo CTO: 0 campaigns, 44% of target uncovered" |
| Channel down | All experiments on a channel are paused | "Google Ads: all campaigns paused" |
| Credit balance | Credits low or denied | "Credits: 0 remaining, DENIED" |

Anomalies are deduped in `alertHistory` to avoid repeat noise.

---

## Slack Delivery

Henry posts to #marketing-team (`C08UY0AF5FA`) as a dedicated Slack bot, not as any team member's personal account.

### How the bot works

1. A Slack app ("Henry — The Ad Performance Agent") was created at `api.slack.com/apps` with `chat:write` scope
2. The bot's OAuth token (`xoxb-...`) is stored in `.env` (gitignored)
3. `scripts/post-to-slack.sh` reads the token and posts via the Slack Web API (`chat.postMessage`)
4. The skill writes the message to a temp file, then calls the script

### Why not use the MCP Slack tool?

The MCP Slack tool authenticates with a personal user token, so messages appear as that person. The bot script uses a separate bot token, giving Henry his own identity in Slack.

### Message format

```
*Weekly Report — Mar 24, 2026* (QTD: Feb 1 – Mar 24)

*Key Metrics:*
• Spend: $6,479 | Leads: 20 | MQLs: 6 | CPL: $324
• 22 active LinkedIn experiments, 0 Google (all paused)

*Goal Pacing: BEHIND*
• 6/18 MQLs (33%) at 59% of quarter elapsed
• LinkedIn: 6/9 (on track) — all from White Paper CTA
• Google: 0/9 (at risk — paused)

*Top Performer:* Blueprint WP → Enterprise v6 at $50 CPL, 33% MQL rate

*Critical Issues:*
1. Demo CTO gap — 0 campaigns covering 44% of MQL target
2. Google Ads paused — $985 spent, 0 leads

*Top Recommendations:*
1. Launch Demo CTO campaigns immediately
2. Scale Blueprint experiments to $50+/day

Full report: https://af975.github.io/Ad-Peformance-Agent/reports/2026-03-24.html
```

---

## HTML Reports

Reports are generated from `templates/weekly_report.html` — a single-file HTML page with inline CSS and Chart.js.

### Viewing reports

Reports are published via GitHub Pages at:
```
https://af975.github.io/Ad-Peformance-Agent/reports/YYYY-MM-DD.html
```

### Report sections

1. **Goal Tracker** — Progress bars for overall, LinkedIn, and Google MQL targets with a CTA breakdown table
2. **Portfolio Summary** — Four metric cards (spend, leads, MQLs, CPL) with week-over-week deltas
3. **LinkedIn Experiments** — Individual cards for each experiment, ranked by efficiency, with badges (Top Performer / Converting / Generating Leads / Watch / Alert)
4. **Google Ads** — Summary card for the Google channel
5. **Issues Flagged** — Red alert cards for each problem
6. **Recommendations** — Blue cards with specific action items
7. **Decision Log** — Timeline of agent observations and decisions
8. **Spend Allocation Chart** — Horizontal bar chart showing spend by experiment (green = converting, red = zero leads)
9. **MQL Trend Chart** — Line chart showing cumulative MQLs vs. target pace over time

### Template design

- Font: DM Sans
- Dark mode support via `prefers-color-scheme`
- Mobile responsive (grid collapses on small screens)
- Charts powered by Chart.js 4.4 (loaded from CDN)

---

## Reporting Cadence

All reports are triggered manually via `/ad-report` in Claude Code.

| Report | Suggested timing | Command | Status |
|---|---|---|---|
| **Weekly report** | Fridays ~4pm PT | `/ad-report` | ✅ Built |
| **Anomaly check** | Wednesdays ~10am PT | `/ad-report anomaly check` | ✅ Built |
| **Monthly roll-up** | End of month | `/ad-report month` | ✅ Built |
| **Quarterly report** | End of quarter | `/ad-report quarter` | ✅ Built |

### Anomaly Check

A lighter check that pulls Mon–Wed data and compares against the last snapshot (pro-rated). Only posts to Slack if anomalies are detected. Stays silent if everything looks normal.

### Monthly Roll-Up (`/ad-report month`)

Covers the **last 30 days** from the date of invocation. Includes:
- 30-day portfolio summary with daily/weekly averages
- Budget utilization by channel (spend split, CPL per channel)
- Weekly trend table from snapshots within the window
- Top 5 performers and bottom 5 underperformers
- CTA coverage analysis
- Strategic recommendations (bigger-picture than weekly)
- Posts to both #marketing-team and #ext_metadata_ambient
- Report saved to `reports/monthly/YYYY-MM-DD.html`

### Quarterly Report (`/ad-report quarter`)

Covers the **last 90 days** from the date of invocation. The most comprehensive report:
- Quarter scorecard with verdict (GOAL MET / PARTIAL / MISSED)
- MQL attainment table by channel and CTA with percentage attainment
- 90-day portfolio summary with monthly averages
- Experiment lifecycle summary (all experiments with status and lifetime metrics)
- Monthly trend table
- Key learnings (what worked, what didn't, channel/CTA insights)
- Strategic recommendations for next quarter
- Posts to both #marketing-team and #ext_metadata_ambient
- Report saved to `reports/quarterly/YYYY-MM-DD.html`

---

## Configuration & Thresholds

All configurable values live in the `config` section of `campaign_agent_state.json`:

```json
{
  "config": {
    "slackChannelId": "C08UY0AF5FA",
    "slackEstaffChannelId": "G01PC9HRFDK",
    "anomalyThresholds": {
      "cplSpikePercent": 30,
      "zeroLeadSpendThreshold": 300,
      "spendDropPercent": 50,
      "creditBalanceLowPercent": 20
    }
  }
}
```

| Threshold | Default | What it means |
|---|---|---|
| `cplSpikePercent` | 30 | Flag if CPL increases > 30% WoW |
| `zeroLeadSpendThreshold` | $300 | Flag if experiment spends > $300 with 0 leads |
| `spendDropPercent` | 50 | Flag if total spend drops > 50% WoW |
| `creditBalanceLowPercent` | 20 | Flag if credits are below 20% remaining |

To change a threshold, edit the value in `campaign_agent_state.json` and commit. Henry will pick it up on the next run.

### Quarterly Goals

Goals are also in the state file under `goals`. When a new quarter starts, update:
- `quarter` — e.g., `"Q2_FY27"`
- `dateRange` — new start/end dates
- `totalMQLTarget` — new target
- `channelTargets` — new channel and CTA breakdowns

---

## Troubleshooting

### "Slack message posted as me, not the bot"

The skill used the MCP Slack tool instead of the bot script. Check that Step 7 in `skill.md` references `scripts/post-to-slack.sh`, not `mcp__...slack_send_message`.

### "Bot can't post to channel"

Make sure the bot is invited to #marketing-team. In Slack: `/invite @Henry - The Ad Performance Agent`

### "HTML report shows raw code on GitHub"

You're viewing the GitHub file page, not the GitHub Pages URL. Use:
```
https://af975.github.io/Ad-Peformance-Agent/reports/YYYY-MM-DD.html
```
not:
```
https://github.com/AF975/Ad-Peformance-Agent/blob/main/reports/YYYY-MM-DD.html
```

### "State file is corrupted"

Delete `campaign_agent_state.json` and run `/ad-report`. Henry will start fresh with default goals. You'll lose snapshot history but the next report will work.

### "Git push failed"

Henry will still complete the report and Slack post. The push failure is noted in the output. Check for authentication issues or network problems and push manually.

### "Metadata MCP returns empty data"

This usually means no experiments had spend in the requested date range. The report will note "No active experiments" for that channel. Check Metadata.io directly to confirm campaign status.

### "Experiment classified as 'other'"

The offer name didn't match any CTA pattern. Check the experiment in Metadata.io, identify the correct CTA type, and either:
- Add the pattern to the classification rules in `skill.md` (Step 4)
- Manually set the `cta` field in `experimentTracking.knownExperiments` in the state file
