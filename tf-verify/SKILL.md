---
name: tf-verify
description: Verify a change against stated intent, not just "does it still build", or find out why something is failing. Use when the user asks to verify, check, or validate a change or a pull request (a PR link or number), before opening a pull request, or when they hand you an incident or a bug report (often as a Slack, ticket or PR link) and ask what is wrong. Establishes what should be true, works out what it can reach, and checks it against evidence.
---

<!-- tf-skill-version: 11 -->

# Verify a change with Testfold

Testfold runs the verification: it reads the change, replays the recorded flows the change reaches,
drives the app, and reports each criterion with evidence. Your job is to give it what it needs,
follow the run, and relay what it found. Do not run the checks yourself instead.

## 0. Resolve every link first

When the user gives you a link — a Slack thread, a ticket, a pull request, a doc — read it with the
tools you have (a Slack, Jira, Linear or GitHub MCP, or `gh`) before anything else. Testfold cannot
open it. Put what it says into the run: the goal in the user's words plus the thread's content —
who reported what, when it started, which customer or records, and what has been tried — in the
`goal` or as an attachment. If a link cannot be read, say so and ask the user to paste it; never
start a run on a link you did not read.

## 1. What should be true. Ask if you do not have it.

You need the acceptance criteria: **what should be true after this change that is not true now?**
If the user has not said, ask them, in one question:

> What should be true after this change that isn't true now?

Also capture the feature or area, and any ticket reference, from what the user gave you and the
links you resolved in step 0. Do not infer criteria from the diff and present them as the user's. A
criterion you invented is a claim you are grading yourself against.

For an incident or a bug report the criterion is the question itself, taken from the report:
"Name why some people cannot book a class since the morning deploy, and reproduce it with evidence",
provenance `extracted_from_goal`.

**Verifying a pull request** ("verify PR #42", or the change has one): pass it as `pull_request`
(step 3). Its title, description and commit titles are read for you and added to the run, in
addition to whatever the user tells you. If the user gave no criteria, call `verify` with the PR
anyway: it answers `needs_intent` with `proposed_criteria` restated from the PR's own words. Show
them to the user, and call `verify` again with the ones they accept — unedited ones keep provenance
`from_pull_request`, anything they reword or add is `stated_by_user`.

Record where each criterion came from: `stated_by_user`, `from_pull_request`, `from_branch`,
`extracted_from_goal`, `inferred_from_diff`, or `unknown`. Criteria inferred from the diff alone are
refused.

## 2. Where to run it. Ask unless the user said.

> Should I run this against your local stack, or a shared environment?

- **A change that is not pushed exists only on this machine.** Run it locally. A shared
  environment is running its last deploy, so a run there checks code that is not the change.
- Name an environment from the repository's `tfold.yaml` (its `metadata.name`, such as `local`), or
  pass its base URL.
- Never choose one silently. With no environment, the run falls back to the repository's
  pull-request default, which is usually a shared one.
- **An incident happens somewhere already.** Use the environment the report is about, from
  `tfold.yaml`, and set `deploys_environment: true` when the run should stand up its own copy there
  and restore the affected customer's data into it. The system that is failing is never driven.
- **Happened in one place, reproduced in another:** when the report says where it happened
  (production) and where to reproduce it (staging), the run stands the second one up and restores a
  masked subset of the affected customer's data into it — pass the reproduction environment with
  `deploys_environment: true`. Its readiness checks are what the run fills; an empty staging is
  expected, not a reason to stop.
- **A remote machine travels with its environment.** When the report names the environment and
  its address, use them as given. Read `tfold.yaml` only from the repository you are working in —
  never search the disk for it — for how that environment's work runs (`target`: host, user, the
  key's reference — never the key). Before you start the run, tell the user which environment,
  its address and the machine. Pass it as the same Environment document Testfold exports, so the
  call says where the work happens and which apps exist:
  `{"kind": "Environment", "metadata": {"name": ..., "displayName": ...}, "spec": {"runbook": ...,
  "target": {"kind": "ssh", "host": ..., "user": ..., "credential": ..., "workingDir": ...},
  "connections": {"web": {"type": "web", "config": {"baseUrl": ...}}}}}`.

