# hatch-plugins

Kerry Hatcher's AI tools marketplace for [Claude Code](https://code.claude.com/docs/en/plugin-marketplaces.md) plugins.

## Usage

Add this marketplace in Claude Code:

```
/plugin marketplace add kerryhatcher/hatch-plugins
```

Then install a plugin from it:

```
/plugin install status-line@hatch-plugins
```

## Plugins

| Plugin | Description |
|--------|-------------|
| [status-line](https://github.com/kerryhatcher/status-line) | A rich terminal status line for Claude Code showing model, context usage, active tasks, and project state. |

## Repo layout

```
hatch-plugins/
├── .claude-plugin/
│   └── marketplace.json   # marketplace manifest
└── plugins/                # one directory per locally-hosted plugin
```

To add a new plugin, create a directory under `plugins/`, give it a `.claude-plugin/plugin.json`, and add an entry to `.claude-plugin/marketplace.json`.
