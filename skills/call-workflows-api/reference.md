# Pulsar Workflows run API: reference

Base URL `https://workflows.pulsarbot.cloud/api/v1`. Every request sends `Authorization: Bearer <project key>`. Bodies are JSON.

## Endpoints

| Method and path | SDK | Answers |
|---|---|---|
| `POST /runs` | `runs.create(params, { idempotencyKey })` | `201` the run, `queued`. `200` the first run for a repeated `Idempotency-Key` with the same body. |
| `GET /runs/{id}` | `runs.get(run)`, `runs.wait(run)` | The run. While it is not finished, `Retry-After` says when to poll next (15 s queued, 5 s running). |
| `GET /runs?limit=&cursor=` | `runs.list()`, `runs.listAll()` | `{ object: "list", data, has_more, next_cursor }`, newest first. `limit` 1 to 50, default 20. |
| `POST /runs/{id}/cancel` | `runs.cancel(run)` | The run. A finished run comes back unchanged. |
| `DELETE /runs/{id}` | `runs.delete(run)` | `{ id, object: "run.deleted", deleted: true, files_deleted }`. Finished runs only. |
| `GET /runs/{id}/files/{path}` | `runs.downloadFile(run, file)` | The file's bytes, `application/octet-stream`. |

## Create body

| Field | | Limit |
|---|---|---|
| `workflow` | required | The slug. |
| `input` | required | A string up to 50,000 characters, or `[{ role: "user", content: [...] }]` with `{ type: "input_text", text }` and `{ type: "input_file", filename, file_data }` (base64) items. |
| `instructions` | | 8,000 characters. |
| `output_schema` | | `{ name, schema }`. 32 KB, 32 levels. |
| `metadata` | | 16 keys, values 512 characters. |
| `model` | | One the API offers; default the workflow's pin. |

Files: 5 per run, 10 MB each. The whole request: 25 MB. Header `Idempotency-Key`: up to 255 characters, matched for 24 hours.

## Other limits

| | |
|---|---|
| Creates per key | 60 a minute |
| Queued runs | 20 per key, 40 per organisation |
| Creates in flight | 4 per organisation at once |
| Run length | Stopped after 30 minutes (`timeout`) |
| Structured output | 1 MB |
| Output files listed | 100 files or 200 MB (`files_truncated`) |
| Files and inputs kept | 30 days after the run ends |
| Spend | A monthly cap per organisation (`usage_limit`) |

## API error codes

HTTP errors are `{"error": {"code", "message"}}`. The SDK raises the same codes as `PulsarWorkflowsError.code`.

| Code | Status | When | What to do |
|---|---|---|---|
| `invalid_request` | 400 | A field is missing or wrong; the message names it. | Fix the field. |
| `invalid_output_schema` | 400 | The schema uses something the API refuses. | Simplify the schema. |
| `model_unavailable` | 400 | Not an offered model. | Leave `model` out. |
| `unauthorized` | 401 | Missing, malformed, unknown or revoked key. | Check the key. |
| `org_disabled` | 403 | The organisation's access is off. | Contact Pulsar. |
| `workflow_not_found` | 404 | No workflow with this slug in the key's project. | Check the slug and that the key is from the workflow's project. |
| `run_not_found` | 404 | No such run in the key's project. | Check the id. |
| `file_not_found` | 404 | No such file, the run is not finished, or the file expired. | Use a `url` from the finished run. |
| `idempotency_conflict` | 409 | Same `Idempotency-Key`, different body. | New job, new key. |
| `run_live` | 409 | Deleting a run that is still going. | Cancel, wait, then delete. |
| `request_too_large` | 413 | Body over 25 MB. | Fewer or smaller files. |
| `rate_limited` | 429 | Too many creates. `Retry-After`. | Wait; the SDK retries. |
| `queue_full` | 429 | Too many queued runs. | Wait for queued runs to start. |
| `usage_limit` | 429 | Monthly spend cap reached. `Retry-After` is the start of next month (UTC). | Stop and tell someone. |
| `internal_error` | 500 | Failure on Pulsar's side. | Retry later. |
| `sandbox_unavailable` | 503 | Busy. `Retry-After`. | Wait; the SDK retries. |

SDK-only codes (status `0`): `network_error` (no answer), `timeout` (no answer within the client's `timeoutMs`, default 60 s), `wait_timeout` (`wait` ran out; the run continues, `err.run` is the last state), `unexpected_response` (not the API's JSON: check the base URL).

## Run error codes

On a finished run that did not complete, `run.error.code`:

| Code | Meaning |
|---|---|
| `timeout` | The run took too long and was stopped. |
| `interrupted` | Interrupted before it could finish. |
| `invalid_output` | The output did not match `output_schema`, even after a correction pass. |
| `output_too_large` | The output was too large to return. |
| `usage_limit` | The monthly cap was reached mid-run. |
| `connection_missing` | An app the workflow needs is not connected in the project. |
| `stopped_unexpectedly` | The run stopped unexpectedly. |
| `sandbox_unavailable` | The run's environment was not available. |
| `cancelled` | Cancelled. |
| `key_revoked` | The key was revoked, so the run was stopped. |

`interrupted`, `stopped_unexpectedly` and `sandbox_unavailable` are worth one retry with the same input (under a new idempotency key). The others need a change first: to the input, the schema, the project's connections or its budget.

## SDK client options

`new PulsarWorkflows({ apiKey, baseUrl, fetch, maxRetries, maxRetryDelayMs, timeoutMs })`: key from `PULSAR_WORKFLOWS_API_KEY`, base URL from `PULSAR_WORKFLOWS_BASE_URL`; `maxRetries` 2 (3 tries), `maxRetryDelayMs` 60000, `timeoutMs` 60000 per try. Every method takes `{ signal }` last; aborting rethrows your abort reason, not a `PulsarWorkflowsError`.

`wait(run, { pollMs, timeoutMs, onUpdate, signal })`: `timeoutMs` default 15 minutes, `Infinity` waits forever. Polls never closer than 1 s.

Exports: `PulsarWorkflows`, `PulsarWorkflowsError`, `isPulsarWorkflowsError`, `isTerminal`, `DEFAULT_BASE_URL`, and the types `Run`, `RunStatus`, `RunFile`, `RunError`, `RunErrorCode`, `RunList`, `DeletedRun`, `CreateRunParams`, `CreateRunOptions`, `InputFile`, `OutputSchema`, `WaitOptions`, `ErrorCode`, and more.
