---
name: learn-from-runs
description: Use when the person asks why a Pulsar workflow run failed, was slow or gave a wrong answer, wants to review past runs for patterns, or wants a workflow improved from its run history. Covers listing runs, finding failed or slow tool calls, reading full traces page by page, and turning what the traces show into concrete changes.
---

# Learn from a workflow's past runs

Every Pulsar run keeps a full trace: the input, the instructions the agent was given, every message, and every tool call with its full arguments and result (secrets masked). Read traces to find out why something happened, not only that it happened.

## Find the runs worth reading

- `list_runs` for the workflow. Filter by `status` (for example `failed`), by `since` (an ISO date), or by `version` to compare before and after an edit. Each run records the workflow version it ran.
- Start with failed runs and runs whose answers the person says were wrong. Then look at slow or expensive successes.

## Find the moment it went wrong

1. `list_tool_calls` on a run, with `ok: false` to see only failures, or a `tool` name to see one tool. Each row has the tool, whether it worked, an error kind (`timeout`, `not_found`, `permission`, `invalid_input`, `other`), its duration and the trace `entry` where it happened. Page with `limit` and `offset` for long runs.
2. `get_trace` from that `entry` (use `from`), a page at a time; follow `next` to continue. The first call also returns the header: status, error code, model, usage and the workflow version.
3. Read the steps just before the failure: the cause is usually a wrong assumption a few calls earlier, not the call that errored.

## Look for patterns across runs

When reviewing several runs, keep a short table for yourself: run, version, outcome, first bad step, cause. Common causes and their fixes:

- **The run guessed a missing value.** Give a default in the instructions.
- **A tool was called with the wrong arguments.** Show the exact command or example call in the step.
- **A step was skipped.** Make it its own numbered step with a one-line summary.
- **A connected app or secret was missing.** Declare it under `connections:` or `secrets:`; see the `build-workflow` skill.
- **The answer was right in form but wrong in substance.** Add a check in the steps and a test case with a verifiable check.
- **It was slow.** Move mechanical work into a script and avoid repeated reads of the same files.

## Turn it into changes

Propose the smallest change to `SKILL.md` or a script that addresses the cause, make it (read first, pass `expected_sha256`), and prove it with the `test-and-improve` loop: rerun the failing input, then the test cases. Save the failing input as a test case so it cannot regress silently.

Traces can contain customer data from real runs. Quote only what you need, and never copy a trace's contents anywhere outside Pulsar without the person asking.
