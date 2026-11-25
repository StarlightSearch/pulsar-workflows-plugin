---
name: learn-from-runs
description: Use when the person asks why a Pulsar workflow run failed, was slow or gave a wrong answer, wants to review past runs for patterns, or wants a workflow improved from its run history. Covers listing runs, finding failed or slow tool calls, reading one call or subagent answer whole, reading full traces page by page, comparing a run's instructions with today's, and turning what the traces show into concrete changes.
---

# Learn from a workflow's past runs

Every Pulsar run keeps a full trace: the input, the instructions the agent was given, every message, and every tool call with its full arguments and result (secrets masked). Read traces to find out why something happened, not only that it happened.

## Find the runs worth reading

- `list_runs` for the workflow. Filter by `status` (for example `failed`), by `since` (an ISO date), or by `version` to compare before and after an edit. Each run records the workflow version it ran; `get_workflow` gives the current one.
- Start with failed runs and runs whose answers the person says were wrong. Then look at slow or expensive successes.

## Find the moment it went wrong

1. `list_tool_calls` on a run, with `ok: false` to see only failures, or a `tool` name to see one tool. Each row has the `ordinal`, the `call_id`, the tool, whether it worked, an error kind (`timeout`, `not_found`, `permission`, `invalid_input`, `other`), its duration and the trace `entry` where it happened. A `subagent` call also has its `description` (the subagent's name), so `tool: "subagent"` lists the reviewers of every pass and tells them apart. Page with `limit` and `offset` for long runs.
2. `get_tool_call` with an `ordinal` (or a `tool_call_id`) gives one call whole: its full arguments and the full text of its result. For a subagent call the result is the subagent's final answer, so a reviewer's verdict is one call away, with no paging. A result longer than `max_result_chars` comes back cut (`result_cut: true`); read on with `result_offset`.
3. `get_trace` from that `entry` (use `from`) to read what led up to it, a page at a time; follow `next` to continue. Pages stop at about 60 KB (`max_bytes`), and each entry carries its `index`. The header (input, system prompt, status, error code, model, usage, workflow version) comes only with the page that starts at 0; pass `include_header: true` to get it elsewhere. To keep pages small:
   - `top_level_only: true` leaves out what subagents did inside their own sessions (entries with `parentToolCallId`), so the main agent's steps read straight through. A subagent's answer is its call's result (`get_tool_call`); to see how it got there, page that stretch without `top_level_only`.
   - `max_result_chars` (for example 2000) cuts long tool results; a cut result ends with a marker naming the call's id, to read whole with `get_tool_call`.
4. Read the steps just before the failure: the cause is usually a wrong assumption a few calls earlier, not the call that errored.

## Check which instructions the run had

A workflow may have been edited since the run. Before blaming the current `SKILL.md`, compare the run's `workflow_version` (from `get_run` or `list_runs`) with the current version from `get_workflow`. If they differ, `read_file` with `version` set to the run's `workflow_version` gives the file as that run ran it (`list_files` with `version` lists those files), so you can diff it against today's and see whether a later edit already addressed the cause.

## Look for patterns across runs

When reviewing several runs, keep a short table for yourself: run, version, outcome, first bad step, cause. Common causes and their fixes:

- **The run guessed a missing value.** Give a default in the instructions.
- **A tool was called with the wrong arguments.** Show the exact command or example call in the step.
- **A step was skipped.** Make it its own numbered step with a one-line summary.
- **A connected app or secret was missing.** Declare it under `connections:` or `secrets:`; see the `build-workflow` skill.
- **The answer was right in form but wrong in substance.** Add a check in the steps and a test case with a verifiable check.
- **It was slow.** Move mechanical work into a script and avoid repeated reads of the same files.

## Turn it into changes

When you point at a run, link it with its `dashboard_url` (from `get_run` or `list_runs`) so the person can open it in Pulsar.

Propose the smallest change to `SKILL.md` or a script that addresses the cause, make it (read first, pass `expected_sha256`), and prove it with the `test-and-improve` loop: rerun the failing input, then the test cases. Save the failing input as a test case so it cannot regress silently.

Traces can contain customer data from real runs. Quote only what you need, and never copy a trace's contents anywhere outside Pulsar without the person asking.
