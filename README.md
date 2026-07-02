# plugin-marketplace

Kerry Hatcher's AI tools marketplace for [Claude Code](https://code.claude.com/docs/en/plugin-marketplaces.md) plugins.

## Usage

Add this marketplace in Claude Code:

```
/plugin marketplace add kerryhatcher/plugin-marketplace
```

Then install a plugin from it:

```
/plugin install grimoire@plugin-marketplace
```

## Plugins

| Plugin | Description |
|--------|-------------|
| [grimoire](plugins/grimoire) | Second-brain system for capturing, organizing, and retrieving knowledge |

## Repo layout

```
plugin-marketplace/
├── .claude-plugin/
│   └── marketplace.json   # marketplace manifest
└── plugins/
    └── grimoire/           # one directory per plugin
        └── .claude-plugin/plugin.json
```

To add a new plugin, create a directory under `plugins/`, give it a `.claude-plugin/plugin.json`, and add an entry to `.claude-plugin/marketplace.json`.
