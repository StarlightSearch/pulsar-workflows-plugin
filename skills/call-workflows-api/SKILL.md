---
name: call-workflows-api
description: Use when code (an app, a backend job, a script, a CI step) needs to run a Pulsar workflow and use its answer, through the TypeScript SDK `@starlightsearch/workflows` or the HTTP API with curl. Covers keys and slugs, starting a run, waiting for it, structured output, sending and downloading files, idempotent retries, errors, cancel, delete and listing.
---

# Call Pulsar Workflows from code

A Pulsar workflow runs in Pulsar's sandbox. Code starts a **run** of it, waits for the run to end, and reads the answer and the files it wrote. There is one resource, the run, at `https://workflows.pulsarbot.cloud/api/v1/runs`.

- **TypeScript or JavaScript**: use the SDK, `@starlightsearch/workflows`. It retries what is safe to retry, honours `Retry-After`, encodes files and types the output. Don't hand-roll fetch calls in a TS project.
- **Anything else** (shell, Python, Go, CI): call the HTTP API. The curl examples below give the exact request shapes.

Limits, every error code and the run error codes are in [reference.md](reference.md).

## Before writing code: the slug and the key

1. **The workflow's slug.** It is the API name, e.g. `quote-prep`, and never changes after creation. Get it from `list_workflows` (the `slug` field) if you have the pulsar-workflows tools, or from the workflow's **API** tab in Pulsar. Don't use the display name or the workflow id.
2. **A project API key** (`pk_live_...`). Only a project admin can create one, on the project's **Keys** page; ask the person for it. A key acts as its project: runs use that project's workflows, connections, knowledge and spend, and only that project's workflows can be run with it.
3. **Keep the key in the environment**, as `PULSAR_WORKFLOWS_API_KEY`, which the SDK reads by itself. Never write it into source, a test fixture, a commit or browser code: anyone with the key can start runs on the project. Call the API from a server, never from a browser.
4. `PULSAR_WORKFLOWS_BASE_URL` points at another deployment (self-hosted, local). Leave it unset for Pulsar's cloud. Use `http://` only for localhost.

If the workflow needs connected apps or secrets, they must be set up in the project first. A run of a workflow whose apps are not connected fails with `connection_missing`.

## Run a workflow and use the answer

### TypeScript (SDK)

```sh
npm install @starlightsearch/workflows
```

```ts
import { PulsarWorkflows } from "@starlightsearch/workflows";

const client = new PulsarWorkflows(); // reads PULSAR_WORKFLOWS_API_KEY

const run = await client.runs.create({
  workflow: "quote-prep",
  input: "Prepare a quote for 40 units of part A-113.",
});
const done = await client.runs.wait(run);

if (done.status === "completed") {
  console.log(done.output_text);
} else {
  throw new Error(`Run ${done.id} ${done.status}: ${done.error?.code} ${done.error?.message}`);
}
```

- `create` answers at once with the run `queued`. `wait` polls until the run ends and **resolves whatever the outcome**: it does not throw for a failed run. Always check `status`.
- A run can take minutes (the hard stop is 30 minutes). `wait` gives up after 15 minutes by default with a `wait_timeout` error, and the run keeps going. Set `timeoutMs` to what the caller can afford, and don't hold a web request open for a run: start it, store `run.id`, and finish the work in a job or a later poll.
- `onUpdate` shows progress: `wait(run, { onUpdate: (r) => log(r.status, r.steps.done, r.steps.total, r.steps.current) })`.
- Pass the run or its id anywhere a run is asked for: `client.runs.get("run_...")`.

### curl

```sh
curl https://workflows.pulsarbot.cloud/api/v1/runs \
  -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"workflow": "quote-prep", "input": "Prepare a quote for 40 units of part A-113."}'
```

That answers `201` with the run (`"status": "queued"`). Poll `GET /runs/{id}` and wait the `Retry-After` it sends (15 s queued, 5 s running) between polls:

```sh
RUN_ID=run_...   # the "id" from the create
while :; do
  RUN=$(curl -s https://workflows.pulsarbot.cloud/api/v1/runs/$RUN_ID \
    -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY")
  STATUS=$(echo "$RUN" | jq -r .status)
  [ "$STATUS" = queued ] || [ "$STATUS" = in_progress ] || break
  sleep 5
done
echo "$RUN" | jq '{status, output_text, error}'
```

The final statuses are `completed`, `failed`, `cancelled` and `incomplete` (treat it as failed). Any other status means the run is still going.

## The run object

