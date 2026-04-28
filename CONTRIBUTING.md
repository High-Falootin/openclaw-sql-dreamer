# CONTRIBUTING — openclaw-SQL-dreamer

> Guidelines for development, testing, and submitting changes to the SQL Dreamer pipeline.

---

## Table of Contents

1. [Development Setup](#1-development-setup)
2. [Branch and Commit Conventions](#2-branch-and-commit-conventions)
3. [Testing Lifecycle Rule](#3-testing-lifecycle-rule)
4. [How to Run Tests](#4-how-to-run-tests)
5. [PR Checklist](#5-pr-checklist)

---

## 1. Development Setup

### Prerequisites

- Python 3.9+
- SQL Server (local or cloud) with `memory` and `dreams` schemas created
- OpenClaw workspace (for integration tests that touch `memory/` files)
- `pip` or a virtual environment manager

### Clone and Install

```bash
git clone https://github.com/High-Falootin/openclaw-sql-dreamer.git
cd openclaw-sql-dreamer

# Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate       # Linux/macOS
# .venv\Scripts\activate        # Windows

# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt   # Test tools: pytest, pytest-cov, etc.
```

### Configure

```bash
# Copy example config
cp config/example.yml config/config.yml

# Edit with your workspace path and SQL credentials
nano config/config.yml

# Set up SQL credentials in .env (project root or workspace root)
cp .env.example .env
nano .env
```

Minimum config for development:

```yaml
sql:
  backend: "local"            # Use local SQL Server for dev

corpus:
  importance_threshold: 5     # Lower threshold for dev/testing
  lookback_days: 7            # Broader window for test data

dreaming:
  workspace_dir: "/home/you/.openclaw/workspace"
  phases:
    light:
      enabled: true
    rem:
      enabled: true
    deep:
      enabled: true
  archive_after_days: 3       # Clean up faster in dev

confluence:
  enabled: false              # Never publish from dev environments
```

### Create Database Schema

```bash
# Create dreams.* tables
python3 sql/migrate.py

# Verify
python3 -c "
from src.sql_connector import SQLDreamerConnector
with SQLDreamerConnector.from_config('config/config.yml') as conn:
    tables = conn.query(\"SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='dreams'\")
    for t in tables: print(t['TABLE_NAME'])
"
```

### Verify Your Setup

```bash
python3 scripts/pre_dream_sql_feed.py --dry-run
python3 scripts/post_dream_archiver.py --dry-run
python3 scripts/phase_signal_reconciler.py --dry-run
```

All should complete without errors (empty results are fine — the pipeline is wired up).

---

## 2. Branch and Commit Conventions

### Branch Naming

Branch names **must** include the Jira ticket ID. This is required for smart commit detection —
Jira links the PR to the ticket automatically.

**Format:**

```
{PROJECT}-{TICKET_ID}
{PROJECT}-{TICKET_ID}/{short-description}
```

**Examples:**

```bash
git checkout -b HFTC-33
git checkout -b HFTC-33/how-to-dream-docs
git checkout -b HFTC-36/contributing-testing-lifecycle
git checkout -b OB-25/add-backfill-script
```

The description suffix is optional but recommended for anything beyond a trivial change.

### Commit Messages

```
{PROJECT}-{TICKET_ID}: <imperative summary>

Optional body explaining why, not what. The diff shows what.
```

**Examples:**

```
HFTC-33: add HOW_TO_DREAM.md with full pipeline documentation

HFTC-36: enforce test lifecycle rule — delete output before asserting

OB-25: add backfill_corpus.py for retroactive DreamCorpus population
```

**Rules:**
- Use imperative mood: "add", "fix", "remove" — not "added", "fixes", "removing"
- Keep the summary under 72 characters
- Reference the ticket in every commit on a feature branch
- Do not use `WIP:` commits on PRs — squash or rebase before opening

---

## 3. Testing Lifecycle Rule

> **This is the core convention introduced in HFTC-36. Read it carefully.**

### The Problem

Several scripts in this repo generate output files:

- `pre_dream_sql_feed.py` writes `memory/YYYY-MM-DD.md`
- `post_dream_archiver.py` reads `memory/dreaming/{light,rem,deep}/YYYY-MM-DD.md`
- `light_sleep_synthesizer.py` writes `memory/dreaming/light/YYYY-MM-DD.md`

If a test asserts that a file exists and contains correct content, it may pass not because the
feature works, but because a **previous run** left a file behind. This produces false confidence.
Stale files from different configurations, different dates, or different code versions can silently
corrupt test results.

### The Rule

> **When a test covers a feature that generates files, the test MUST:**
>
> 1. **Delete** the output directory or files **before** running the feature
> 2. **Run** the feature
> 3. **Assert** all expected files exist with correct content
> 4. **Never** rely on pre-existing files — assume they are stale

This rule applies to:
- Any script that writes to `memory/` subdirectories
- Any script that writes config outputs
- Any script that creates `.md` dream files

### Correct Pattern

```python
import shutil
import pytest
from pathlib import Path

def test_post_dream_archiver_creates_light_entries(tmp_workspace, config_path):
    """
    post_dream_archiver should parse light sleep markdown and insert SQL rows.
    """
    dreaming_dir = tmp_workspace / "memory" / "dreaming"

    # --- Step 1: DELETE output directory before running ---
    if dreaming_dir.exists():
        shutil.rmtree(dreaming_dir)
    assert not dreaming_dir.exists(), "Output dir must not exist before test"

    # Set up: place input fixture where the archiver expects it
    light_dir = dreaming_dir / "light"
    light_dir.mkdir(parents=True)
    fixture = light_dir / "2026-04-25.md"
    fixture.write_text(LIGHT_SLEEP_FIXTURE)

    # --- Step 2: RUN the feature ---
    from scripts.post_dream_archiver import run
    run(config_path=str(config_path), target_date="2026-04-25", dry_run=False)

    # --- Step 3: ASSERT expected files and SQL rows exist ---
    # (The archiver consumes the file; verify SQL state)
    with SQLDreamerConnector.from_config(str(config_path)) as conn:
        rows = conn.query(
            "SELECT COUNT(*) AS cnt FROM dreams.DreamLight WHERE dream_date = '2026-04-25'"
        )
        assert rows[0]["cnt"] > 0, "Expected DreamLight rows to be inserted"
```

### Anti-Pattern (Do Not Do This)

```python
# ❌ WRONG — does not delete output first
def test_pre_dream_feed_writes_file(config_path, workspace_dir):
    from scripts.pre_dream_sql_feed import run
    run(config_path=str(config_path))

    output = workspace_dir / "memory" / "2026-04-25.md"
    assert output.exists()  # May pass because a previous run left this file
    assert "## Facts" in output.read_text()  # May read stale content
```

### Pattern for Pre-Dream Feed Tests

```python
def test_pre_dream_feed_writes_memory_file(tmp_workspace, config_path):
    today = "2026-04-25"
    output_file = tmp_workspace / "memory" / f"{today}.md"

    # Step 1: Delete any existing output file
    if output_file.exists():
        output_file.unlink()
    assert not output_file.exists()

    # Step 2: Seed SQL with test memories above threshold
    seed_test_memories(importance=8)

    # Step 3: Run the feature
    from scripts.pre_dream_sql_feed import run
    run(config_path=str(config_path))

    # Step 4: Assert file was created with correct content
    assert output_file.exists(), "Memory file must be created by pre_dream_sql_feed"
    content = output_file.read_text()
    assert "# Session Memory" in content
    assert "## Facts" in content
```

### Pattern for light_sleep_synthesizer Tests

```python
def test_light_sleep_synthesizer_creates_md_and_sql(tmp_workspace, config_path, seed_recall_json):
    cycle_date = "2026-04-25"
    output_file = tmp_workspace / "memory" / "dreaming" / "light" / f"{cycle_date}.md"

    # Step 1: Delete existing output
    if output_file.parent.exists():
        shutil.rmtree(output_file.parent)
    assert not output_file.exists()

    # Step 2: Run the synthesizer
    from scripts.light_sleep_synthesizer import run
    run(config_path=str(config_path), cycle_date=cycle_date)

    # Step 3: Assert .md file created
    assert output_file.exists()
    content = output_file.read_text()
    assert "# Light Sleep" in content
    assert "- Candidate:" in content

    # Step 4: Assert SQL rows created
    with SQLDreamerConnector.from_config(str(config_path)) as conn:
        rows = conn.query(
            "SELECT COUNT(*) AS cnt FROM dreams.DreamLight WHERE dream_date = ?",
            (cycle_date,)
        )
        assert rows[0]["cnt"] > 0
```

### SQL Cleanup in Tests

Tests that insert SQL rows must clean up after themselves. Use fixtures or teardown:

```python
@pytest.fixture(autouse=True)
def clean_dream_tables(config_path):
    """Delete SQL rows for test dates before and after each test."""
    test_dates = ["2026-04-25", "2026-04-26"]
    yield
    with SQLDreamerConnector.from_config(str(config_path)) as conn:
        for date in test_dates:
            for table in ["DreamLight", "DreamREM", "DreamDeep", "DreamCorpus"]:
                conn.execute(f"DELETE FROM dreams.{table} WHERE dream_date = ?", (date,))
```

---

## 4. How to Run Tests

### Run All Tests

```bash
pytest tests/ -v
```

### Run a Specific Test File

```bash
pytest tests/test_pre_dream_sql_feed.py -v
pytest tests/test_post_dream_archiver.py -v
pytest tests/test_light_sleep_synthesizer.py -v
pytest tests/test_phase_signal_reconciler.py -v
```

### Run with Coverage

```bash
pytest tests/ --cov=scripts --cov=src --cov-report=term-missing
```

### Run Only Unit Tests (No SQL Required)

Tests that require a live SQL connection are marked `@pytest.mark.integration`.
To skip them:

```bash
pytest tests/ -v -m "not integration"
```

### Run Only Integration Tests

```bash
pytest tests/ -v -m "integration"
```

### Dry Run Tests

For scripts that support `--dry-run`, there are corresponding test modes that don't
require a database connection:

```bash
pytest tests/ -v -m "dryrun"
```

### Environment Variables for Tests

```bash
# Use a test-specific config
export SQL_DREAMER_TEST_CONFIG=config/test-config.yml

# Use a temporary workspace
export SQL_DREAMER_TEST_WORKSPACE=/tmp/openclaw-test-workspace

pytest tests/ -v
```

---

## 5. PR Checklist

Before opening a pull request, verify all of the following:

### Code Quality

- [ ] All new code has docstrings (module-level + function-level)
- [ ] No hardcoded paths — use `workspace_dir` from config
- [ ] No hardcoded credentials — use `.env` + connector
- [ ] `--dry-run` flag implemented for any script that writes files or SQL rows
- [ ] `--config` flag implemented for any script that reads config

### Testing

- [ ] Tests written for all new functionality
- [ ] **Testing lifecycle rule followed:** output deleted before running feature, then asserted after
- [ ] No tests that pass only because a pre-existing file was present
- [ ] SQL cleanup fixtures in place (no test data left in `dreams.*` tables)
- [ ] Tests pass locally: `pytest tests/ -v`
- [ ] Integration tests pass against a real SQL connection: `pytest tests/ -v -m integration`

### Documentation

- [ ] `HOW_TO_DREAM.md` updated if pipeline behavior changes
- [ ] `GETTING_STARTED.md` updated if setup steps change
- [ ] `SKILL_REFERENCE.md` updated if schema or config fields change
- [ ] Config examples in `config/example.yml` reflect any new fields

### Branch and Commit

- [ ] Branch name includes Jira ticket ID: `HFTC-XX` or `OB-XX`
- [ ] Commits reference ticket ID in message: `HFTC-XX: description`
- [ ] No `WIP:` commits — squash or rebase before opening PR
- [ ] Branch is up to date with `main` (rebase, don't merge)

### Safety

- [ ] `confluence.enabled: false` in any test configs committed to the repo
- [ ] No credentials, tokens, or API keys in committed files
- [ ] Destructive operations (file deletes, SQL deletes) are guarded by `dry_run` flag
- [ ] Scripts fail gracefully when input files are missing (warn and skip, don't crash)

### Review

- [ ] Self-review: read your own diff as if you were the reviewer
- [ ] PR description includes: what changed, why, how to test
- [ ] Link to Jira ticket in PR description

---

*Last updated: 2026-04-28 | openclaw-SQL-dreamer*