## 3. Start the run

**Publish to the pull request? Ask before the run, never after.** When the change has an open pull
request (the user linked it, or `gh pr view --json url` on the branch finds one), ask:

> Post the result as a comment on PR #42 when the run finishes?

Set `publish: true` only on a yes. That yes is the approval: nothing asks again, and the comment is
posted when the run ends. On a no, leave `publish` out; the PR still goes in as `pull_request`.

Call `verify` with:

- `goal`: what the user asked for, in their words.
- `acceptance_criteria`: from step 1, each with its provenance.
- `repos`: the absolute path of every service the change spans, not just the one you edited.
- `environments`: the answer from step 2.
- `base`: when the change is one commit or a range, the commit before it (for example `HEAD~1`).
  Omit it for everything since the default branch.
- `triggered_by`: your own name, for example `claude-code`.
- `kind`: what the work is, when you know it from the ticket or the conversation: `feature`,
  `bug`, `incident` (a live service hurting its users now) or `dependency`. A report of wrong
  behaviour on a local or test app is a `bug`.
- `pull_request`: `https://github.com/<owner>/<repo>/pull/<number>`, when the change has one. Its
  branch is what is verified for that repository.
- `publish`: `true`, only after the user said yes above.

`verify` computes the blast radius and the prior failures itself. Do not call `blast_radius` first.
Tell the user the session id, so they can watch the run in Testfold. If `verify` answers
`cannot_publish` or `cannot_read_pull_request`, tell the user its `reason` and ask how to go on.

## 4. Follow it

Poll `verify_status` about every 30 seconds until the run finishes. When `pending_approvals` is not
empty, the run is blocked on you: show the user what it wants to do and answer with
`verify_approve`. Never approve anything destructive on their behalf.

A run started with `publish` is shared to its project when it ends, and `verify_status` then answers
`status: shared` with `publication`: `comment_url` once the comment is posted, or a `message`
saying why it was not. Give the user the link, or that message.

## 5. Report per criterion, not one global PASS

Read `findings` in the finished `verify_status`: each criterion with `pass`, `fail` or `unknown`,
and the checks behind it — what was expected, what was requested, the status code, what came back,
and why a check was not counted. For each criterion say **Pass**, **Fail** or **Unknown**, with that
evidence, and for the last one, why. Then say which recorded flows
were replayed and how each went. A flow the change broke is a result, not noise. When the status
has `noticed`, list each one with the app's own words quoted: things a user would struggle with that
no criterion asked about. Finish with anything the run reached that nothing verified.

## 6. When a criterion fails, fix it and verify again

If the user asked you to make the change, a criterion that fails is yours to fix:

1. Fix the code from what the failing check saw, not from a guess. Do not change a criterion to
   make it pass.
2. Commit, and rebuild the environment from that commit. A run against a build that is not the
   commit is refused.
3. Call `verify` again with the same arguments, and report the second run the same way. Keep
   `publish` only if the user wants this run's result posted too; otherwise leave it out.

Stop after two rounds that still fail, and hand the findings to the user. An `unknown` is not
a failure to fix in code: report what blocked it.

## 7. Close the loop where it started

When the run came from a thread or a ticket, draft a short reply for it: the cause in one sentence,
the evidence it rests on, and the Testfold run. Show the draft to the user and post it only when they
approve. Never post on your own.

## Only asked what a change could break?

Call `blast_radius` with the absolute path of the repository, and a `scope` when the change is
narrower than the working tree. Read its `notes` before the rest. Rows marked `uncovered` are
reached by the change with nothing testing them; say so plainly.
