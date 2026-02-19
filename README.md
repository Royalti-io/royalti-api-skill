# Royalti.io API Skill

An [Agent Skill](https://agentskills.io) that gives your AI assistant complete knowledge of the [Royalti.io](https://royalti.io) REST API v2.6. When installed, your agent can help you write integration code, debug API calls, set up webhooks, and navigate the 287-endpoint API surface without you having to explain the patterns each time.

Works with **Claude Code**, **Claude Desktop**, **Cursor**, **OpenAI Codex**, **GitHub Copilot**, **OpenClaw**, **Gemini CLI**, **Amp**, **Goose**, **Roo Code**, **OpenCode**, **Letta**, and any tool that supports the [Agent Skills open standard](https://agentskills.io/specification).

## What's included

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
/plugin marketplace add Royalti-io/royalti-api-skill
/plugin install royalti-api
```

### Claude Code (manual)

```bash
# Clone and copy to your project
git clone https://github.com/Royalti-io/royalti-api-skill.git
cp -r royalti-api-skill/skills/royalti-api .claude/skills/

# Or install globally (all projects)
cp -r royalti-api-skill/skills/royalti-api ~/.claude/skills/
```

### Claude Desktop

1. Download the `skills/royalti-api/` folder from this repo
2. Zip it: `zip -r royalti-api.zip royalti-api/`
3. Open Claude Desktop → **Settings** → **Features**
4. Upload the `royalti-api.zip` file

The skill will be available in all your Claude Desktop conversations.

### OpenClaw

```bash
# Copy into your OpenClaw skills directory
cp -r skills/royalti-api ~/.openclaw/skills/
```

Or add the path to `skills.load.extraDirs` in `~/.openclaw/openclaw.json`.

### OpenSkills (universal installer)

```bash
npx openskills install Royalti-io/royalti-api-skill
```

### Cursor / Codex / GitHub Copilot / Gemini CLI / Others

This skill follows the [Agent Skills open standard](https://agentskills.io/specification). Copy `skills/royalti-api/` into the skills directory for your tool:

| Tool | Path |
|------|------|
| Cursor | `.cursor/skills/royalti-api/` |
| OpenAI Codex | `.codex/skills/royalti-api/` |
| GitHub Copilot (VS Code) | `.github/skills/royalti-api/` |
| Gemini CLI | `.gemini/skills/royalti-api/` |
| Amp | `.amp/skills/royalti-api/` |
| Goose | `.goose/skills/royalti-api/` |
| Roo Code | `.roo/skills/royalti-api/` |
| OpenCode | `.opencode/skills/royalti-api/` |
| Letta | `.letta/skills/royalti-api/` |

## Usage

Once installed, the skill activates automatically when your conversation involves the Royalti API. Examples:

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
│   └── plugin.json                    # Plugin manifest (Claude Code)
└── skills/
    └── royalti-api/
        ├── SKILL.md                   # The skill (Agent Skills standard)
        └── references/
            ├── examples-javascript.md # Node.js/TypeScript examples
            ├── examples-python.md     # Python examples
            └── examples-php.md        # PHP examples
```

Code examples cover authentication, CRUD operations, pagination, file upload, WebSocket events, and webhook receivers for each language.

## Compatibility

This skill follows the [Agent Skills open standard](https://agentskills.io/specification), an open specification originally developed by Anthropic and adopted by 25+ tools. Any AI coding assistant or agent that supports `SKILL.md` files can use this skill — write once, use everywhere.

## About Royalti.io

[Royalti.io](https://royalti.io) is a royalty management platform for music labels, distributors, and publishers. The API provides programmatic access to manage artists, tracks, releases, royalty splits, payments, analytics, and more.

- **API Base URL:** `https://api.royalti.io`
- **Developer Docs:** [apidocs.royalti.io](https://apidocs.royalti.io)
- **MCP Server:** [royalti.io/mcp](https://royalti.io/mcp) — AI assistant integration for direct API interaction
- **MCP Setup Guide:** [royalti.io/help/setting-up-the-royalti-mcp-server](https://royalti.io/help/setting-up-the-royalti-mcp-server)

## License

MIT
