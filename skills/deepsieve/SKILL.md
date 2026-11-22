---
name: deepsieve
description: Run cited deep research and read the resulting dataset through the DeepSieve API. Use when the user asks to research companies/people/markets into structured rows, to check where a stored fact came from, to keep a dataset fresh, or to query/export a DeepSieve dataset. Covers auth, running research (free test mode first), reading cited data, saved reports, webhooks, and chatting with the research agent.
license: Apache-2.0
metadata:
  version: "0.1.0"
  homepage: "https://deepsieve.ai"
---

# DeepSieve

DeepSieve turns a research question into **structured rows where every cell
carries a confidence verdict, and cited cells carry their sources**, and keeps
them fresh. You drive it over a REST API.

## Setup (once)

If you're being added over MCP with `--transport http`, the user logs in
through the browser and you're authenticated — no key needed; skip to the
capabilities below. That session always gets one fixed set of permissions: it
can run research, read data and reports, and create or propose Blueprints, but
never apply a Blueprint change, manage keys or touch billing (`scopes` on
`GET /v1/me` lists them). Otherwise (REST calls, or MCP in CI), the user creates a
key at `/settings/api-keys` (**Agent** preset — it can run
research and read results, but can never edit their schema or billing). Store
it as `DEEPSIEVE_API_KEY`; never commit it. Every request:

```
Authorization: Bearer ds_live_...
```

Base URL is the user's deployment origin. Verify before anything else:
`GET /v1/me` → `{"object":"identity", ...}`. It also carries `plan`,
the two balances (`balance_cents` prepaid dollars, `plan_run_credits` the
plan's included runs — spent first) and `capabilities` (`max_research`,
`webhooks`, `audit_log`, `teammates`, `priority_queue`, `sso`, `white_label`,
`blueprint_limit`) — check those BEFORE a run rather than
discovering a limit from a 402 mid-workflow. A `null` balance means UNKNOWN,
never zero. A 401 means the key is wrong; `key_revoked`/`key_expired` mean
they need to mint a new one. `key_suspended` / `account_suspended` (403) mean the
abuse guard paused the credential, or (signed in as an AI agent through OAuth)
the person's agent access — **stop and tell a human; do NOT mint a new key** (a
new key of a suspended user is refused too, and minting into it escalates). The
403's message says which. `suspended` (+ `suspended_reason`) on `/v1/me` says so
up front.

## The five things you'll actually do

**1. Discover the schema — never guess it.** `GET /v1/blueprints` lists the
Blueprints and marks the active one; `GET /v1/blueprints/{blueprint_id}`
returns that one's customer-defined entities and columns. They differ per Blueprint and can
change. Hardcoding column names is the most common way an integration breaks.

**If a response warns `parameter_renamed`** (a `Deprecation`/`Sunset` header, or a `warnings`
entry), you sent an old parameter name: `workspace_id` is now `blueprint_id`. Switch now; after the
`Sunset` date the old name is `400 parameter_renamed`, so resend with `blueprint_id`. **Ask your human
before changing saved skills or instructions** that use `workspace_id`: they are theirs to change.

**Name the Blueprint you mean.** Runs, estimates and webhooks REQUIRE
`blueprint_id`: without it the call is `400 invalid_request` with
`param: "blueprint_id"` and nothing is charged (only a `dry_run` may omit it).
Reports take it too. Their responses, chat, the monitoring overview and `GET /v1/me`
carry `blueprint: {id, name}`: check it is the one you meant before you act on
the answer, because a call on the wrong Blueprint looks exactly like a
successful one. Check before a run spends, on the estimate or a `dry_run`.

**2. Test for free before spending.** `POST /v1/research/runs` with
`{"query": "...", "dry_run": true}` behaves exactly like a real run (202 +
`Location`, poll to completion in ~15s) but costs nothing and persists
nothing. Build your whole create → poll → parse loop here. Send `query` OR
`seeds`, never both; send neither and the run sweeps whatever the
Blueprint is about (the run reports what it chose as `derived_brief`).

**3. Run real research — and tell the user what it costs.** Drop `dry_run`
and send `blueprint_id`.
A run bills prepaid credits (typically $10, or $20 at `"depth":"max"`) and
takes **15–60 minutes**. Always send an `Idempotency-Key` header so a retry
can't double-bill. Poll `GET /v1/research/runs/{run_id}` every ~15s until
`"done": true`; never hold a connection open. `"monitored": true` is the
default and bills ongoing per-target monitoring — pass `false` if the user
only wants a one-off.