| Field | |
|---|---|
| `id` | `run_...` |
| `status` | `queued`, `in_progress`, then `completed`, `failed`, `cancelled` or `incomplete`. |
| `output_text` | The final answer as text, once the run has ended. |
| `output` | The structured result, only when an output schema applied and the result matched it. |
| `assumptions` | What the agent had to assume because the input did not say. Log them: each one is a gap in the input or the workflow. |
| `files` | `{ path, size, url }[]`, the files the run wrote. Empty until the run has ended. `files_truncated: true` means it wrote more than 100 files or 200 MB. |
| `steps` | `{ done, total, current }`: progress through the workflow's steps. |
| `error` | `{ code, message }` when the run did not complete. |
| `metadata` | Your own strings, returned as sent. |
| `workflow`, `model`, `created_at`, `started_at`, `completed_at` | Times are ISO 8601, UTC. |

## Structured output

Send a JSON Schema when the calling code needs fields, not prose. The API checks the result against it; a mismatch gets one correction pass, then the run fails with `invalid_output`. `output` is set only when the result matched.

```ts
type Quote = { currency: string; total: number; lines: { part: string; qty: number }[] };

const run = await client.runs.create<Quote>({
  workflow: "quote-prep",
  input: "40 units of A-113",
  outputSchema: {
    name: "quote",
    schema: {
      type: "object",
      properties: {
        currency: { type: "string" },
        total: { type: "number" },
        lines: {
          type: "array",
          items: {
            type: "object",
            properties: { part: { type: "string" }, qty: { type: "integer" } },
            required: ["part", "qty"],
          },
        },
      },
      required: ["currency", "total", "lines"],
    },
  },
});
const done = await client.runs.wait(run); // Run<Quote>
if (done.status === "completed" && done.output) console.log(done.output.total);
```

- The generic is a promise to TypeScript. The SDK does not validate again, so keep the type and the schema in step (or derive one from the other).
- The schema is JSON Schema 2020-12, at most 32 KB and 32 levels deep. These are refused (`invalid_output_schema`): `pattern`, `patternProperties`, `format`, `$dynamicRef`, `$id`, and a `$ref` outside the schema. `uniqueItems` needs `maxItems` of 1000 or less beside it. `name` is 1 to 64 letters, digits, `_` or `-`.
- Over HTTP the field is `output_schema` (snake case): `"output_schema": {"name": "quote", "schema": {...}}`.
- A workflow may declare its own default schema in its SKILL.md. A schema you send overrides it.

## Files in

Up to 5 files, 10 MB each, and the whole request under 25 MB. The workflow sees each file by its name.

```ts
import { readFile } from "node:fs/promises";

const run = await client.runs.create({
  workflow: "contract-review",
  input: "Review this contract against our checklist.",
  files: [
    { name: "contract.pdf", data: await readFile("./contract.pdf") },
    { name: "notes.txt", data: "Customer asked for net 60." },
  ],
});
```

`data` is a `Uint8Array` (a Node `Buffer` is one), an `ArrayBuffer`, a `Blob` or `File`, or a string. **A string is the file's text, not base64**: the SDK does the encoding. `input` may be `""` when files carry the request.

Over HTTP, `input` becomes an array with one user item whose `content` mixes text and base64 files. Build the body with `jq`, so a large file never goes through a shell argument:

```sh
base64 < contract.pdf | tr -d '\n' > contract.b64
jq -n --rawfile pdf contract.b64 '{
  workflow: "contract-review",
  input: [{ role: "user", content: [
    { type: "input_text", text: "Review this contract against our checklist." },
    { type: "input_file", filename: "contract.pdf", file_data: $pdf }
  ]}]
}' | curl https://workflows.pulsarbot.cloud/api/v1/runs \
  -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY" \
  -H "Content-Type: application/json" \
  -d @-
```

## Files out

```ts
import { mkdir, writeFile } from "node:fs/promises";
import { dirname, join } from "node:path";

for (const file of done.files) {
  const bytes = await client.runs.downloadFile(done, file); // Uint8Array
  const target = join("./out", file.path);
  await mkdir(dirname(target), { recursive: true });
  await writeFile(target, bytes);
}
```

`file.path` can hold folders (`reports/q4.pdf`). Over HTTP, `url` is a path on the base URL and needs the key:

```sh
curl https://workflows.pulsarbot.cloud/api/v1/runs/$RUN_ID/files/reports/q4.pdf \
  -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY" -o q4.pdf
```

Files and inputs are kept for 30 days after the run ends. Download what you need to keep.

## Retrying safely: idempotency keys

A create that fails on the network may or may not have started a run. Send an idempotency key and the create becomes safe to retry: the same key with the same body within 24 hours returns the first run (`200`) instead of starting another.

```ts
const run = await client.runs.create(
  { workflow: "quote-prep", input: "Order 8812", metadata: { order: "8812" } },
  { idempotencyKey: "quote-order-8812" },
);
```

