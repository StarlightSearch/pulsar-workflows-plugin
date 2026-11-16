# Pulsar Workflows plugin

Build, run, test and improve [Pulsar Workflows](https://workflows.pulsarbot.cloud) without leaving Claude. Claude becomes the workflow builder: it lists your workflows, writes a workflow's `SKILL.md` and scripts, runs it in Pulsar's sandbox, keeps test cases, and reads the full traces of past runs to work out why a run went wrong and what to change.

## What you get

- **The Pulsar Workflows MCP server** (`https://workflows.pulsarbot.cloud/mcp`), with tools for projects, workflows, files, runs, test cases, run traces, the knowledge base and app connections.
- **Three skills** that teach Claude how to use them well:
  - `build-workflow`: pick the project, write SKILL.md, declare secrets and connected apps, get a first run working.
  - `test-and-improve`: run, read every result, fix the instructions, keep test cases.
  - `learn-from-runs`: find failed or slow tool calls, read a call or a subagent's answer whole, read full traces and the instructions a run had, and turn run history into concrete improvements.

## Install

In Claude Code:

```
/plugin marketplace add StarlightSearch/pulsar-workflows-plugin
/plugin install pulsar-workflows@starlightsearch
```

The first time Claude uses a Pulsar tool, your browser opens to sign in to Pulsar Workflows with your usual account and allow access. You need a Pulsar Workflows account: sign up at https://workflows.pulsarbot.cloud.

Using another agent? The same server works anywhere that supports remote MCP servers with OAuth; see https://workflows.pulsarbot.cloud/docs/agents.

## What it sends, and where

- Everything goes to Pulsar Workflows at `https://workflows.pulsarbot.cloud/mcp`, over HTTPS, signed in as you. Sign-in is handled by WorkOS AuthKit (`real-pleasure-88.authkit.app`); the plugin stores no credentials.
- The tools send what you ask Claude to do: workflow files you write, run inputs (text and files), test cases, knowledge files you add. They read back workflows, run results, files and traces from the projects you can open in Pulsar, and nothing else.
- Running a workflow executes it in Pulsar's sandbox and counts toward your organisation's usage. A workflow can act on apps connected to its project (for example Linear or Gmail), exactly as when you run it in the Pulsar web app.
- The plugin contains no scripts, hooks or executables: only this manifest, the MCP server address and three Markdown skills.

Privacy policy: https://pulsarbot.cloud/privacy. Terms: https://pulsarbot.cloud/terms.

## Support

Open an issue at https://github.com/StarlightSearch/pulsar-workflows-plugin/issues.

## License

MIT, see [LICENSE](LICENSE).
