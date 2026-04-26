# openclaw-sql-dreamer

> An OpenClaw skill that replaces file-based memory ingestion with SQL-backed storage, while preserving full dreaming pipeline compatibility.

---

## Why This Exists

OpenClaw's native `memory-core` dreaming system reads from `memory/YYYY-MM-DD.md` files — flat text logs that contain everything: debug noise, error traces, real decisions, and meaningful insights all mixed together. The dreamer can't distinguish signal from noise without a priority filter.

This skill solves that by:

1. **Feeding the dreamer from SQL** — queries your database for high-importance memories (configurable threshold) and writes a clean daily memory file before the dream cycle runs.
2. **Archiving dream outputs back to SQL** — after each dream cycle, structured dream data (light/REM/deep phases) is stored durably in SQL instead of living only as files.
3. **Keeping the filesystem lean** — dream output files older than N days are pruned automatically.
4. **Providing a stable foundation** for future wiki-to-Confluence publishing.

The OpenClaw dreamer still runs exactly as designed. This skill is a wrapper around it, not a replacement.

---

## Architecture

```
                    ┌─────────────────────────────────┐
                    │         SQL Database              │
                    │   memory.Memories (importance)    │
                    │   dreams.DreamCorpus              │
                    │   dreams.DreamLight               │
                    │   dreams.DreamREM                 │
                    │   dreams.DreamDeep                │
                    └────────┬─────────────┬────────────┘
                             │             │
                    [3:00 AM] │             │ [4:00 AM]
                    pre_dream │             │ post_dream
                    sql_feed  │             │ archiver
                             ▼             │
                    memory/YYYY-MM-DD.md   │
                    (clean, curated)        │
                             │             │
                    [3:30 AM]│             │
                    ┌────────▼─────────┐   │
                    │  OpenClaw native │   │
                    │  dream cycle     │   │
                    │  (unchanged)     │   │
                    └────────┬─────────┘   │
                             │             │
                    memory/dreaming/        │
                    ├── light/YYYY-MM-DD.md │
                    ├── rem/YYYY-MM-DD.md   │
                    └── deep/YYYY-MM-DD.md  │
                             └─────────────┘
                                           │
                                  [4:30 AM]│ (optional)
                                  confluence_publisher
                                           │
                                           ▼
                                  Confluence Memory Palace
```

---

## Prerequisites

- OpenClaw installed and configured with `memory-core` plugin
- SQL Server (or compatible) with pyodbc driver
- Python 3.10+
- `pyodbc` or `pymssql` installed
- `.env` file with database credentials (see Configuration)

---

## Quick Start

```bash
# 1. Clone the repo
git clone https://github.com/High-Falootin/openclaw-SQL-dreamer.git
cd openclaw-SQL-dreamer

# 2. Install dependencies
pip install -r requirements.txt

# 3. Copy and fill in config
cp config/example.yml config/config.yml
# Edit config/config.yml with your SQL credentials and settings

# 4. Run schema migration
python sql/migrate.py

# 5. Test the pre-dream feed
python scripts/pre_dream_sql_feed.py --dry-run

# 6. Add to crontab (adjust for your timezone)
# See: Crontab Integration section below
```

---

## Configuration

Copy `config/example.yml` and fill in your values. **Never commit `config/config.yml`** — it's in `.gitignore`.

