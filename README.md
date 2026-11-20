# Pulsar Workflows plugin

Build, run, test and improve [Pulsar Workflows](https://workflows.pulsarbot.cloud) from Claude Code, Cursor or Codex. Your agent becomes the workflow builder: it lists your workflows, writes a workflow's `SKILL.md` and scripts, runs it in Pulsar's sandbox, keeps test cases, and reads the full traces of past runs to work out why a run went wrong and what to change.

## What you get

- **The Pulsar Workflows MCP server** (`https://workflows.pulsarbot.cloud/mcp`), with tools for projects, workflows, files, runs, test cases, run traces, the knowledge base and app connections.
- **Four skills** that teach your agent how to use them well:
  - `build-workflow`: pick the project, write SKILL.md, declare secrets and connected apps, get a first run working.
  - `test-and-improve`: run, read every result, fix the instructions, keep test cases.
  - `learn-from-runs`: find failed or slow tool calls, read a call or a subagent's answer whole, read full traces and the instructions a run had, and turn run history into concrete improvements.
  - `call-workflows-api`: call a workflow from your own code, with the TypeScript SDK ([`@starlightsearch/workflows`](https://www.npmjs.com/package/@starlightsearch/workflows)) or the HTTP API with curl: keys and slugs, waiting, structured output, files, idempotent retries and errors.

## Install

**Claude Code**

```
/plugin marketplace add StarlightSearch/pulsar-workflows-plugin
/plugin install pulsar-workflows@starlightsearch
```

**Cursor**: install **Pulsar Workflows** from the [Cursor Marketplace](https://cursor.com/marketplace), or from Customize → Plugins in Cursor.

**Codex**

```
codex plugin marketplace add StarlightSearch/pulsar-workflows-plugin
```

Then run `codex`, open `/plugins`, choose the Starlight Search marketplace and install **Pulsar Workflows**.

Each of these installs the MCP server and the skills together. The first time your agent uses a Pulsar tool, your browser opens to sign in to Pulsar Workflows with your usual account and allow access. You need a Pulsar Workflows account: sign up at https://workflows.pulsarbot.cloud.

Using another agent? The same server works anywhere that supports remote MCP servers with OAuth; see https://workflows.pulsarbot.cloud/docs/agents.

## Layout

One repository serves three agents. Keep the copies in step when you change the server or bump the version:

| File | Read by |
| --- | --- |
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.mcp.json` | Claude Code |
| `.cursor-plugin/plugin.json` (server inline) | Cursor |
| `plugin.json`, `mcp.json` ([Agent Plugins](https://agent-plugins.org) standard), `.agents/plugins/marketplace.json` | Codex, and any agent that reads Agent Plugins |
| `skills/` | all of them |

## What it sends, and where

- Everything goes to Pulsar Workflows at `https://workflows.pulsarbot.cloud/mcp`, over HTTPS, signed in as you. Sign-in is handled by WorkOS AuthKit (`real-pleasure-88.authkit.app`); the plugin stores no credentials.
- The tools send what you ask your agent to do: workflow files you write, run inputs (text and files), test cases, knowledge files you add. They read back workflows, run results, files and traces from the projects you can open in Pulsar, and nothing else.
- Running a workflow executes it in Pulsar's sandbox and counts toward your organisation's usage. A workflow can act on apps connected to its project (for example Linear or Gmail), exactly as when you run it in the Pulsar web app.
- The plugin contains no scripts, hooks or executables: only manifests, the MCP server address and three Markdown skills.

Privacy policy: https://pulsarbot.cloud/privacy. Terms: https://pulsarbot.cloud/terms.

## Support

Open an issue at https://github.com/StarlightSearch/pulsar-workflows-plugin/issues.

## License

MIT, see [LICENSE](LICENSE).