Over HTTP it is the `Idempotency-Key` header (up to 255 characters).

- Use a key that names the job (an order id, a ticket id plus the workflow), never a random value made per attempt: that defeats the point. To run the same job again on purpose, use a new key.
- The same key with a different body is `409 idempotency_conflict`.
- With a key, the SDK retries the create on network errors, 429 and 503. **Without one it never retries a create.** Every job that must not run twice, such as one that sends email or writes to another system, needs a key.

## Errors

The SDK throws `PulsarWorkflowsError` for every failure. Switch on `code`, which is stable; `message` is for people.

```ts
import { isPulsarWorkflowsError } from "@starlightsearch/workflows";

try {
  const run = await client.runs.create({ workflow: "quote-prep", input });
  return await client.runs.wait(run, { timeoutMs: 10 * 60_000 });
} catch (err) {
  if (!isPulsarWorkflowsError(err)) throw err;
  switch (err.code) {
    case "usage_limit": // monthly spend cap: err.retryAfter seconds until it frees up
    case "queue_full": // too many queued runs: try again later
      return scheduleRetry(err.retryAfter);
    case "wait_timeout": // the run is still going: err.run is the last state seen
      return rememberRun(err.run!.id);
    default:
      throw err;
  }
}
```

- HTTP errors are JSON: `{"error": {"code": "...", "message": "..."}}`, with `Retry-After` on 429 and 503.
- A run that **ended badly is not an exception**. It is a resolved run with `status: "failed"` and `error.code` (`timeout`, `invalid_output`, `connection_missing`, `usage_limit`, ...). Handle both paths.
- The SDK already retries network errors, `rate_limited` and 503 on reads, `wait`'s polls, `cancel`, downloads, and creates that carry an idempotency key. Don't wrap those calls in another retry loop. It never retries `usage_limit`, `queue_full` or a `Retry-After` over 60 s: it throws with `retryAfter` set, so your code decides.
- Status and code types are open-ended: keep a `default` branch so a code added later still compiles and is handled.
- `requestId` is on the error when the API sent one. Log it; support can trace it.

## Cancel, delete, list

```ts
await client.runs.cancel(run); // stops a queued or running run; a finished one comes back unchanged
await client.runs.wait(run); // let it settle
await client.runs.delete(run); // deletes output, files and inputs; id, status and times stay

for await (const r of client.runs.listAll()) console.log(r.id, r.status); // newest first, every API run of the project
```

```sh
curl -X POST https://workflows.pulsarbot.cloud/api/v1/runs/$RUN_ID/cancel -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY"
curl -X DELETE https://workflows.pulsarbot.cloud/api/v1/runs/$RUN_ID -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY"
curl "https://workflows.pulsarbot.cloud/api/v1/runs?limit=20" -H "Authorization: Bearer $PULSAR_WORKFLOWS_API_KEY"
```

Delete refuses a run still going (`409 run_live`): cancel it first. A list page has `data`, `has_more` and `next_cursor`; pass `cursor=<next_cursor>` for the next page (`limit` 1 to 50).

## Other create fields

- `instructions`: extra instructions added after the workflow's own, up to 8,000 characters. Use it for per-call context ("reply in German"). If every caller needs it, it belongs in the workflow.
- `metadata`: up to 16 string keys, values up to 512 characters, returned on the run. Put your own ids here (`{ order: "8812" }`) so a run can be traced back.
- `model`: leave it out. The workflow's pinned model, or the product default, is what the workflow was tested on. A model the API does not offer is `model_unavailable`.
- Unknown fields are refused with `invalid_request`, and the message names the field.

## Testing code that calls the API

- Unit tests: pass a fake `fetch` to the client (`new PulsarWorkflows({ apiKey: "pk_test", fetch: fakeFetch })`), or mock your own wrapper around the client. Don't start real runs from unit tests: every run spends the project's budget.
- One end-to-end check against the real workflow, with a real key from the environment, shows the slug, key and schema are right. Read the run's `output_text`, `output` and `assumptions` before trusting it.
- If the answer is wrong rather than the call, the fix is in the workflow, not the calling code. Use the build and test workflow skills.

## Checklist

- [ ] Slug from `list_workflows` or the API tab, not the display name.
- [ ] Key read from `PULSAR_WORKFLOWS_API_KEY`, never in source or the browser.
- [ ] `status` checked after `wait`; `failed` handled with `error.code`.
- [ ] `timeoutMs` set to what the caller can afford; long runs are stored by id and picked up later.
- [ ] An idempotency key on every create that must not run twice.
- [ ] `output_schema` when the code needs fields, with the TS type kept in step.
- [ ] Output files downloaded within 30 days if they are needed.
- [ ] Errors switched on `code`, with a default branch.
