# ⛏️ Internal Currency System — MiningOS

A multi-agent Python pipeline that converts raw **ActivityWatch** screen-time exports into a points-based internal currency for students, paired with a **MiningOS** dark-themed web dashboard for live leaderboard and analytics.

---

## Overview

The system tracks student activity on their machines via [ActivityWatch](https://activitywatch.net/), parses those exports weekly, scores each student on attendance, active hours, and behaviour, then pushes the results to a **Nucleus API** and renders them in a browser dashboard.

---

## Architecture

```
ActivityWatch JSON exports
        │
        ▼
┌──────────────────────────────────────────────────────┐
│                  pipeline.py (CLI)                   │
│                                                      │
│  Step 1 │ parser_agent      → ActivityRecord objects │
│  Step 2 │ base_points_agent → StudentScore (base)    │
│  Step 3 │ bonus_points_agent→ behavioural bonuses    │
│  Step 4 │ attendance_agent  → attendance rules       │
│  Step 5 │ aggregator_agent  → JSON output            │
│  Step 6 │ nucleus_push_agent→ POST to Nucleus API    │
└──────────────────────────────────────────────────────┘
        │
        ▼
  MiningOS Dashboard (index.html + app.js + styles.css)
  ├── Leaderboard table (rank, active hours, total pts)
  ├── Points Breakdown stacked bar chart (Chart.js)
  ├── Weekly Trend cumulative line chart
  └── Bonus & Exemption Summary panel
```

---

## Agents

| Agent | File | Responsibility |
|---|---|---|
| Parser | `agents/parser_agent.py` | Wraps `aw_raw_parser`; produces `ActivityRecord` objects with attendance status and bonus flags |
| Base Points | `agents/base_points_agent.py` | Computes base score from active hours per day |
| Bonus Points | `agents/bonus_points_agent.py` | Awards behavioural bonuses (helped others, self-learning, early/on-time project) |
| Attendance | `agents/attendance_agent.py` | Applies attendance multipliers; handles medical exemptions from `medical_exceptions.json` |
| Aggregator | `agents/aggregator_agent.py` | Collects all `StudentScore` objects and writes weekly JSON output |
| Nucleus Push | `agents/nucleus_push_agent.py` | POSTs aggregated scores to the Nucleus API endpoint |

---

## Scoring Model

Each student earns points per day/week based on:

- **Base points** — derived from active hours recorded by ActivityWatch
- **Attendance bonus** — awarded for being present
- **Activity bonus** — for sustained engagement levels
- **Behavioural bonus** — flags set in `bonus_input.json`:
  - `helped_others`
  - `self_learning`
  - `project_on_time`
  - `project_early`
- **Penalty** — applied for flagged apps (distracting/disallowed applications)
- **Medical Exempt** — students listed in `medical_exceptions.json` receive 0 pts and 0 penalty for the exempted date

Grading thresholds and minimum hours for attendance are configured in `config/scoring_config.json`.

---

## Dashboard (MiningOS)

A self-contained frontend (`index.html` / `app.js` / `styles.css`) that:

- Accepts an ActivityWatch JSON export via **drag-and-drop or file picker**
- POSTs it to `/upload` with a week-ending date (auto-set to next Friday)
- Polls `/results` and renders:
  - **Stat cards** — total students, points distributed, weeks processed, top earner
  - **Leaderboard** — ranked by total points, with medal icons and status badges (Active / Moderate / Low / Exempt)
  - **Points Breakdown chart** — stacked bar per student for the latest week
  - **Weekly Trend chart** — cumulative points per student across all weeks
  - **Bonus & Exemption Summary** — behavioural bonus earners and medical exemptions for the latest week
  - **Student Modal** — click any row for a full week-by-week breakdown

**Theme:** Dark (`#0D1117` background, `#1D9E75` accent green), Inter font, Chart.js v4.

---

## Usage

### Run the pipeline

```bash
# Single file
python pipeline.py --week 2026-05-09 --input data/input/aw-export-student.json

# Entire folder of exports
python pipeline.py --week 2026-05-09 --folder data/input/
```

### Input files

| File | Purpose |
|---|---|
| `data/input/aw-buckets-export_<name>.json` | Raw ActivityWatch exports per student |
| `data/input/bonus_input.json` | Per-student behavioural bonus flags |
| `data/input/medical_exceptions.json` | `{ "exceptions": [{ "hostname": "...", "date": "YYYY-MM-DD" }] }` |
| `config/scoring_config.json` | Scoring thresholds and `min_hours_for_attendance` |

### Serve the dashboard

Any static file server works:

```bash
python -m http.server 8080
# open http://localhost:8080
```

Or run the backend (if a Flask/FastAPI server is present) that handles `/upload` and `/results`.

---

## Project Structure

```
Internal-Currency-System/
├── pipeline.py                  # CLI entry point
├── models.py                    # ActivityRecord, StudentScore, AbsenceType
├── agents/
│   ├── aw_raw_parser.py         # Parses raw AW JSON → daily summaries
│   ├── parser_agent.py          # Wraps parser; applies attendance & bonus flags
│   ├── base_points_agent.py     # Base score calculation
│   ├── bonus_points_agent.py    # Behavioural bonus application
│   ├── attendance_agent.py      # Attendance rule enforcement
│   ├── aggregator_agent.py      # Aggregates & writes output JSON
│   └── nucleus_push_agent.py    # Pushes to Nucleus API
├── config/
│   └── scoring_config.json      # Thresholds, min hours, grade bands
├── data/
│   ├── input/                   # AW exports + bonus_input.json + medical_exceptions.json
│   └── output/                  # Generated scores_<week>.json files
├── index.html                   # MiningOS dashboard
├── app.js                       # Dashboard logic (Chart.js, fetch, modals)
├── styles.css                   # Dark theme stylesheet
└── run.bat                      # Windows convenience runner
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Pipeline | Python 3.11, `pathlib`, `argparse`, `json` |
| Activity data | ActivityWatch (open-source time tracker) |
| Dashboard | Vanilla JS, HTML5, CSS3, Chart.js v4 |
| Font | Google Fonts — Inter |
| API target | Nucleus (internal REST API) |

---

## Sample Data

The repo includes sample ActivityWatch exports for four students:
- `aw-buckets-export_supreeth.json`
- `aw-buckets-export_karanpillai.json`
- `aw-buckets-export_rohangupta.json`
- `aw-buckets-export_divyanair.json`

And a `scores_2026-05-15.json` output file showing what a processed result looks like.
