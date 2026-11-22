# DeepSieve recipes

## Free end-to-end rehearsal (do this first)

```bash
# 1. auth
curl -s $BASE/v1/me -H "Authorization: Bearer $DEEPSIEVE_API_KEY"

# 2. schema — every read names its Blueprint; take the active one
BP=$(curl -s $BASE/v1/blueprints -H "Authorization: Bearer $DEEPSIEVE_API_KEY" \
  | jq -r '.data[] | select(.is_active) | .id')
curl -s $BASE/v1/blueprints/$BP -H "Authorization: Bearer $DEEPSIEVE_API_KEY"

# 3. a free run that behaves like a real one (naming $BP checks the id before you pay)
LOC=$(curl -s -D - -o /dev/null -X POST $BASE/v1/research/runs \
  -H "Authorization: Bearer $DEEPSIEVE_API_KEY" -H 'Content-Type: application/json' \
  -d "{\"query\":\"anything\",\"dry_run\":true,\"blueprint_id\":\"$BP\"}" | grep -i '^location:' | tr -d '\r' | cut -d' ' -f2)

# 4. poll it exactly as you would a real run
until curl -s "$BASE$LOC" -H "Authorization: Bearer $DEEPSIEVE_API_KEY" | grep -q '"done":true'; do sleep 5; done
```

## A real run, retry-safe

```python
import os, time, uuid, httpx

BASE, KEY = os.environ["DEEPSIEVE_BASE"], os.environ["DEEPSIEVE_API_KEY"]
h = {"Authorization": f"Bearer {KEY}"}

# A run must name its Blueprint (missing = 400). Take the active one unless the human said which.
blueprints = httpx.get(f"{BASE}/v1/blueprints", headers=h).json()["data"]
BP = next(b["id"] for b in blueprints if b["is_active"])

# Cheaper than being refused: ask what it costs and whether the plan allows it.
est = httpx.post(f"{BASE}/v1/research/runs/estimate", headers=h,
                 json={"blueprint_id": BP, "depth": "standard"}).json()
# {"run_price_cents": 1000, "balance_cents": 16900, "affordable": true, "allowed": true, ...}
# `affordable: null` means the balance is UNKNOWN — never treat it as "no".
if est["allowed"] is False or est["affordable"] is False:
    raise SystemExit(est.get("reason") or "cannot afford this run")   # a human must act

r = httpx.post(f"{BASE}/v1/research/runs", headers={**h, "Idempotency-Key": str(uuid.uuid4())},
               json={"blueprint_id": BP, "query": "Solid-state battery startups in Japan",
                     "monitored": False})
if r.status_code == 402:            # insufficient_credits / plan_upgrade_required
    raise SystemExit(r.json()["error"]["message"])   # stop; a human must act
run = r.json()

while not run["done"]:              # 15-60 minutes; never hold a connection
    time.sleep(int(r.headers.get("Retry-After", 15)))
    run = httpx.get(f"{BASE}/v1/research/runs/{run['id']}", headers=h).json()
if run["status"] == "failed":
    raise SystemExit(run["error"]["message"])
```

## Read only what you can defend

```python
# Every read names its Blueprint. Take the active one unless the human said which.
blueprints = httpx.get(f"{BASE}/v1/blueprints", headers=h).json()["data"]
BP = next(b["id"] for b in blueprints if b["is_active"])

page = httpx.get(f"{BASE}/v1/blueprints/{BP}/entities/companies", headers=h,
                 params={"receipts": True, "limit": 100}).json()
if page.get("truncated"):
    print("PREVIEW ONLY —", page["truncated_reason"])   # tell the human

for row in page["data"]:
    rec_id = row["id"]
    for col, cell in row["citations"].items():
        if cell["verdict"] in ("contradicted", "unverifiable") or not cell["cited"]:
            print(f"flag for human: {row['id']}.{col} = {cell['value']!r} ({cell['verdict']})")
```

## Incremental sync

Store the last `updated_at` you saw, then poll `?updated_since=<iso>` on your
own schedule — or subscribe to `dataset.updated` and sync on the event.

## Verify a webhook (Python)

```python
import base64, hashlib, hmac, time

def verify(secret, headers, body: bytes) -> bool:
    ts = headers["webhook-timestamp"]
    if abs(time.time() - int(ts)) > 300:
        return False
    mac = hmac.new(secret.encode(), f"{headers['webhook-id']}.{ts}.".encode() + body,
                   hashlib.sha256)
    return hmac.compare_digest("v1," + base64.b64encode(mac.digest()).decode(),
                               headers["webhook-signature"])
```

## Ask the research agent about a cell

```python
run_id = run["id"]                  # the run from "A real run, retry-safe"
httpx.post(f"{BASE}/v1/research/runs/{run_id}/chat/messages", headers=h,
           json={"content": "Where did the 2024 funding figure come from? Re-check it.",
                 "row_ref": {"table_name": "companies", "record_id": rec_id}})
# poll GET /v1/research/runs/{run_id}/chat until the last agent message
# leaves status "streaming"; applied changes appear as revertible `actions`.
# A change with `status: "pending_confirmation"` was NOT written: read its
# `warnings`, tell your human, and only then POST /v1/chat/actions/{id}/confirm
# (or /dismiss). It is their data; never confirm a warning they have not seen.
```

