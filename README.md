# hatch-plugins

Kerry Hatcher's AI tools marketplace for [Claude Code](https://code.claude.com/docs/en/plugin-marketplaces.md).

## Install a plugin

Add the marketplace once:

```
/plugin marketplace add kerryhatcher/hatch-plugins
```

Then install any plugin from it and restart the session:

```
/plugin install nobots@hatch-plugins
/plugin install status-line@hatch-plugins
```

Run `/plugin` any time to browse, enable, disable, or remove what's installed.

## Plugins

| Plugin | What it does | Repo |
|--------|--------------|------|
| **nobots** | Humanize prose as Claude writes it, detect AI-sounding text on demand, and audit documents for AI writing tells. | [kerryhatcher/nobot](https://github.com/kerryhatcher/nobot) |
| **status-line** | A rich terminal status line showing model, context usage, active tasks, and project state. | [kerryhatcher/status-line](https://github.com/kerryhatcher/status-line) |

Each plugin's own repo has full usage docs. The install commands above are all you need to get started.

## Repo layout

```
hatch-plugins/
├── .claude-plugin/
│   └── marketplace.json   # marketplace manifest
└── plugins/               # one directory per locally-hosted plugin
```

To add a new plugin, create a directory under `plugins/`, give it a `.claude-plugin/plugin.json`, and add an entry to `.claude-plugin/marketplace.json` (or reference an external repo by URL and pinned sha, as `nobots` does).
