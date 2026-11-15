---
name: build-workflow
description: Use when the person wants to create a new Pulsar workflow, turn a repeated task into a workflow, or change what an existing Pulsar workflow does. Covers picking the project, writing SKILL.md and its scripts through the pulsar-workflows MCP tools, declaring secrets and connected apps, and getting a first run working.
---

# Build a Pulsar workflow

A Pulsar workflow is a task someone repeats, written down so an agent can do it on every run with nobody watching. It is a `SKILL.md` plus any scripts and reference files, stored in Pulsar and run in Pulsar's sandbox. You are the author: you write the files with the `pulsar-workflows` tools, Pulsar stores and runs them.

## Before you write anything

1. Call `get_authoring_guide` once per conversation and follow it. It is the source of truth for SKILL.md's frontmatter, the sandbox layout (`/workspace/workflow`, `/workspace/inputs`, `/workspace/knowledge`, `/workspace/out`) and the models on offer. Do not guess these.
2. Call `list_projects` (every tool's `project_id` comes from it) and confirm with the person which project the workflow belongs to when there is more than one. Never pick one silently.
3. Call `list_workflows` for that project. If a workflow already does most of the job, improve it instead of creating a near-duplicate.
4. Ask only what you cannot reasonably default: what the workflow is given, what it must hand back, and which apps or systems it touches. One question at a time.

## Create and write

1. `create_workflow` with a clear name and a one-sentence description.
2. Write `SKILL.md` with `write_file`. The response includes `errors`: fix every one before going on, then write again.
3. Put mechanical work (parsing, maths, formatting) in scripts under `scripts/` and call them by absolute path from the steps. Long reference material goes in `references/`.
4. Before editing an existing file, read it (`read_file` or `get_workflow`) and pass its sha256 as `expected_sha256`. A `file_changed` error means someone else edited it: read it again and merge, never overwrite blindly.

## Secrets and connected apps

- List every project secret the workflow uses under `secrets:` and use it as `with-secrets NAME -- command`. You never see secret values and must never ask the person to paste one into the chat: they add secrets in Pulsar's Secrets page.
- List every connected app under `connections:` (for example `connections: [linear]`). The `write_file` response for SKILL.md shows each app's state.
- When an app is missing, call `request_connection`. A project admin gets a link to open in the browser; anyone else gets who to ask. Give the person the link and wait for them to say it is done. A run started without the app stops with `connection_missing`.
- Writes to connected apps happen during a run with no approval, so the steps must say exactly which writes the workflow may make.

## First run

Draft, then prove it: run it once on a realistic input with `start_run` and `wait_for_run`. If the person has no example, make a small realistic one and say so. Then hand over to the `test-and-improve` skill.

## What not to do

- Do not create workflows in a project the person did not choose.
- Do not delete workflows or files unless the person asked; `delete_workflow` needs the exact name and cannot be undone.
- Do not start the same run twice without changing something: every run costs money.