## Teach DeepSieve what your human knows

Knowledge is what your human knows about the field: where the reliable data is, which
sources mislead, what a value means and its plausible range, what to leave out. Later runs
carry each entry wherever it applies. Write only what they told you, in their words.

```python
import uuid

KN = f"{BASE}/v1/blueprints/{BP}/knowledge"
entry = {"kind": "source_caution",
         "statement": "Crunchbase lags funding rounds by about a quarter.",
         "scope": {"level": "source", "hosts": ["crunchbase.com"]}}
r = httpx.post(KN, headers={**h, "Idempotency-Key": str(uuid.uuid4())}, json=entry)
if r.status_code == 409 and r.json()["error"]["code"] == "knowledge_conflict":
    # Nothing was saved. Show your human what it conflicts with; THEY choose.
    for c in r.json()["error"]["details"]["conflicts"]:
        print(f"conflicts with {c['id']}: {c['statement']!r} ({c['why']})")
    # Ask your human; they decide, never you.
    choice = input("Replace those entries (supersede), keep both (keep_both), or cancel? ")
    if choice not in ("supersede", "keep_both"):
        raise SystemExit("not saved")
    entry["on_conflict"] = choice              # "supersede" or "keep_both"
    r = httpx.post(KN, headers={**h, "Idempotency-Key": str(uuid.uuid4())}, json=entry)
r.raise_for_status()                           # any other refusal: read error.code, don't guess
kid = r.json()["entry"]["id"]

# Stop using it later (reversible), rather than deleting it:
httpx.post(f"{KN}/{kid}/retire", headers=h, json={"reason": "They now publish on time."})
```

**While a Blueprint is still a draft**, the same notes go on the draft and go live with it.
Anything your human's brief, answers or setup-chat messages already stated is captured for
you, in their own words (`origin.captured: true`): show those notes to them before you
approve. When the brief stated none, the draft asks `domain_knowledge`; ask them, and if they
have nothing to add, skip it rather than answering for them.

```python
SESSION = "<the draft id: `id` from POST /v1/blueprints/drafts>"
DR = f"{BASE}/v1/blueprints/drafts/{SESSION}"
entry = {"kind": "exclusion", "statement": "Leave out conversation-intelligence add-ons."}
r = httpx.post(f"{DR}/knowledge", headers={**h, "Idempotency-Key": str(uuid.uuid4())}, json=entry)
r.raise_for_status()                           # 409 knowledge_conflict: as above, THEY choose
for q in httpx.get(DR, headers=h).json()["follow_up_questions"]:
    said = input(q["question"] + " ").strip()  # ask your human, never answer for them
    if said:
        body = {"question_id": q["id"], "answer": said}
    elif q.get("skippable"):
        body = {"question_id": q["id"], "skip": True}
    else:
        continue                               # it needs an answer: ask them again later
    httpx.post(f"{DR}/answers", headers=h, json=body).raise_for_status()
    break                                      # an answer re-drafts: poll before the next
for n in httpx.get(f"{DR}/knowledge", headers=h).json()["items"]:
    print("from their words" if n["origin"].get("captured") else "added", "-", n["statement"])
```

## Show what knowledge did in a run

```python
k = httpx.get(f"{BASE}/v1/research/runs/{run_id}/knowledge", headers=h).json()
if not k["recorded"]:
    # The record was not kept. Say "not recorded", never zero, when the run had knowledge
    # (snapshot_entries > 0) or the Blueprint has some now.
    print("Your knowledge: not recorded for this run.")
else:
    # Three counts, never added: carried (in prompts), cited (QA's claim), decided (by code).
    print(f"decided {k['cells_decided']} cells; QA cited it on {k['cells_cited']}")

# Per cell: what an entry decided THERE, read from the citation, never the verdict note.
words = {e["id"]: e["statement"] for e in httpx.get(KN, headers=h).json()["items"]}
def say(where, d):
    other = d.get("run_id") and d["run_id"] != run_id
    said = words.get(d["id"], "an entry that has since been deleted")
    print(f"{where}: {d['outcome']} by {said!r}" + (" (in another run)" if other else ""))

for row in page["data"]:
    for col, cell in row["citations"].items():
        for d in cell.get("knowledge_decided") or []:          # decided on this value, by code
            say(f"{row['id']}.{col}", d)
        for b in cell.get("beside") or []:                     # decided on a row BESIDE it:
            for d in b.get("knowledge_decided") or []:          # say it about that row
                say(f"{row['id']}.{col} ({b['kind']} value)", d)
        for kid in cell.get("knowledge_cited") or []:          # QA's claim: never "decided"
            print(f"{row['id']}.{col}: QA cited {words.get(kid, kid)!r} in its verdict")
```
