# Grimoire Storage Location Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the `grimoire` plugin a Python utility that resolves and creates its data directory, following the precedence `$GRIMOIRE_HOME` env var > `~/.grimoire/.env` > default `~/.grimoire`, exposed as an importable module and a CLI entry point other plugin components (hooks, skills, commands) can call.

**Architecture:** A small `uv`-managed Python package (`grimoire`) inside `plugins/grimoire/`, with a `storage` module holding pure, dependency-injectable resolution/creation functions, and a `cli` module wrapping it for shell callers. TDD throughout with `pytest`.

**Tech Stack:** Python 3.11+, managed exclusively via `uv` (never system `python`/`python3`), `pytest` for tests, stdlib only (`pathlib`, `os`) — no third-party runtime dependencies.

## Global Constraints

- Linux and macOS only — no Windows support (per spec non-goals).
- All Python invocations use `uv run` / `uv add` — never bare `python`/`python3`.
- First-run directory/file creation is silent — no prompts, no confirmations.
- Env var name is exactly `GRIMOIRE_HOME` (not `GRIMOIRE_DATA_DIR` or any variant).
- The anchor file is exactly `~/.grimoire/.env`, and it is never itself relocated by `GRIMOIRE_HOME`.
- Data directory contents are exactly `notes/` (empty dir, format deferred) and `grimoire.db` (empty file, schema deferred) — do not add anything else.
- No third-party dependencies beyond `pytest` (dev-only) — resolution logic must work with stdlib alone.

---

### Task 1: Scaffold the uv Python project

**Files:**
- Create: `plugins/grimoire/pyproject.toml`
- Create: `plugins/grimoire/src/grimoire/__init__.py`
- Create: `plugins/grimoire/tests/__init__.py`

**Interfaces:**
- Produces: a `grimoire` package importable as `from grimoire import ...`, with `pytest` runnable via `uv run pytest` from `plugins/grimoire/`.

- [ ] **Step 1: Initialize the uv package**

Run from the repo root:

```bash
cd plugins/grimoire && uv init --package --name grimoire .
```

Expected: uv reports initializing the project and creates `pyproject.toml`, `src/grimoire/__init__.py`, and a `.python-version` file. It will skip `README.md` since one already exists.

- [ ] **Step 2: Add pytest as a dev dependency**

Run:

```bash
uv add --dev pytest
```

Expected: `pytest` and its dependencies are added to `pyproject.toml` under a dev dependency group, and `uv.lock` is created/updated.

- [ ] **Step 3: Create the tests package and verify the test runner works**

Create `plugins/grimoire/tests/__init__.py` (empty file).

Run:

```bash
uv run pytest
```

Expected: `no tests ran` (or `collected 0 items`) with exit code 0 — confirms the project and pytest are wired up correctly before any real tests exist.

- [ ] **Step 4: Commit**

```bash
git add plugins/grimoire/pyproject.toml plugins/grimoire/uv.lock plugins/grimoire/src plugins/grimoire/tests plugins/grimoire/.python-version
git commit -m "chore(grimoire): scaffold uv python project"
```

---

### Task 2: Implement `.env` file parsing

**Files:**
- Create: `plugins/grimoire/src/grimoire/storage.py`
- Test: `plugins/grimoire/tests/test_storage.py`

**Interfaces:**
- Produces: `read_env_file(path: Path) -> dict[str, str]` — parses simple `KEY=value` lines, ignores blank lines and lines starting with `#`, returns `{}` if the file doesn't exist.

- [ ] **Step 1: Write the failing tests**

Create `plugins/grimoire/tests/test_storage.py`:

```python
from grimoire.storage import read_env_file


def test_read_env_file_parses_key_value_pairs(tmp_path):
    env_file = tmp_path / ".env"
    env_file.write_text("GRIMOIRE_HOME=/some/path\nOTHER=value\n")
    assert read_env_file(env_file) == {"GRIMOIRE_HOME": "/some/path", "OTHER": "value"}


def test_read_env_file_missing_file_returns_empty(tmp_path):
    assert read_env_file(tmp_path / "does-not-exist.env") == {}


def test_read_env_file_skips_comments_and_blank_lines(tmp_path):
    env_file = tmp_path / ".env"
    env_file.write_text("# a comment\n\nGRIMOIRE_HOME=/x\n")
    assert read_env_file(env_file) == {"GRIMOIRE_HOME": "/x"}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```bash
