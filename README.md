# DeepSieve plugin

Turn anything you need to know into a sourced dataset that keeps itself up to date.

This repository connects your coding agent to [DeepSieve](https://deepsieve.ai), an AI deep-research agent. DeepSieve turns a research question into structured rows: every cell gets a confidence verdict, and cited cells link to their sources. With monitoring on, values are re-checked on the schedule you set.

It holds no product code. It contains:

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

## Links

- Product: https://deepsieve.ai
- Developer docs: https://deepsieve.ai/developers
- Python SDK, MCP server and CLI: the `deepsieve`, `deepsieve-mcp` and `deepsieve-cli` packages on PyPI

## License

Apache-2.0. See [LICENSE](LICENSE).
