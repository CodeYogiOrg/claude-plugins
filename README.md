# CodeYogi Claude Code plugins

Org marketplace for [Claude Code](https://code.claude.com) plugins.

## Install

Once, inside Claude Code:

```
/plugin marketplace add CodeYogiOrg/claude-plugins
/plugin install whatsapp-crm@codeyogi
```

## Update

```
/plugin marketplace update
```

## Plugins

| Plugin | What it gives you |
|---|---|
| `whatsapp-crm` | `/crm-shortlist` — reads a chat export from the CodeYogi WhatsApp desktop app and writes a shortlist of conversations worth adding to the CRM. No network, no CRM writes; adding a contact stays a human action. |

## Adding a plugin

Add a folder under `plugins/<name>/` with `.claude-plugin/plugin.json` and your
`skills/`/`commands/`/`agents/`, then list it in `.claude-plugin/marketplace.json`.
Bump the plugin's `version` on every change — that is what tells installed clients
to pick up the update.
