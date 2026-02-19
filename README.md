# Royalti.io API Skill

An [Agent Skill](https://agentskills.io) that gives your AI coding assistant complete knowledge of the [Royalti.io](https://royalti.io) REST API v2.6. When installed, your agent can help you write integration code, debug API calls, set up webhooks, and navigate the 287-endpoint API surface without you having to explain the patterns each time.

## What's included

The skill covers:

- **Authentication** — Two-step JWT flow, API keys (RWAK/RUAK), OAuth, rate limits, RBAC roles
- **Request patterns** — Pagination, filtering, search, bulk operations
- **Response patterns** — Success, error, paginated list, and summary response shapes
- **Core resources** — CRUD patterns for users, artists, assets, products, splits, royalties, payments, files, labels, releases, and more
- **Analytics** — 11 royalty analytics endpoints with filtering and period comparison
- **Webhooks** — 25 event types, payload structure, HMAC validation, delivery management
- **WebSocket events** — Real-time file processing via Socket.io
- **Data models** — User, Artist, Asset, Product, Split, Transaction schemas
- **Integration patterns** — Step-by-step flows for catalog sync, file upload, analytics, and webhook setup

## Installation

### Claude Code (plugin)

```bash
/plugin marketplace add Royalti-io/api-reference
/plugin install royalti-api
```

### Claude Code (manual)

Copy the skill directory into your project or global skills:

```bash
# Project-level (this project only)
mkdir -p .claude/skills
cp -r skills/royalti-api .claude/skills/

# Global (all your projects)
mkdir -p ~/.claude/skills
cp -r skills/royalti-api ~/.claude/skills/
```

Or clone and copy:

```bash
git clone https://github.com/Royalti-io/api-reference.git
cp -r api-reference/skills/royalti-api ~/.claude/skills/
```

### OpenSkills (cross-agent)

```bash
npx openskills install Royalti-io/api-reference
```

### Cursor / Codex CLI / Gemini CLI

This skill follows the [Agent Skills open standard](https://agentskills.io/specification) and is compatible with any tool that supports `SKILL.md` files. Copy the `skills/royalti-api/` directory into the appropriate location for your tool:

| Tool | Path |
|------|------|
| Cursor | `.cursor/skills/royalti-api/` |
| Codex CLI | `.codex/skills/royalti-api/` |
| Gemini CLI | `.gemini/skills/royalti-api/` |
| Roo Code | `.roo/skills/royalti-api/` |

## Usage

Once installed, the skill activates automatically when your conversation involves the Royalti API. You can also reference it directly:

```
"How do I authenticate with the Royalti API?"
"What's the pagination pattern for listing assets?"
"Help me set up webhooks to track payment events"
"Write a function to upload and process a royalty file"
```

## Repo structure

```
.
├── README.md
├── LICENSE
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
└── skills/
    └── royalti-api/
        └── SKILL.md         # The skill (Agent Skills standard)
```

## About Royalti.io

[Royalti.io](https://royalti.io) is a royalty management platform for music labels, distributors, and publishers. The API provides programmatic access to manage artists, tracks, releases, royalty splits, payments, analytics, and more.

- **API Base URL:** `https://api.royalti.io`
- **Developer Docs:** [docs.royalti.io](https://docs.royalti.io)

## License

MIT
