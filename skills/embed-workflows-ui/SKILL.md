---
name: embed-workflows-ui
description: Use when a web app's users should start Pulsar workflows themselves and see the progress and the result in the app's own UI (a "Run" button, a form that kicks off a workflow, a results panel, a list of past runs). Covers the server route that keeps the API key out of the browser (`createWorkflowsHandler` from `@starlightsearch/workflows/server`), per-user scope, the browser routes, following a run's progress, showing the result and files, and the React UI kit `@starlightsearch/workflows-react`.
---

# Put Pulsar workflows in a web app's UI

The goal: a person using someone's app presses a button, a Pulsar workflow runs, the app shows how far it got, then the answer and the files. Three pieces:

1. **A server route** in the app's backend, `createWorkflowsHandler`. It holds the API key and decides who may run what and see which runs.
2. **Browser code** that calls that route: start a run, follow it, show the result.
3. Optionally **the React UI kit**, which is that browser code already written.

For server-only use (a job, a script, a backend that runs a workflow and stores the answer), use the `call-workflows-api` skill instead: no UI kit, no handler.

## Rule one: the key never reaches the browser

A project key (`pk_live_...`) can start, read, cancel and delete every run in its project. Never put it in client code, a `NEXT_PUBLIC_*` / `VITE_*` variable, a bundle, local storage or a response. Never call `https://workflows.pulsarbot.cloud/api/v1` from the browser. The browser talks only to the app's own route; the route adds the key.

Keep the key in the server's env as `PULSAR_WORKFLOWS_API_KEY`. Ask the person for it (a project admin creates it on the project's **Keys** page); don't invent one.

## 1. The server route

Needs `@starlightsearch/workflows` with the `/server` entry (the release after 0.2.0; check `npm view @starlightsearch/workflows version` and the package README for "Browser apps").

```ts
// Next.js App Router: app/api/pulsar/[...path]/route.ts
import { createWorkflowsHandler } from "@starlightsearch/workflows/server";

export const { GET, POST } = createWorkflowsHandler({
  apiKey: process.env.PULSAR_WORKFLOWS_API_KEY!,
  workflows: ["invoice-summary"],            // slugs this app offers; never "*" for a public app
  authorize: async (req) => {
    const user = await getSessionUser(req);  // the app's own auth
    if (!user) return false;                 // 401
    return { scope: { user_id: user.id } };  // whose runs these are
  },
});
```

Express or Node: `app.use("/api/pulsar", toNodeHandler(handler))` (same entry). Other runtimes with `Request`/`Response` (Hono, Bun, Deno, Remix, SvelteKit): call `handler.handle(request)`.

### Scope decides who sees which runs

`authorize` returns `false` or `{ scope, workflows? }`.

- `scope` is up to 4 metadata keys, e.g. `{ user_id }` or `{ org_id, user_id }`. The handler writes them onto every run the caller creates (overriding anything the browser sent) and answers **404 `run_not_found`** for any run that doesn't carry them. Pick the key that matches who should see a run: one person (`user_id`), a team (`team_id`), a record (`ticket_id` plus `user_id`).
- `{ scope: {} }` shows every run the key can see. Only for a single-user internal tool. Getting this wrong leaks one user's results to another, so ask if unsure.
- `workflows` narrows the allow-list per caller (e.g. admins get more workflows).
- `onRunCreated(run, req)` is where to store `run.id` against the app's own record.

### What the route exposes

Fixed routes, relative to the mount. Anything else is 404, and there is no delete.

| Route | Use |
|---|---|
| `POST /runs` | Start a run. `multipart/form-data`: `workflow`, `input` (text), `files` (0-5, 10 MB each), optional `metadata` (JSON object of strings). Send `Idempotency-Key: <uuid>` (one per submit) so a retry or double click can't start two runs. |
| `GET /runs/:id` | The run now, plus `Retry-After` (seconds to wait before asking again) |
| `GET /runs` | The caller's runs, newest first. `limit`, `cursor`, `workflow` |
| `POST /runs/:id/cancel` | Cancel |
| `GET /runs/:id/files/*` | A file the run wrote. Use `run.files[i].url` as given: it already points here |
| `GET /runs/:id/events` | Live events. 404 on today's API: poll `GET /runs/:id` |
| `GET /workflows/:slug` | The workflow's inputs and output. 404 on today's API |

Errors are JSON `{ error: { code, message } }`. The ones to handle in UI: `invalid_request` (400, show the message), `unauthorized` (401, sign in), `workflow_not_found` / `run_not_found` (404), `request_too_large` (413), `rate_limited` / `queue_full` (429, wait `Retry-After`), `insufficient_credits` (402, the workspace is out of credit; an admin tops up), `server_key_rejected` (502: the server's key is wrong; a deploy problem, not the user's).

