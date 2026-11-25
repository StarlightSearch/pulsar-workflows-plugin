---
name: test-and-improve
description: Use when the person wants to test a Pulsar workflow, check it still works after a change, save test cases, or make a workflow better at its job. Covers running workflows with the pulsar-workflows MCP tools, reading every result properly, keeping test cases, and fixing the instructions rather than the symptom.
---

# Test and improve a Pulsar workflow

A workflow is only as good as its last real run. Your loop: run, read everything that came back, fix the instructions, run again, and keep the inputs that matter as test cases.

## Run it

1. `start_run` with the workflow id and a realistic input: `text`, `files` (base64), or both. Pass `output_schema` only to try a schema the workflow does not declare.
2. When a run starts, give the person its `dashboard_url` so they can watch it in Pulsar. If `start_run` came back as a background task (the link is also at the end of its status), wait for its result (the status names the run id, which works with `cancel_run`, `get_trace` and `list_tool_calls` meanwhile). Otherwise `wait_for_run` until the run ends. Each call waits at most 50 seconds; when it answers `still_running`, call it again. Do not start another run meanwhile.
3. Every run costs money and takes up to a few minutes. Never repeat an input without a change in between, and tell the person before running many cases.

## Read everything

Look at all of it, not just the status:

- the answer and its `## Assumptions`: each assumption is a gap in the instructions;
- the structured output, when there is one: right in form is not the same as right in substance;
- the returned files;
- why it took the path it did: use the `learn-from-runs` skill (`list_tool_calls`, `get_trace`) whenever the result is wrong, slow or surprising.

## Fix the instructions, not the symptom

- Read `SKILL.md` (and scripts) again before editing, and pass `expected_sha256` when writing.
- Prefer explaining why a step matters, and give a default for every judgement call the run had to guess.
- Change one thing per iteration when you can, so the next run tells you whether it worked.
- After a change, rerun the input that failed, then the test cases, to catch regressions.

## Test cases

- `save_test_case` from a run whose input represents a real situation: the typical case, an edge case, a case that once failed. Name it after the situation.
- `list_test_cases` shows each case and how its last run went; `run_test_cases` reruns all of them (or the ids you pass) and returns run ids to `wait_for_run`. If it came back as a background task, wait for its result instead (the status names the run ids, and the result lists each case's run and status); `start_prepared_run` works the same way.
- Rerun the whole set after any meaningful change, and report pass and fail per case to the person.
- Only run an evaluation of many cases when the person asks; it costs one run per case.

## Report

Tell the person, briefly: what you changed, which inputs you ran, what passed and what still fails, and what you would try next. Link each run you discuss with its `dashboard_url`, and the whole set with `run_test_cases`'s `dashboard_url` (the workflow's Runs tab). Use the links exactly as the tools give them; never build one.