uv run pytest tests/test_storage.py -v
```

Expected: `ModuleNotFoundError: No module named 'grimoire.storage'` (or collection error) — `storage.py` doesn't exist yet.

- [ ] **Step 3: Implement `read_env_file`**

Create `plugins/grimoire/src/grimoire/storage.py`:

```python
from __future__ import annotations

from pathlib import Path


def read_env_file(path: Path) -> dict[str, str]:
    """Parse a simple KEY=value .env file. Returns {} if the file doesn't exist."""
    if not path.is_file():
        return {}
    values: dict[str, str] = {}
    for line in path.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#") or "=" not in line:
            continue
        key, _, value = line.partition("=")
        values[key.strip()] = value.strip()
    return values
```

- [ ] **Step 4: Run tests to verify they pass**

Run:

```bash
uv run pytest tests/test_storage.py -v
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/grimoire/src/grimoire/storage.py plugins/grimoire/tests/test_storage.py
git commit -m "feat(grimoire): parse .env key-value files"
```

---

### Task 3: Implement data directory resolution precedence

**Files:**
- Modify: `plugins/grimoire/src/grimoire/storage.py`
- Modify: `plugins/grimoire/tests/test_storage.py`

**Interfaces:**
- Consumes: `read_env_file(path: Path) -> dict[str, str]` (Task 2)
- Produces: `resolve_data_dir(env: dict[str, str] | None = None, home: Path | None = None) -> Path` — resolves the data directory per precedence: `env["GRIMOIRE_HOME"]` > `GRIMOIRE_HOME` key from `<home>/.grimoire/.env` > `<home>/.grimoire`. Defaults `env` to `os.environ` and `home` to `Path.home()` when not passed.

- [ ] **Step 1: Write the failing tests**

Append to `plugins/grimoire/tests/test_storage.py`:

```python
from pathlib import Path

from grimoire.storage import resolve_data_dir


def test_resolve_data_dir_uses_env_var_when_set(tmp_path):
    result = resolve_data_dir(env={"GRIMOIRE_HOME": "/from/env/var"}, home=tmp_path)
    assert result == Path("/from/env/var")


def test_resolve_data_dir_uses_env_file_when_env_var_absent(tmp_path):
    anchor = tmp_path / ".grimoire"
    anchor.mkdir()
    (anchor / ".env").write_text("GRIMOIRE_HOME=/from/env/file\n")
    result = resolve_data_dir(env={}, home=tmp_path)
    assert result == Path("/from/env/file")


def test_resolve_data_dir_env_var_takes_precedence_over_env_file(tmp_path):
    anchor = tmp_path / ".grimoire"
    anchor.mkdir()
    (anchor / ".env").write_text("GRIMOIRE_HOME=/from/env/file\n")
    result = resolve_data_dir(env={"GRIMOIRE_HOME": "/from/env/var"}, home=tmp_path)
    assert result == Path("/from/env/var")


def test_resolve_data_dir_defaults_to_home_dotgrimoire(tmp_path):
    result = resolve_data_dir(env={}, home=tmp_path)
    assert result == tmp_path / ".grimoire"
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```bash
uv run pytest tests/test_storage.py -v
```

Expected: the 4 new tests fail with `ImportError: cannot import name 'resolve_data_dir'`.

- [ ] **Step 3: Implement `resolve_data_dir`**

Add to `plugins/grimoire/src/grimoire/storage.py` (below `read_env_file`):

```python
import os

ANCHOR_DIRNAME = ".grimoire"
ENV_FILE_NAME = ".env"
ENV_VAR_NAME = "GRIMOIRE_HOME"


def anchor_dir(home: Path) -> Path:
    return home / ANCHOR_DIRNAME


def resolve_data_dir(env: dict[str, str] | None = None, home: Path | None = None) -> Path:
    """Resolve grimoire's data directory.

    Precedence: $GRIMOIRE_HOME env var > GRIMOIRE_HOME in ~/.grimoire/.env > ~/.grimoire default.
    """
    env = os.environ if env is None else env
    home = Path.home() if home is None else home

    if env.get(ENV_VAR_NAME):
        return Path(env[ENV_VAR_NAME]).expanduser()

    env_file_values = read_env_file(anchor_dir(home) / ENV_FILE_NAME)
    if env_file_values.get(ENV_VAR_NAME):
        return Path(env_file_values[ENV_VAR_NAME]).expanduser()

    return anchor_dir(home)
```