**When your human names the pages.** If they list specific pages and say to use only
those, pass their words as the `query` (labels too: `P01: https://…`). Each URL then
stays its own target, in their order and under their label, and the 202 says
`seed_mode: "as_typed"`. A `seeds` request instead reads each page to find what to
research (`"discover"`); `seed_labels` (`{url: label}`) still names its targets.
Before spending on URL seeds, an estimate with `check_urls: true` reads each one and
says which find nothing (`url_outcomes`). It is opt-in: each URL uses your
organization's model allowance.

**4. Read the cited dataset.**
`GET /v1/blueprints/{blueprint_id}/entities/{entity_key}?receipts=true`.
Each cell carries `{value, confidence, verdict, cited, source_urls, ...}`.
**Read the verdict before you trust the value** — `contradicted`,
`unverifiable` or `cited: false` means surface it to the human, don't state
it as fact. Sync incrementally with `?updated_since=` or the cursor.
If the response says `"truncated": true` you have a *preview*, not the
dataset — say so rather than analyzing it as complete.

**5. Ask the research agent.** `POST /v1/research/runs/{run_id}/chat/messages`
when the user asks "why does this say X?" or wants a value re-checked. Your
messages appear in the human's own chat history, attributed to your API key.
Every change proposed in chat comes back `status: "pending_confirmation"` with
`warnings`, however clearly you asked: nothing was written. Show your human the
warnings; only when they decide,
`POST /v1/chat/actions/{action_id}/confirm` (or `/dismiss`). Never confirm a warning
they have not seen.

## Rules that keep you out of trouble

- **Never edit the active schema.** You may create a *new* Blueprint
  (`POST /v1/blueprints/drafts` → review draft → `chat` → approve), but
  editing a live one is human-only and will be refused. Propose changes to
  the user instead.
- **Undoing a merge, rename or successor link changes the customer's data.**
  `GET /v1/changes` lists them; `POST /v1/changes/{change_id}/undo` undoes one
  (`data:write`, member role). Confirm with your human first, and send an
  `Idempotency-Key` so a retry is safe.
