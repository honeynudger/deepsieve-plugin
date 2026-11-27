# DeepSieve plugin

Turn anything you need to know into a sourced dataset that keeps itself up to date.

This repository connects your coding agent to [DeepSieve](https://deepsieve.ai), an AI deep-research agent. DeepSieve turns a research question into structured rows: every cell gets a confidence verdict, and cited cells link to their sources. With monitoring on, values are re-checked on the schedule you set.

DeepSieve is operated by Honey Nudger Inc. This repository holds no product code. It contains:

- **Plugin manifests** for Claude Code, Cursor and Gemini CLI. Each one points at DeepSieve's remote MCP server, `https://deepsieve.ai/mcp`, over streamable HTTP.
- **The DeepSieve skill** (`skills/deepsieve/`). It shows your agent how to run research, read cited data, keep a dataset fresh and export it. It is the same skill that is published at [deepsieve.ai/developers](https://deepsieve.ai/developers).

## Install

**Claude Code**

```
/plugin marketplace add honeynudger/deepsieve-plugin
/plugin install deepsieve@deepsieve-plugin
```

**Gemini CLI**

```
gemini extensions install https://github.com/honeynudger/deepsieve-plugin
```

**Any MCP client:** add the remote server `https://deepsieve.ai/mcp` (streamable HTTP).

The first time you connect, you sign in to DeepSieve in your browser. You do not need an API key. For REST calls or CI, create an API key in the app instead; the skill explains how.

The plugin, MCP connection, and dry_run test mode are free; live research runs use your DeepSieve account (new organizations include one free standard run, no card required).

**Other MCP clients (JSON config):**

```json
{
  "mcpServers": {
    "deepsieve": { "type": "http", "url": "https://deepsieve.ai/mcp" }
  }
}
```

## What the MCP server does

The server gives your agent typed tools over DeepSieve's API. With them, an agent can:

- **Design a dataset:** propose a Blueprint (the tables, columns and research instructions) from a plain-language brief, then adjust it before anything is built.
- **Run research:** estimate a run's cost, start it (or a free `dry_run` first), and check its status.
- **Read cited data:** query the resulting rows. Each cell carries a confidence verdict, and cited cells link to their sources.
- **Check history:** see what changed in a dataset, or in a single cell, over time.
- **Keep it fresh:** turn monitoring on, so values are re-checked on a schedule.
- **Ask and export:** ask the research agent about a report, and export a dataset.

A browser-login session can't apply a change to your active Blueprint, manage API keys, or start or change a charge. Those stay with a person in the app.

## Using it

Once connected, ask your agent in plain language, for example:

- "Use DeepSieve to build a dataset of AI meeting assistants with pricing, integrations and security certifications. Show me the plan before running it."
- "Do a free dry run first, then tell me what a real run would cost."
- "Which values in my market map are contradicted by their sources?"
- "What changed in my competitor dataset since last week?"

Full tool reference and recipes: [deepsieve.ai/developers/mcp](https://deepsieve.ai/developers/mcp) and the skill in `skills/deepsieve/`.

## Links

- Product: https://deepsieve.ai
- Developer docs: https://deepsieve.ai/developers
- Python SDK, MCP server and CLI: the `deepsieve`, `deepsieve-mcp` and `deepsieve-cli` packages on PyPI
- Terms of Service: https://deepsieve.ai/terms
- Privacy Policy: https://deepsieve.ai/privacy (DeepSieve is operated by Honey Nudger Inc.)
- Support: [GitHub Issues](https://github.com/honeynudger/deepsieve-plugin/issues) on this repository, or support+deepsieve@honeynudger.ai

## License

Apache-2.0. See [LICENSE](LICENSE).