Add `import os` at the top of the file alongside the existing `from pathlib import Path` (keep `from __future__ import annotations` first).

- [ ] **Step 4: Run tests to verify they pass**

Run:

```bash
uv run pytest tests/test_storage.py -v
```

Expected: 7 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/grimoire/src/grimoire/storage.py plugins/grimoire/tests/test_storage.py
git commit -m "feat(grimoire): resolve data dir with env var / env file / default precedence"
```

---

### Task 4: Implement data directory creation

**Files:**
- Modify: `plugins/grimoire/src/grimoire/storage.py`
- Modify: `plugins/grimoire/tests/test_storage.py`

**Interfaces:**
- Consumes: `resolve_data_dir(...)` (Task 3)
- Produces: `ensure_data_dir(data_dir: Path) -> Path` — creates `<data_dir>/notes/` and an empty `<data_dir>/grimoire.db` if missing, silently, without overwriting an existing `grimoire.db`; returns `data_dir`. `get_data_dir(env=None, home=None) -> Path` — resolves then ensures; the main entry point for consumers.

- [ ] **Step 1: Write the failing tests**

Append to `plugins/grimoire/tests/test_storage.py`:

```python
from grimoire.storage import ensure_data_dir, get_data_dir


def test_ensure_data_dir_creates_notes_subdir(tmp_path):
    data_dir = tmp_path / "vault"
    ensure_data_dir(data_dir)
    assert (data_dir / "notes").is_dir()


def test_ensure_data_dir_creates_empty_grimoire_db(tmp_path):
    data_dir = tmp_path / "vault"
    ensure_data_dir(data_dir)
    assert (data_dir / "grimoire.db").is_file()


def test_ensure_data_dir_does_not_overwrite_existing_db(tmp_path):
    data_dir = tmp_path / "vault"
    data_dir.mkdir()
    db = data_dir / "grimoire.db"
    db.write_bytes(b"existing-content")
    ensure_data_dir(data_dir)
    assert db.read_bytes() == b"existing-content"


def test_get_data_dir_resolves_and_creates_default(tmp_path):
    result = get_data_dir(env={}, home=tmp_path)
    assert result == tmp_path / ".grimoire"
    assert (result / "notes").is_dir()
    assert (result / "grimoire.db").is_file()
```

- [ ] **Step 2: Run tests to verify they fail**

Run:

```bash
uv run pytest tests/test_storage.py -v
```

Expected: the 4 new tests fail with `ImportError: cannot import name 'ensure_data_dir'`.

- [ ] **Step 3: Implement `ensure_data_dir` and `get_data_dir`**

Add to `plugins/grimoire/src/grimoire/storage.py` (at the end of the file):

```python
def ensure_data_dir(data_dir: Path) -> Path:
    """Create the data dir's notes/ subdir and an empty grimoire.db if missing. Silent, no prompts."""
    (data_dir / "notes").mkdir(parents=True, exist_ok=True)
    db_path = data_dir / "grimoire.db"
    if not db_path.exists():
        db_path.touch()
    return data_dir


def get_data_dir(env: dict[str, str] | None = None, home: Path | None = None) -> Path:
    """Resolve and ensure grimoire's data directory exists. Main entry point for consumers."""
    data_dir = resolve_data_dir(env=env, home=home)
    return ensure_data_dir(data_dir)
```

- [ ] **Step 4: Run tests to verify they pass**

Run:

```bash
uv run pytest tests/test_storage.py -v
```

Expected: 11 passed.

- [ ] **Step 5: Commit**

```bash
git add plugins/grimoire/src/grimoire/storage.py plugins/grimoire/tests/test_storage.py
git commit -m "feat(grimoire): create data dir contents on first use"
```

---

### Task 5: Add a CLI entry point for hooks/skills to consume

**Files:**
- Create: `plugins/grimoire/src/grimoire/cli.py`
- Create: `plugins/grimoire/tests/test_cli.py`
- Modify: `plugins/grimoire/pyproject.toml`

**Interfaces:**
- Consumes: `get_data_dir(...)` (Task 4)
- Produces: `main() -> None` in `grimoire.cli`, printing the resolved data dir path to stdout. Registered as the `grimoire-storage-path` console script, invocable as `uv run grimoire-storage-path` from `plugins/grimoire/` (other plugin components will call this via `uv run --project ${CLAUDE_PLUGIN_ROOT}/plugins/grimoire grimoire-storage-path` once wired into hooks/skills in a later sub-project).

- [ ] **Step 1: Write the failing test**

Create `plugins/grimoire/tests/test_cli.py`:

```python
import grimoire.cli as cli