- **Domain knowledge is your human's expertise, not yours.** The app calls it **Custom
  Guidance** (called *knowledge* in the API). What they know about the field (where
  reliable data is, which sources mislead, what a value means, what to leave out) can be
  saved on the Blueprint so later runs carry it:
  `POST /v1/blueprints/{blueprint_id}/knowledge` (`knowledge:write`, member role). Write
  only what they stated, in their words, or what you verified from a source; knowledge
  guides research and is never evidence. A `409 knowledge_conflict` saved nothing: show
  them `error.details.conflicts` and resend with `on_conflict` as they decide. Retire an
  entry rather than delete it. Reading what it did (`GET /v1/research/runs/{run_id}/knowledge`,
  and `knowledge_decided` / `knowledge_cited` on a cell's citation): **cited is QA's claim,
  decided is what code did; never add them or call a citation a decision**, and a run
  whose record was not kept is "not recorded", never zero. Across recent runs
  (`GET /v1/blueprints/{blueprint_id}/knowledge/usage`), each entry has a `state`: say it
  in words ("decided 4 times across 3 of the last 20 runs", "nothing in its scope came up",
  "not checked: left out by the length limit"), never as a bare zero, and never call
  `decisions` cells. Before suggesting an entry is dead weight, check it was checked.
- **A cell your human set on purpose wins over research.** Research never
  overwrites it; what it finds since is kept beside it. `GET /v1/holds?pending=true`
  lists those cells with the newer value (`pending_finding`). Show it to your
  human; `POST /v1/cells/{cell_id}/apply` takes it (with the `finding_evidence_id`
  you showed them), `POST /v1/cells/{cell_id}/keep` keeps their value and dismisses
  it (research does not ask again unless it finds a different value or a new page),
  `POST /v1/cells/{cell_id}/release` stops protecting the cell. All three need
  `data:write` and the member role. Never act on a finding they did not see.
- **Merging rows is the same: preview it and confirm with your human.**
  `POST /v1/merges/preview`, then `POST /v1/merges` with the preview's `preview_digest`,
  so it applies exactly what they saw (`409 merge_changed` means it changed: show them
  `error.details.preview` again). Never settle a tie for them: send each tie's choice as
  they decided it.
- **Removing a wrong many-to-many link is reversible, and research respects it.**
  `POST /v1/links/remove` with `entity`, `record_id`, `to_entity` and `target_id` takes one
  linked record out of a record's links; if it was the primary link, the next one becomes
  primary. Research does not add it back. The answer is a `link_removed` change: undo it
  with `POST /v1/changes/{change_id}/undo`. Needs `data:write` and the member role; send an
  `Idempotency-Key`. Tell your human what you removed.
- **Turning a report's monitoring back on after its free 30 days commits 30 days
  of spend.** `PATCH /v1/research/runs/{run_id}/monitoring` with `{"active": true}`
  is refused `409 lock_confirmation_required` until you send `"commit_lock": true`;
  `error.details.locked_until` says when the 30 days would end (the last billed day
  is the day before it). Tell your human that date and send `commit_lock` only after
  they agree, with `expected_locked_until` set to that same date. Never send it on your
  own judgement. If the answer is `409 lock_quote_changed`, a UTC midnight passed while
  they decided: nothing was committed, so ask them again with the new date.
- **Retrying a failed run can charge for a new run.** `POST /v1/research/runs/{run_id}/retry`
  is refused `409 retry_confirmation_required` when the retry would charge, and nothing
  is created. `error.details` is the quote: `total_price_cents` (the list price), the
  `refund` of the failed run, what is carried over or skipped at no charge, and
  `funding.covered_by` (`free_run`, `plan_credits`, `balance` or `mixed`), what would actually pay.
  Tell your human what pays, the total and the refund, and resend with `expected_total_cents` set to that total
  and `expected_funding` set to that `covered_by` only after they agree, so a retry they agreed to
  pay with plan credits is never paid from the balance instead. For `mixed` (plan credits for part
  of the run, the balance for the rest) also send `expected_prepaid_cents` = `funding.prepaid_cents`,
  the balance's part they agreed to. Never confirm it on your own
  judgement. If the answer is `409 retry_price_changed`, the price or what pays moved while they
  decided: nothing was charged, so ask them again with the new quote. A retry that charges nothing starts at once.
- **Approving a report's waiting re-research spends money.** Monitoring can leave
  re-research waiting for approval once a report's included re-checks are used.
  `GET /v1/research/runs/{run_id}/reruns` lists it: `items`, `awaiting_total_cents` and a
  `quote_id`. Show your human the items and the total, and only after they agree send
  `POST …/reruns/approve` with that `expected_total_cents` and `quote_id`; the approval
  buys exactly the rows they saw. `409 rerun_price_changed` means what is waiting changed:
  nothing was charged, so read the list again and ask again. There is no decline: a rerun
  left waiting costs nothing. `PATCH …/reruns/auto-approve` with `{"auto_approve": true,
  "auto_approve_cap_cents": …}` is standing authority to spend up to that cap a month on
  the report's future re-research: turn it on only when your human asks, with the cap they
  chose. It is recorded against your key and switches off if the key is revoked. Turning
  it off never spends.
- **Errors are typed.** Every failure is `{"error":{code, message, retriable,
  request_id}}`, plus `details` when there is a fact to act on (such as `locked_until`, or a retry's quote). If `retriable` is false, do not retry — fix the request or
  tell the human. `503 maintenance` means the API is down for scheduled maintenance: tell
  the human, and retry after `Retry-After` (`details.expected_back_at` says when, if set).
  `plan_upgrade_required` means the PLAN lacks the capability —
  topping up will not help, so do not retry; a human must upgrade. If
  `GET /v1/me`'s `subscription.trial.eligible` is true, tell them a free trial is open to
  them (`offer_days` long) and hand them `{app_url}/pricing`: checkout takes a card.
  A `subscription.status` of `none` is not always a new org: with `expired_from_plan` set
  its term ended (read and export only, no new runs), and with `retention_until` set it is
  closed and is deleted on that date (once it has passed, the research is gone). Tell your
  human which, and the date.
  `insufficient_credits` / `budget_exceeded` mean *stop and
  ask a human*, never retry in a loop.
- **Respect the limits.** `RateLimit-*` headers are on every response; honor
  `Retry-After` on 429.
- Keys are `/v1`-only; the `/api/*` routes are the web app's private surface.

## Going deeper

`references/endpoints.md` — the full endpoint list with request/response
shapes. `references/recipes.md` — worked examples (first run, incremental
sync, webhook verification, saved reports). The live contract is always at
`/openapi.json`, and any docs page is available as markdown by appending
`.md` to its URL.
