# HOW TO DREAM — openclaw-SQL-dreamer

> **The complete guide to understanding, operating, and troubleshooting the SQL Dreamer pipeline.**

---

## Table of Contents

1. [What Dreaming Is and Why It Exists](#1-what-dreaming-is-and-why-it-exists)
2. [The Full Pipeline](#2-the-full-pipeline)
3. [SQL Tables — Schema and Purpose](#3-sql-tables--schema-and-purpose)
4. [Configuration: sql_dreamer.yml](#4-configuration-sql_dreameryml)
5. [Noise Filtering — Why Not Every Memory Dreams](#5-noise-filtering--why-not-every-memory-dreams)
6. [Backfill — Retroactively Filling DreamCorpus](#6-backfill--retroactively-filling-dreamcorpus)
7. [Running a Dream Cycle Manually](#7-running-a-dream-cycle-manually)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. What Dreaming Is and Why It Exists

OpenClaw agents accumulate knowledge continuously — facts discovered, decisions made, incidents logged,
lessons learned. Over time this corpus grows large and noisy. Not everything stored in memory deserves
equal weight. Many observations are ephemeral, low-signal, or redundant.

**Dreaming is the consolidation pass.**

Inspired by how biological memory works during sleep, the dream cycle processes the raw stream of agent
memory and does three things:

- **Light Sleep** — Surfaces high-recall candidates: snippets that appeared frequently across multiple
  sessions and score above a confidence threshold. These are things the agent keeps "noticing."
- **REM Sleep** — Identifies cross-memory themes: patterns that emerge when multiple memories are
  considered together. The agent's subconscious makes connections.
- **Deep Sleep** — Promotes the most durable insights: the synthesis of what should survive long-term.
  Promoted entries may eventually be written back into persistent memory or published to a knowledge base.

Without a dream cycle, agents accumulate noise. With it, signal rises to the surface on a predictable
schedule and is stored in a queryable, durable SQL schema — not just transient markdown files.

### Why SQL?

OpenClaw's native dreamer reads from and writes to markdown files in the workspace
(`memory/YYYY-MM-DD.md`, `memory/dreaming/light/`, `memory/dreaming/rem/`, `memory/dreaming/deep/`).
These files are ephemeral by design. They disappear, get rewritten, and don't accumulate history.

The SQL Dreamer layer wraps that native pipeline with:

- **Pre-dream feed** — Curates SQL memory into the clean input file the native dreamer expects
- **Post-dream archiver** — Parses the native dreamer's markdown outputs and persists them to SQL tables
- **Phase signal reconciler** — Syncs OpenClaw's internal JSON state files to SQL for audit and replay
- **Light sleep synthesizer** — Optionally generates `DreamLight` entries directly from SQL + JSON,
  bypassing the markdown round-trip

The result: a complete audit trail of every dream cycle, queryable by date, filterable by confidence,
joinable to the original memories that seeded them.

---

## 2. The Full Pipeline

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         SQL DREAMER PIPELINE                                 │
│                                                                              │
│  [3:00 AM] pre_dream_sql_feed.py                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  1. Query memory.Memories WHERE importance >= threshold             │     │
│  │     AND created_at >= NOW() - lookback_days                         │     │
│  │  2. Group by category, sort by importance DESC                      │     │
│  │  3. Write clean memory/YYYY-MM-DD.md to workspace                  │     │
│  │  4. Record corpus batch → dreams.DreamCorpus (audit trail)          │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                              ↓                                               │
│  [3:30 AM] OpenClaw Native Dreamer (automatic, managed by OpenClaw)          │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Reads: memory/YYYY-MM-DD.md (curated by pre_dream_sql_feed)        │     │
│  │  Reads: memory/.dreams/short-term-recall.json (phase signals)       │     │
│  │  Reads: memory/.dreams/phase-signals.json                           │     │
│  │  Writes: memory/dreaming/light/YYYY-MM-DD.md                        │     │
│  │  Writes: memory/dreaming/rem/YYYY-MM-DD.md                          │     │
│  │  Writes: memory/dreaming/deep/YYYY-MM-DD.md                         │     │
│  │  Updates: memory/.dreams/phase-signals.json (phase state)           │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                              ↓                                               │
│  [4:00 AM] post_dream_archiver.py                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  1. Parse memory/dreaming/light/YYYY-MM-DD.md → DreamLight rows     │     │
│  │  2. Parse memory/dreaming/rem/YYYY-MM-DD.md → DreamREM rows         │     │
│  │  3. Parse memory/dreaming/deep/YYYY-MM-DD.md → DreamDeep rows       │     │
│  │  4. Insert structured rows into SQL                                 │     │
│  │  5. Delete dream MD files older than archive_after_days             │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                              ↓                                               │
│  [4:05 AM] phase_signal_reconciler.py                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  1. Read memory/.dreams/phase-signals.json                          │     │
│  │  2. UPSERT into dreams.PhaseSignals (idempotent)                    │     │
│  │  3. Sync daily-ingestion.json file tracker to DreamCorpus           │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  [Optional] light_sleep_synthesizer.py (SQL-native, bypasses native dreamer) │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  1. Read short-term-recall.json directly                            │     │
│  │  2. Score entries using OpenClaw's confidence formula               │     │
│  │  3. Write top N candidates to dreams.DreamLight (idempotent)        │     │
│  │  4. Also write memory/dreaming/light/YYYY-MM-DD.md (compat)         │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step: What Each Script Does

#### `pre_dream_sql_feed.py`

**Purpose:** Bridge between SQL Memory and the OpenClaw native dreamer.

The native dreamer reads a markdown file in `memory/YYYY-MM-DD.md`. Without intervention, that file
contains raw session notes — whatever the agent happened to write that day. The pre-dream feed
replaces that with a **curated, SQL-backed corpus** that only includes memories above the importance
threshold.

**What it produces:**
- `memory/YYYY-MM-DD.md` — clean, grouped-by-category, importance-sorted memory file
- Rows in `dreams.DreamCorpus` — audit trail of what was fed to each dream cycle

**Key flags:**
- `--dry-run` — Print what would be written without touching disk or SQL
- `--config /path/to/config.yml` — Specify config file location

---

#### OpenClaw Native Dreamer

This is OpenClaw's built-in dream engine. SQL Dreamer does not replace it — it wraps it.

The native dreamer reads the memory file prepared by the feed script and runs three phases:
- **Light sleep:** Scores short-term recall candidates by frequency and conceptual weight
- **REM sleep:** Identifies themes spanning multiple memories
- **Deep sleep:** Promotes candidates to long-term status based on a composite score

Output files land in `memory/dreaming/{light,rem,deep}/YYYY-MM-DD.md`.

---

#### `post_dream_archiver.py`

**Purpose:** Parse the native dreamer's markdown output and persist it to SQL.

Reads each phase file and inserts structured rows into the corresponding SQL table:
- `memory/dreaming/light/YYYY-MM-DD.md` → `dreams.DreamLight`
- `memory/dreaming/rem/YYYY-MM-DD.md` → `dreams.DreamREM`
- `memory/dreaming/deep/YYYY-MM-DD.md` → `dreams.DreamDeep`

Also runs cleanup: deletes `.md` files older than `archive_after_days` to prevent workspace bloat.
SQL records are permanent — the markdown is transient.

**Key flags:**
- `--date YYYY-MM-DD` — Archive a specific past date (defaults to today)
- `--dry-run` — Preview without writing
- `--config /path/to/config.yml`

---

#### `phase_signal_reconciler.py`

**Purpose:** Back up OpenClaw's internal JSON state to SQL.

OpenClaw maintains phase signal state in `memory/.dreams/phase-signals.json` and related files.
These JSON files are the live state the native dreamer reads and writes. They can be lost if the
workspace is rebuilt or the files are accidentally deleted.

The reconciler reads these files and UPSERTs into `dreams.PhaseSignals`. SQL becomes the durable
backup. The script is idempotent — safe to run multiple times.

**Key flags:**
- `--dry-run` — Show what would be upserted
- `--config /path/to/config.yml`

---

#### `light_sleep_synthesizer.py`

**Purpose:** SQL-native alternative to the native dreamer's light sleep phase.

Instead of waiting for the native dreamer to write a markdown file, this script reads
`short-term-recall.json` directly, scores entries using the same formula the native dreamer uses,
and writes results directly to `dreams.DreamLight`. It also writes the markdown for backward
compatibility.

Use this when you want SQL-first light sleep data, or when the native dreamer is unavailable.

**Confidence formula:**
```
avgScore       = totalScore / max(1, recallCount)
recallStrength = min(1, log1p(recallCount) / log1p(6))
consolidation  = min(1, len(recallDays) / 3)
conceptual     = 0.5 if len(conceptTags) > 0 else 0.0
confidence     = avgScore*0.45 + recallStrength*0.25 + consolidation*0.20 + conceptual*0.10
```

**Key flags:**
- `--date YYYY-MM-DD` — Run for a specific date
- `--dry-run` — Preview without writing
- `--config /path/to/config.yml`

---

## 3. SQL Tables — Schema and Purpose

All dream tables live in the `dreams` schema. Memory tables (read by the pre-dream feed) live in
the `memory` schema.

---

### `memory.Memories` (read-only for dreamer)

The source of truth for what gets dreamed about. The pre-dream feed queries this table.

```sql
CREATE TABLE memory.Memories (
    id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT newid(),
    category    NVARCHAR(100)   NOT NULL,      -- 'facts', 'decisions', 'incidents', 'lessons_learned'
    key         NVARCHAR(255)   NOT NULL,      -- Unique key within category
    content     NVARCHAR(MAX)   NOT NULL,      -- The memory text
    importance  INT             DEFAULT 3,     -- 1-10 scale; dreamer only reads >= threshold (default 7)
    tags        NVARCHAR(500)   DEFAULT '',    -- Optional comma-separated tags
    status      NVARCHAR(50)    DEFAULT 'active', -- 'active' or 'archived'
    created_at  DATETIME2       DEFAULT GETUTCDATE(),
    updated_at  DATETIME2       DEFAULT GETUTCDATE()
);
```

**Purpose in dreaming:** Pre-dream feed queries this with `importance >= threshold AND updated_at >= lookback`.
Only high-importance, recent memories enter the dream cycle.

---

### `dreams.DreamCorpus`

Audit table recording exactly which memories were fed into each dream cycle.

```sql
CREATE TABLE dreams.DreamCorpus (
    id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT newid(),
    dream_date  DATE            NOT NULL,      -- YYYY-MM-DD of the dream cycle
    memory_id   UNIQUEIDENTIFIER,              -- FK to memory.Memories
    importance  INT,                           -- Importance at time of ingestion
    source      NVARCHAR(100),                 -- 'memory' or 'session'
    created_at  DATETIME2       DEFAULT GETUTCDATE()
);
```

**Purpose:** Reproducibility and audit. You can always answer "what did Oblio dream about on 2026-04-25?"
by querying `DreamCorpus` joined to `Memories`.

---

### `dreams.DreamLight`

Light sleep phase results. High-recall, high-confidence memory candidates.

```sql
CREATE TABLE dreams.DreamLight (
    id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT newid(),
    dream_date   DATE            NOT NULL,     -- Dream cycle date
    snippet      NVARCHAR(MAX),               -- The candidate memory text
    confidence   FLOAT,                        -- 0.0-1.0 composite score
    recall_count INT,                          -- How many times this surfaced in recall
    status       NVARCHAR(50),                 -- 'staged', 'promoted'
    created_at   DATETIME2       DEFAULT GETUTCDATE()
);
```

**Purpose:** Light sleep candidates are the first filter. They surface memories that are frequently
recalled but not yet deeply consolidated. Rows here may be promoted to `DreamDeep` over subsequent
cycles.

---

### `dreams.DreamREM`

REM sleep phase results. Cross-memory themes and lasting truths.

```sql
CREATE TABLE dreams.DreamREM (
    id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT newid(),
    dream_date  DATE            NOT NULL,     -- Dream cycle date
    theme_text  NVARCHAR(MAX),               -- The identified theme
    frequency   INT,                          -- How many memories surfaced this theme
    confidence  FLOAT,                        -- 0.0-1.0 confidence score
    created_at  DATETIME2       DEFAULT GETUTCDATE()
);
```

**Purpose:** REM themes are the connective tissue — patterns the agent notices across multiple
unrelated memories. High-frequency, high-confidence themes are candidates for synthesis into
new facts or decisions.

---

### `dreams.DreamDeep`

Deep sleep phase results. Promoted, durable memories.

```sql
CREATE TABLE dreams.DreamDeep (
    id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT newid(),
    dream_date  DATE            NOT NULL,     -- Dream cycle date
    snippet     NVARCHAR(MAX),               -- The promoted memory content
    ranked      INT,                          -- Rank by composite score (lower = better)
    promoted    BIT,                          -- Was this entry promoted to long-term?
    created_at  DATETIME2       DEFAULT GETUTCDATE()
);
```

**Purpose:** The most valuable output of the dream cycle. Deep sleep promotions represent memories
that have been consolidated across multiple light/REM cycles and are now durable enough to be
treated as long-term knowledge. `promoted = 1` entries are candidates for Confluence publishing
or re-ingestion into `memory.Memories` at elevated importance.

---

### `dreams.PhaseSignals`

Durable SQL backup of OpenClaw's internal phase signal JSON state.

```sql
CREATE TABLE dreams.PhaseSignals (
    id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT newid(),
    signal_key   NVARCHAR(500)   NOT NULL UNIQUE, -- Key from phase-signals.json
    light_hits   INT             DEFAULT 0,        -- Times surfaced in light sleep
    rem_hits     INT             DEFAULT 0,        -- Times surfaced in REM sleep
    last_light_at DATETIME2,                       -- Last light sleep appearance
    last_rem_at   DATETIME2,                       -- Last REM sleep appearance
    updated_at    DATETIME2      DEFAULT GETUTCDATE()
);
```

**Purpose:** The native dreamer's JSON state is ephemeral. This table is the SQL mirror. If the
workspace is rebuilt or the JSON files are lost, the reconciler can bootstrap from here (or at
minimum, history is preserved for analysis).

---

## 4. Configuration: sql_dreamer.yml

Located at: `config/sql_dreamer.yml` (or `config/config.yml` depending on install path).

```yaml
sql:
  backend: "cloud"           # "cloud" or "local" — matches SQL_DEFAULT_BACKEND in .env
                             # This tells the connector which credentials block to use.
                             # Actual credentials live in .env, never in this file.

corpus:
  importance_threshold: 7   # Integer 1-10. Only memories with importance >= this value
                             # are included in the dream corpus. 7 is the recommended
                             # default; lower it to get more dreams, raise it for higher
                             # signal-to-noise ratio.
  lookback_days: 2           # How many days back to query memory.Memories.
                             # 2 = memories updated in the last 2 days. Increase to
                             # dream on older memories; decrease to focus on recency.

dreaming:
  workspace_dir: "/home/username/.openclaw/workspace"
                             # REQUIRED. Absolute path to your OpenClaw workspace.
                             # Scripts write memory/YYYY-MM-DD.md and read
                             # memory/dreaming/* relative to this path.
                             # Run `openclaw status` to confirm yours.
  phases:
    light:
      enabled: true          # Run light sleep phase (surfacing recall candidates)
    rem:
      enabled: true          # Run REM phase (theme identification)
    deep:
      enabled: true          # Run deep sleep phase (promotion pass)
  archive_after_days: 7      # Dream .md files older than this are deleted by
                             # post_dream_archiver. SQL records are never deleted.
                             # 7 days is a good default.

light_sleep:
  top_candidates: 50         # light_sleep_synthesizer only: how many top-scored
                             # candidates to write to DreamLight per cycle.

confluence:
  enabled: false             # Set true to publish dream summaries to Confluence.
                             # If false, all other confluence keys are ignored.
  domain: "yourorg.atlassian.net"
  email: "your@email.com"
  api_token: "xxxxx"         # From Atlassian account settings → API tokens.
                             # Also readable from CONFLUENCE_API_TOKEN env var.
  space_key: "YOUR_SPACE"    # Confluence space ID (the key, not the name).
  parent_page_id: "12345"    # Numeric ID of the parent page for dream reports.
                             # Find it in the page URL: /pages/{id}/...
```

### Config File Location

Scripts auto-discover the config by walking up from the current directory looking for
`config/config.yml`. You can override with `--config /absolute/path/to/config.yml`.

---

## 5. Noise Filtering — Why Not Every Memory Dreams

Not every memory stored by agents is worth dreaming about. The dreamer applies two filtering layers:

### Layer 1: Importance Threshold

The `importance_threshold` in `sql_dreamer.yml` (default: **7**) is the primary gate.

When agents store memories using `SQLMemoryConnector.remember()`, they assign an importance score
from 1–10. The pre-dream feed only queries memories where `importance >= threshold`:

```sql
SELECT * FROM memory.Memories
WHERE importance >= 7        -- configured threshold
  AND updated_at >= DATEADD(day, -2, GETUTCDATE())  -- configured lookback
  AND status = 'active'
ORDER BY importance DESC, updated_at DESC
```

**Importance scale guidance:**

| Score | Meaning | Dream? |
|-------|---------|--------|
| 1–3   | Noise, logging, minor observations | No |
| 4–6   | Routine facts, standard operations | No (below default threshold) |
| 7–8   | Important findings, notable patterns | **Yes** |
| 9–10  | Critical facts, key decisions | **Yes** |

The threshold is intentionally conservative. 70% of typical agent memories are routine churn.
Only the top tier should influence the dream cycle.

### Layer 2: Stopword Filtering (light_sleep_synthesizer)

For the SQL-native light sleep synthesizer, a secondary filter removes entries whose snippet
is composed entirely of common stopwords (`the`, `a`, `and`, `or`, etc.) or is fewer than 3 characters.

This catches recall fragments that survived the importance threshold but carry no semantic content.

```python
STOPWORD_ONLY_PATTERNS = {
    "the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for",
    "of", "is", "are", "was", "were", "be", "been", ...
}

def is_stopword_only(text: str) -> bool:
    words = text.lower().split()
    return all(w in STOPWORD_ONLY_PATTERNS for w in words)
```

### Layer 3: Recency Filter (light_sleep_synthesizer)

Recall entries in `short-term-recall.json` are only included if at least one `recallDay`
falls within the `lookback_days` window. Entries not recalled recently are excluded even if
their scores are high — they had their moment; they'll return when the agent re-encounters them.

---

## 6. Backfill — Retroactively Filling DreamCorpus

If you've been running `memory.Memories` for weeks before installing SQL Dreamer, the
`dreams.DreamCorpus` table will be empty — it only records what the pre-dream feed actively sent.

To retroactively fill `DreamCorpus` with existing memories:

### Option A: Backfill Script (if available)

```bash
python3 scripts/backfill_corpus.py --from 2026-01-01 --to 2026-04-28
```

### Option B: Manual SQL Backfill

Insert historical memories directly into `DreamCorpus`. This marks them as retrospectively
eligible for the dream corpus without re-running the dream cycle itself:

```sql
INSERT INTO dreams.DreamCorpus (dream_date, memory_id, importance, source, created_at)
SELECT
    CAST(m.updated_at AS DATE)   AS dream_date,
    m.id                         AS memory_id,
    m.importance,
    'memory'                     AS source,
    GETUTCDATE()                 AS created_at
FROM memory.Memories m
WHERE m.importance >= 7
  AND m.updated_at >= '2026-01-01'
  AND NOT EXISTS (
      SELECT 1 FROM dreams.DreamCorpus dc
      WHERE dc.memory_id = m.id
  );
```

### Option C: Re-run the Pre-Dream Feed for a Past Date

The pre-dream feed accepts no `--date` flag; it always runs for today. To backfill a specific
date, temporarily override the date in the script or adjust `lookback_days` to cover the range
you want, run the feed, then restore.

### After Backfill

The `DreamCorpus` entries created by backfill won't have corresponding `DreamLight`, `DreamREM`,
or `DreamDeep` entries unless you also re-run the archiver for those dates:

```bash
python3 scripts/post_dream_archiver.py --date 2026-04-25
python3 scripts/post_dream_archiver.py --date 2026-04-24
# etc.
```

This will attempt to parse whatever dream markdown files exist for those dates. If the files
were deleted (they expire after `archive_after_days`), the archiver will warn and skip missing
phases gracefully.

---

## 7. Running a Dream Cycle Manually

Use this when you want to test the pipeline, force a dream outside the cron schedule, or
re-process a specific date.

### Full Manual Run

```bash
# Step 1: Pre-dream feed — curate SQL memories into workspace file
python3 scripts/pre_dream_sql_feed.py --config config/config.yml

# Step 2: Trigger the native dreamer (OpenClaw manages this)
# In most setups this is automatic. To force it manually:
openclaw dream --date $(date +%Y-%m-%d)

# Step 3: Post-dream archiver — parse outputs to SQL
python3 scripts/post_dream_archiver.py --config config/config.yml

# Step 4: Phase signal reconciler — sync JSON state to SQL
python3 scripts/phase_signal_reconciler.py --config config/config.yml
```

### Dry Run (Preview Without Writing)

All scripts support `--dry-run`. Run all stages in dry-run mode to verify what would happen:

```bash
python3 scripts/pre_dream_sql_feed.py --dry-run
python3 scripts/post_dream_archiver.py --dry-run
python3 scripts/phase_signal_reconciler.py --dry-run
python3 scripts/light_sleep_synthesizer.py --dry-run
```

### Run a Single Phase

```bash
# Just the pre-feed (no native dreamer, no archiver)
python3 scripts/pre_dream_sql_feed.py

# Just the archiver for a specific past date
python3 scripts/post_dream_archiver.py --date 2026-04-25

# Just the light sleep synthesizer (SQL-native, bypasses native dreamer)
python3 scripts/light_sleep_synthesizer.py --date 2026-04-25
```

### Verify Results

After running, check what was written:

```python
from src.sql_connector import SQLDreamerConnector

with SQLDreamerConnector.from_config("config/config.yml") as conn:
    # Check corpus
    corpus = conn.query("""
        SELECT COUNT(*) AS cnt FROM dreams.DreamCorpus
        WHERE dream_date = CAST(GETDATE() AS DATE)
    """)
    print(f"Corpus entries today: {corpus[0]['cnt']}")

    # Check light sleep
    light = conn.query("""
        SELECT snippet, confidence
        FROM dreams.DreamLight
        WHERE dream_date = CAST(GETDATE() AS DATE)
        ORDER BY confidence DESC
    """)
    for row in light:
        print(f"[{row['confidence']:.2f}] {row['snippet'][:80]}")
```

### Crontab Reference

The standard cron schedule (all times local to the host):

```cron
# Pre-dream: 3:00 AM
0 3 * * * /usr/bin/python3 /path/to/scripts/pre_dream_sql_feed.py >> ~/.openclaw/logs/pre_dream.log 2>&1

# Post-dream: 4:00 AM (1 hour after native dreamer at 3:30 AM)
0 4 * * * /usr/bin/python3 /path/to/scripts/post_dream_archiver.py >> ~/.openclaw/logs/post_dream.log 2>&1

# Phase signal reconciler: 4:05 AM (after archiver)
5 4 * * * /usr/bin/python3 /path/to/scripts/phase_signal_reconciler.py >> ~/.openclaw/logs/reconciler.log 2>&1

# Optional: Confluence publisher 4:30 AM
# 30 4 * * * /usr/bin/python3 /path/to/scripts/confluence_dream_publisher.py >> ~/.openclaw/logs/confluence.log 2>&1
```

---

## 8. Troubleshooting

### Dreams Are Always Empty

**Symptoms:** `DreamLight`, `DreamREM`, `DreamDeep` all have zero rows for today.

**Diagnosis:**

```bash
# 1. Check pre-dream feed output
tail -50 ~/.openclaw/logs/pre_dream.log

# 2. Confirm memories exist above threshold
python3 -c "
from src.sql_connector import SQLDreamerConnector
with SQLDreamerConnector.from_config('config/config.yml') as conn:
    rows = conn.query('SELECT COUNT(*) AS cnt FROM memory.Memories WHERE importance >= 7')
    print('Memories above threshold:', rows[0]['cnt'])
"

# 3. Check the generated memory file
cat memory/$(date +%Y-%m-%d).md
```

**Fixes:**
- If `cnt = 0`: Lower `importance_threshold` in config, or store higher-importance memories
- If memory file is empty: The pre-dream feed found nothing — check `lookback_days` (increase it)
- If memory file exists but dreams are empty: Native dreamer didn't run — check OpenClaw logs

---

### "No memories above threshold — skipping write"

The pre-dream feed ran but found no qualifying memories.

**Check:**
1. Do you have memories in the lookback window?

   ```sql
   SELECT COUNT(*) FROM memory.Memories
   WHERE importance >= 7
     AND updated_at >= DATEADD(day, -2, GETUTCDATE());
   ```

2. Is `lookback_days` too small? Increase to `7` or `14` temporarily.
3. Is `importance_threshold` too high? Lower from `7` to `5` to test.

---

### SQL Tables Not Found

**Symptoms:** `Invalid object name 'dreams.DreamCorpus'` or similar.

**Fix:**

```bash
# Create dream schema tables
python3 sql/migrate.py

# Verify tables exist
python3 -c "
from src.sql_connector import SQLDreamerConnector
with SQLDreamerConnector.from_config('config/config.yml') as conn:
    tables = conn.query(\"SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='dreams'\")
    for t in tables:
        print(t['TABLE_NAME'])
"
```

---

### post_dream_archiver Skips All Phases

**Symptoms:** `⚠️ light/YYYY-MM-DD.md not found — skipping`

The native dreamer either didn't run, or already cleaned up the files.

**Fix:**
- Check if the native dreamer ran: look for `memory/dreaming/` files
- If files exist but for a different date, use `--date YYYY-MM-DD` to target the correct date
- If files are gone (expired), reduce `archive_after_days` or schedule the archiver to run sooner

---

### Phase Signal Reconciler Finds No Signals

**Symptoms:** `⚠️ No phase signals to sync — dream cycle may not have run yet`

**Check:**
```bash
ls -la memory/.dreams/
cat memory/.dreams/phase-signals.json
```

If `phase-signals.json` is missing, the native dreamer hasn't completed a full cycle yet.
Run at least one full dream cycle and retry.

---

### Connection Refused / Cannot Connect to SQL Server

```bash
# Test SQL connection
python3 -c "
from src.sql_connector import SQLDreamerConnector
with SQLDreamerConnector.from_config('config/config.yml') as conn:
    print('Connected' if conn.ping() else 'Failed')
"
```

**Check:**
- `.env` credentials match the `sql.backend` in config (`cloud` or `local`)
- SQL Server is running and reachable on port 1433
- Firewall allows outbound port 1433 from this host

---

### High-Importance Memories Not Appearing in Dreams

**Check that `status = 'active'`:**

```sql
SELECT key, importance, status
FROM memory.Memories
WHERE importance >= 7
ORDER BY importance DESC;
```

Memories with `status = 'archived'` are excluded. If you archived memories you want to dream on,
update their status:

```sql
UPDATE memory.Memories SET status = 'active' WHERE key = 'your_key';
```

---

### Log Reference

| Log File | Script |
|----------|--------|
| `~/.openclaw/logs/pre_dream.log` | pre_dream_sql_feed.py |
| `~/.openclaw/logs/post_dream.log` | post_dream_archiver.py |
| `~/.openclaw/logs/reconciler.log` | phase_signal_reconciler.py |

All scripts print progress to stdout with `✅` for success and `⚠️` / `❌` for warnings/errors.
In crontab, these go to the log file. When running manually, they print to terminal.

---

*Last updated: 2026-04-28 | openclaw-SQL-dreamer*