def test_cli_main_prints_resolved_data_dir(tmp_path, monkeypatch, capsys):
    monkeypatch.setenv("GRIMOIRE_HOME", str(tmp_path / "custom-vault"))
    cli.main()
    captured = capsys.readouterr()
    assert captured.out.strip() == str(tmp_path / "custom-vault")
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
uv run pytest tests/test_cli.py -v
```

Expected: `ModuleNotFoundError: No module named 'grimoire.cli'`.

- [ ] **Step 3: Implement the CLI module**

Create `plugins/grimoire/src/grimoire/cli.py`:

```python
from __future__ import annotations

from grimoire.storage import get_data_dir


def main() -> None:
    print(get_data_dir())


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Register the console script**

Edit `plugins/grimoire/pyproject.toml`, adding this section (create the `[project.scripts]` table if `uv init` didn't already add one):

```toml
[project.scripts]
grimoire-storage-path = "grimoire.cli:main"
```

Run:

```bash
uv sync
```

Expected: uv reinstalls the project in editable mode and registers the `grimoire-storage-path` script.

- [ ] **Step 5: Run tests to verify they pass**

Run:

```bash
uv run pytest -v
```

Expected: 12 passed (all tests from Tasks 2-5).

- [ ] **Step 6: Verify the console script works end to end**

Run:

```bash
GRIMOIRE_HOME=/tmp/grimoire-cli-check uv run grimoire-storage-path
```

Expected: prints `/tmp/grimoire-cli-check` and creates `/tmp/grimoire-cli-check/notes/` and `/tmp/grimoire-cli-check/grimoire.db`. Clean up afterward:

```bash
rm -rf /tmp/grimoire-cli-check
```

- [ ] **Step 7: Commit**

```bash
git add plugins/grimoire/src/grimoire/cli.py plugins/grimoire/tests/test_cli.py plugins/grimoire/pyproject.toml plugins/grimoire/uv.lock
git commit -m "feat(grimoire): add CLI entry point for data dir resolution"
```

---

### Task 6: Document the storage configuration

**Files:**
- Modify: `plugins/grimoire/README.md`
- Modify: `plugins/grimoire/.claude-plugin/plugin.json`

**Interfaces:**
- Consumes: nothing (documentation only)
- Produces: nothing consumed by later tasks — this is the terminal task for this plan.

- [ ] **Step 1: Document storage configuration in the plugin README**

In `plugins/grimoire/README.md`, replace the existing "Status" section's bullet list with the following (keep the rest of the file, including the Installation section, as-is):

```markdown
## Status

Storage layer implemented. Remaining components not yet built:

- **Skills** — knowledge capture, organization, and retrieval patterns
- **Commands** — user-invoked actions (e.g. capture a note, review inbox)
- **Agents** — autonomous tasks (e.g. linking related notes, summarizing)
- **Workflows** — multi-step orchestration for larger operations (e.g. periodic review)
- **Hooks** — event-driven automation (e.g. auto-tagging on save)

## Data Storage

Grimoire stores its data in a directory resolved in this order:

1. `$GRIMOIRE_HOME` environment variable, if set.
2. `GRIMOIRE_HOME=` line in `~/.grimoire/.env`, if present.
3. Default: `~/.grimoire`.

`~/.grimoire/.env` is a fixed location — it is always checked there regardless of where `GRIMOIRE_HOME` points the actual data. The resolved directory is created automatically on first use, containing `notes/` and `grimoire.db`.

Requires [uv](https://docs.astral.sh/uv/) to run; managed via `plugins/grimoire/pyproject.toml`.
```

- [ ] **Step 2: Add `uv` as a documented prerequisite in `plugin.json`**

Read the current contents of `plugins/grimoire/.claude-plugin/plugin.json`, then update the `"keywords"` array to include `"uv"` and `"sqlite"`:

```json
"keywords": ["second-brain", "knowledge-management", "notes", "pkm", "uv", "sqlite"]
```

- [ ] **Step 3: Commit**

```bash
git add plugins/grimoire/README.md plugins/grimoire/.claude-plugin/plugin.json
git commit -m "docs(grimoire): document storage configuration"
```

---

## Verification

After Task 6, run the full suite once more from `plugins/grimoire/`:

```bash
uv run pytest -v
```

Expected: 12 passed, 0 failed.