```yaml
# config/example.yml — copy to config/config.yml and fill in values

sql:
  server: "your-sql-server-hostname"   # e.g. 10.0.0.110 or sql.example.com
  database: "Oblio_Memories"            # your database name
  username: "your_sql_username"         # SQL auth username
  password: ""                          # leave blank, set via env SQL_PASSWORD
  # Alternatively, use a full connection string:
  # connection_string: "DRIVER={ODBC Driver 17 for SQL Server};SERVER=...;"

corpus:
  importance_threshold: 7     # Min importance score to include in dream corpus (1-10)
  lookback_days: 2            # How many days back to pull memories

dreaming:
  workspace_dir: "/path/to/.openclaw/workspace"  # OpenClaw workspace root
  phases:
    light:
      enabled: true
    rem:
      enabled: true
    deep:
      enabled: true
  archive_after_days: 7       # Delete dream .md files older than this

confluence:
  enabled: false              # Set true to enable Confluence publishing
  domain: ""                  # e.g. yourorg.atlassian.net
  email: ""                   # Atlassian account email
  api_token: ""               # leave blank, set via env CONFLUENCE_API_TOKEN
  space_key: ""               # Confluence space key
  parent_page_id: ""          # Parent page ID for Memory Palace
```

**Environment variables (override config, never commit values):**

```bash
SQL_PASSWORD=your_sql_password
CONFLUENCE_API_TOKEN=your_token
```

---

## Crontab Integration

Add these entries to your crontab (`crontab -e`). Adjust timezone as needed (shown in EDT = UTC-4):

```cron
# Pre-dream SQL feed: populate clean memory file from SQL (30 min before dream cycle)
0 7 * * * /path/to/python /path/to/openclaw-SQL-dreamer/scripts/pre_dream_sql_feed.py >> /var/log/oblio/pre_dream.log 2>&1

# OpenClaw native dream cycle: 3:30 AM EDT (07:30 UTC) — configured in openclaw.json
# This runs automatically via OpenClaw's memory-core cron. No crontab entry needed.

# Post-dream archiver: archive outputs to SQL + cleanup (1 hr after dream cycle)
0 8 * * * /path/to/python /path/to/openclaw-SQL-dreamer/scripts/post_dream_archiver.py >> /var/log/oblio/post_dream.log 2>&1

# Confluence publisher (optional, runs after archiver)
30 8 * * * /path/to/python /path/to/openclaw-SQL-dreamer/scripts/confluence_dream_publisher.py >> /var/log/oblio/confluence.log 2>&1
```

---

## SQL Schema Overview

All tables live in the `dreams` schema. Run `sql/migrate.py` to create them.

| Table | Purpose |
|---|---|
| `dreams.DreamCorpus` | Curated memories queued for each dream cycle (filtered from `memory.Memories`) |
| `dreams.DreamLight` | Light sleep candidates and phase signal equivalents |
| `dreams.DreamREM` | REM sleep themes and pattern reflections |
| `dreams.DreamDeep` | Deep sleep promotions — durable recall entries |

---

## How the Three Dream Phases Work

OpenClaw's dreaming system runs three sequential phases each night:

### Light Sleep
Ingests recent daily memory files and session transcripts. Ranks entries by recency + recall frequency. Writes top N candidates as "Imported Insights" — things that surfaced recently and deserve attention.

**SQL mapping:** `dreams.DreamLight` stores each candidate with its confidence score, evidence source, recall count, and status (staged → promoted).

### REM Sleep
Processes short-term recall store. Finds patterns and themes across multiple sessions. Reflects on recurring concepts (what kept surfacing?). Synthesizes "Possible Lasting Truths."

**SQL mapping:** `dreams.DreamREM` stores theme entries with frequency counts, supporting evidence, and confidence levels.

### Deep Sleep
Promotes high-scoring short-term memories to durable (MEMORY.md / long-term). Uses weighted scoring: recall frequency × recency × query diversity. Only entries meeting minimum score + recall thresholds are promoted.

**SQL mapping:** `dreams.DreamDeep` stores promotions with scoring breakdown and promotion timestamp.

---

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Make changes with tests
4. Run `pytest tests/`
5. Open a PR against `development` branch

**Never commit secrets, credentials, or personal data.** This is a public repository.

---

## License

MIT — see [LICENSE](LICENSE)

---

## Related

- [OpenClaw](https://openclaw.ai) — the AI assistant platform this skill extends
- [ClawHub](https://clawhub.ai) — the OpenClaw skill registry where this skill will be published
