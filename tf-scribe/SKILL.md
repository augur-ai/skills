---
name: tf-scribe
description: Author Testfold test suites, cases and API test collections. Use when the user asks to create, add, draft or generate test cases or a test suite, to turn requirements into test cases, or to write or import API tests (requests with checks) for a service. Reads what already exists first, so it extends instead of duplicating.
---

<!-- tf-skill-version: 3 -->

# Author Testfold test cases

Four tools for suites and cases: `active_project`, `context`, `propose`, `create`. Three for API
tests: `api_context`, `api_create`, `api_import` — see section 4.

## 0. Who drafts? Decide this first.

- The user asked for **Testfold's agent** — "use Testfold", "let Testfold draft these", "generate
  these from this spec/PRD" → call **`propose`**. It runs inside the app, on the app's own model, and
  can read requirement documents.
- Otherwise → **you draft them yourself** from `context` and the repository in front of you. This is
  the default. It runs on the user's own subscription, so do not spend the app's agent uninvited.

Either way the drafts land in the same place: you review them with the user, then call `create`.

## 1. Ground yourself before writing anything

Call **`context`**. It returns every existing suite with its cases, plus the closed set of case
`kinds` and `max_drafts`.

Read it, then say what you found — how many suites, and which one this work belongs in. Two things
follow, and neither is optional:

- **Extend, do not restate.** A case whose title already exists in the target suite is noise. Say
  "suite X already covers Y" rather than proposing Y again.
- **Pick the suite deliberately.** A new suite or an existing one, and why. If it is genuinely
  ambiguous, ask — one question, naming the candidates you saw.

`active_project` says which project the app is pointed at. Call it when the user asks, or to name the
project back to them before writing. Everything else is scoped to it automatically; there is no
project argument to pass.

## 2a. Draft them yourself (the default)

Write cases from what the repository and the user actually tell you. Each one gets a `title`, a
`description`, and a `kind` from the set `context` returned.

Ground every case in something you can point at — a route, a handler, a validation rule, a stated
requirement. A case invented to round the list out to ten is a case somebody has to triage later.

## 2b. Or ask Testfold's agent

Call **`propose`** with the `suite_id`. Optionally `source_docs` (absolute paths on this machine, or
library entries from the app), `kinds`, `count`, and `instructions` — the user's steer in their own
words, not your paraphrase of it.

It returns drafts. **They are drafts, not results.** Nothing is persisted until `create`.

## 3. Review with the user, then create

Show the drafts — title, kind, one line of why each is there. Let the user cut or edit. Then call
**`create`** with the ones that survived:

- `suite_id` for an existing suite, or `name` to make a new one. One or the other, not both.
- `drafts` — only the kept ones.

`create` reports `created`, `skipped` (with the duplicate title it matched) and `attached`. **Read
that response and relay it.** If something was skipped, say which and why. If `attach_error` is set,
the cases exist but are not in the suite — say so plainly rather than reporting a clean success.

## 4. API tests — requests with checks, from the code in front of you

When the user wants API tests for a service (a route, a handler, an OpenAPI file, "test this
endpoint"), the unit is an **API collection**: requests, each with plain checks, run by the app
against an environment and recorded in Test Runs.

1. Call **`api_context`** first: existing collections and their requests, the environments (a
   `baseUrl` each), the datasets a run can iterate, and the check forms. Do not recreate a
   collection that exists; do not write a request a collection already has.
2. Write the requests yourself from the handlers, routes and validation you can read — a request
   per behaviour worth checking, including the ones that should fail (a 404, a 422 on a bad body).
   Use `/path` URLs so the environment's `baseUrl` decides the host. Put secrets in as
   `{{secret:<ref>}}`; never a literal token.
3. Checks are plain lines, one per check: `status 201`, `body.id exists`, `body.total = 3`,
   `header Content-Type contains json`, `time < 500`. A line outside these forms REFUSES the create
   and is quoted back — fix it, do not drop the check.
4. Call **`api_create`** with `name` and `requests`. Relay the result: the collection's name and
   the `run` line that says where to run it. Running is the user's act in the app (API Tests →
   Run collection); the run then appears in Test Runs.

**`api_import`** is for a collection that already exists on disk — a Bruno or OpenCollection
folder in the repository. Pass its absolute `path`; a `.bru` folder is converted on import. A file
Bruno cannot read refuses the whole import and is named — say which.