## 2. The browser side

### With React: `@starlightsearch/workflows-react`

Check it exists first (`npm view @starlightsearch/workflows-react`). If it does not yet, write the hook in "Without the kit" below.

```tsx
import { WorkflowsProvider, WorkflowRunner } from "@starlightsearch/workflows-react";
import "@starlightsearch/workflows-react/styles.css";

<WorkflowsProvider endpoint="/api/pulsar">
  <WorkflowRunner workflow="invoice-summary" onComplete={(run) => save(run.output)} />
</WorkflowsProvider>
```

- `<WorkflowRunner>`: form, live progress, result, Cancel, Run again. `fields` gives separate inputs instead of one text box.
- `<RunStatus runId variant="pill" | "bar" | "steps">`, `<RunResult run>`, `<RunList onSelect>` when the app has its own form or layout.
- `useWorkflowRun("slug")` returns `{ status, run, steps, error, start, cancel, reset }` for fully custom UI; `useWorkflowRun({ runId })` reattaches after a reload.
- `<RunResult showCaveats>` also shows the run's assumptions and "not verified" notes. Off by default; turn it on for analyst-style users who need to judge the answer.
- Style: the kit is neutral black and white. Brand it with CSS variables (`--pw-accent`, `--pw-radius`, `--pw-font`), or pass `unstyled` and `classNames` for Tailwind/shadcn.

### Without the kit (other frameworks, or before it ships)

Follow these rules; they are what the kit does:

1. **Start**: `POST /runs` with `FormData`. Make the idempotency key once per submit, keep it if the request fails, and reuse it on retry. Disable the button while the request is out.
2. **Follow**: `GET /runs/:id` until `status` is one of `completed`, `failed`, `cancelled`, `incomplete`. Wait `Retry-After` seconds between polls (15 while `queued`, 5 while `in_progress`), never less than 2 s. Pause while `document.hidden` and do one fetch when the tab is visible again. Keep `run.id` in the URL or storage so a reload can reattach.
3. **Progress**: `run.steps` is `{ done, total, current }`. Show a bar `done / total` and `current` as the label. `total` can be 0 early on: show an indeterminate bar.
4. **Result**: `run.output` is the structured answer (when the workflow has an output schema), `run.output_text` is markdown (render it with HTML disabled; it is model output), `run.files` are downloads, `run.assumptions` are guesses the run made.
5. **Failure**: `run.error.message` is safe, fixed wording per `run.error.code`. Show it as is, or map codes to the app's own copy. `connection_missing` means an app the workflow needs isn't connected in the Pulsar project: tell the app's admin, not the end user.

```ts
async function follow(id: string, onRun: (r: Run) => void, signal: AbortSignal) {
  const done = new Set(["completed", "failed", "cancelled", "incomplete"]);
  for (;;) {
    const res = await fetch(`/api/pulsar/runs/${id}`, { signal });
    if (!res.ok) throw new Error((await res.json()).error?.message ?? `HTTP ${res.status}`);
    const run = await res.json();
    onRun(run);
    if (done.has(run.status)) return run;
    const wait = Math.max(2, Number(res.headers.get("retry-after")) || 5) * 1000;
    await new Promise((r) => setTimeout(r, wait));
  }
}
```

## Choosing what to show

- A run takes seconds to minutes (hard stop 30 minutes). Don't block the page: show progress and let people leave and come back (reattach by id, or a `<RunList>`).
- Show `steps.current`: people trust a run they can watch.
- Show only the result to most users; assumptions and caveats are opt-in.
- Each run uses the Pulsar workspace's credit. For anything public, rate-limit per user in `authorize` (return `false` past the limit) and keep the allow-list short.

## Testing

- Test the route with a fake `fetch` passed as the `fetch` option: assert the key is sent upstream, the scope is stamped on create, and a run with another user's metadata answers 404 on get, cancel and files.
- In UI tests, stub the app's own `/api/pulsar/*` routes, never Pulsar.
- For a real end-to-end check, run one cheap workflow with a test key and make sure the run appears in the workflow's run history in Pulsar with source "API".

## Checklist

- [ ] Key only in server env (`PULSAR_WORKFLOWS_API_KEY`); nothing in client code or public env vars
- [ ] `authorize` returns a real per-user (or per-team) scope; `{}` only for a single-user tool
- [ ] Allow-list names the workflows this app offers
- [ ] Start sends one `Idempotency-Key` per submit
- [ ] Polling honours `Retry-After`, stops at a terminal status, pauses when hidden
- [ ] `output_text` rendered as markdown with HTML disabled
- [ ] Failure shows `run.error.message`; `server_key_rejected` and `connection_missing` reach the developer or admin, not end users
