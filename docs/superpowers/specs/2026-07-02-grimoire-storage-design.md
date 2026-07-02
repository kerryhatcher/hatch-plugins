# Grimoire: Data Storage Location Design

## Context

Grimoire is a second-brain plugin for Claude Code (skills, commands, agents, workflows, and hooks for capturing, organizing, and retrieving knowledge). It is the first plugin in the `plugin-marketplace` repo, currently scaffolded but unimplemented.

Grimoire is too large to spec in one pass — it spans several independent subsystems (capture, organization/linking, retrieval, review, and the storage layer underneath all of them). This document specs only the **storage location and configuration** sub-project: where grimoire's data lives on disk, cross-platform (Linux and macOS), with a sensible default that's user-configurable. Everything else (note format, database schema, capture/retrieval behavior) is out of scope here and will get its own spec.

## Goals

- A single, predictable default location for grimoire's data that works on both Linux and macOS.
- A configuration mechanism that lets a user relocate the data directory without editing code.
- No circular dependency between "where is the config" and "where is the data."

## Non-Goals

- Note file format/schema (frontmatter fields, linking syntax) — deferred to the capture/data-model sub-project.
- `grimoire.db` schema (tables, indexes, search index design) — deferred to the same sub-project.
- Multi-vault support (switching between multiple `GRIMOIRE_HOME`s at runtime) — not requested; YAGNI.
- Windows support — explicitly out of scope; Linux and macOS only.

## Design

### Fixed anchor + override

`~/.grimoire/.env` is a **fixed, never-relocated anchor point**. It is always checked at this exact path regardless of where the actual vault data ends up living. This avoids the chicken-and-egg problem of a relocatable config file that has to be found before you know where things are.

### Resolution order

The data directory is resolved in this order, first match wins:

1. **`$GRIMOIRE_HOME` environment variable**, if set — used directly as the data directory.
2. **`~/.grimoire/.env`**, if it exists and contains a `GRIMOIRE_HOME=` line — that value is used as the data directory.
3. **Default** — `~/.grimoire`.

If the data directory is never overridden, config and data simply live in the same place (`~/.grimoire`).

### `.env` format

Simple `KEY=value` per line, e.g.:

```
GRIMOIRE_HOME=/path/to/vault
```

Only `GRIMOIRE_HOME` is defined for now; the format is extensible for future settings without a redesign.

### Data directory contents

Once resolved, the data directory contains:

- `<data_dir>/notes/` — note files (format defined in a later sub-project; plain files, likely Markdown with frontmatter)
- `<data_dir>/grimoire.db` — SQLite database for structured metadata: tags, links, timestamps, search index

SQLite was chosen for the structured half of the data (over plain JSON/YAML index files) because it's single-file, zero-config, requires no server process, and is well-suited to a local CLI/agent tool.

### First-run behavior

If the resolved data directory does not exist, grimoire creates it **silently, without prompting**: `notes/` subdirectory plus an empty `grimoire.db`. If `~/.grimoire/.env` does not exist, that's not an error — it's optional; resolution just falls through to the default.

## Environment Variable Naming

`GRIMOIRE_HOME`, following the convention of tools like `$HOME`, `$CARGO_HOME`, `$GOPATH` — it signals "root of everything grimoire," not just a narrow data path.

## Open Questions for Future Sub-Projects

- Note file format and frontmatter schema
- `grimoire.db` table design
- Whether multi-vault support becomes worth adding later
