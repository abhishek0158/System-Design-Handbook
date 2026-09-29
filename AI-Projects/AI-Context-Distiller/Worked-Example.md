# Worked Example — one task, from raw repo to a minimal, ranked context bundle

*This shows Context Distiller working on one concrete task, on a small example repo. The task:
**"Add rate limiting to the login endpoint."** We follow the data through every offline stage
(built once) and every online stage (run for this one task), and end with a real token count
comparison. Simple English.*

> Read this next to `Project-Playbook.md` Part C. There, each stage is explained. Here, you see
> each stage actually happen on real code.

---

## Step 0 — The example repo (the input)

Imagine a small Python shop built with Flask. In real life this repo has around 40 files and
roughly **60,000 tokens** of source code in total. We only show the files that matter for this
task.

**`app.py`** (the web routes)
```python
from flask import Flask, request, jsonify
from services.auth import login_user
from services.checkout import place_order   # unrelated to this task

app = Flask(__name__)

@app.post("/login")                          # line 6
def login():                                 # line 7
    data = request.get_json()
    result = login_user(data["email"], data["password"])
    if result["ok"]:
        return jsonify(result), 200
    return jsonify(result), 401
```

**`services/auth.py`** (the login logic)
```python
from db import get_user_by_email
from utils.security import check_password, create_session

def login_user(email, password):            # line 4
    """Check a user's password and start a session if it is correct."""
    user = get_user_by_email(email)
    if not user or not check_password(password, user["password_hash"]):
        return {"ok": False, "error": "invalid credentials"}
    session = create_session(user["id"])
    return {"ok": True, "session": session}
```

**`services/api_rate_limiter.py`** (already exists, used only for the public data API — not
called from anywhere near login today)
```python
import time

class RateLimiter:
    """A simple fixed-window rate limiter backed by Redis."""
    def __init__(self, redis_client, max_hits=100, window_seconds=60):
        self.redis = redis_client
        self.max_hits = max_hits
        self.window_seconds = window_seconds

    def allow(self, key: str) -> bool:
        """Return True if 'key' is still under its hit limit for this window."""
        window = int(time.time() // self.window_seconds)
        redis_key = f"rl:{key}:{window}"
        hits = self.redis.incr(redis_key)
        if hits == 1:
            self.redis.expire(redis_key, self.window_seconds)
        return hits <= self.max_hits
```

There are 37 other files (checkout, inventory, admin, reporting, and so on) that have nothing
to do with login or rate limiting. A naive assistant does not know that in advance — it tends
to read far more than these three files "just in case."

---

## PHASE 1 — Offline preprocess (built once, before this task ever arrives)

### Stage 1 — Parse (text → tree)
tree-sitter turns `services/auth.py` into a tree. Simplified view:
```
file: services/auth.py
 └─ function_definition  name="login_user"  start_line=4  end_line=10
      docstring: "Check a user's password and start a session if it is correct."
      body:
        ├─ call  name="get_user_by_email"  line=6
        ├─ call  name="check_password"     line=7
        └─ call  name="create_session"     line=9
```
Same is done for every file, including `api_rate_limiter.py` (which defines a class
`RateLimiter` with methods `__init__` and `allow`).

### Stage 2 — Build graphs (who calls who)
| Node | Symbol | File | Lines |
|---|---|---|---|
| N1 | `login` | app.py | 7–12 |
| N2 | `login_user` | services/auth.py | 4–10 |
| N3 | `get_user_by_email` | db.py | 3–6 |
| N4 | `check_password` | utils/security.py | 5–7 |
| N5 | `create_session` | utils/security.py | 9–13 |
| N6 | `RateLimiter.allow` | services/api_rate_limiter.py | 10–16 |

Call graph edges:
```
N1 login       ──calls──> N2 login_user
N2 login_user  ──calls──> N3 get_user_by_email
N2 login_user  ──calls──> N4 check_password
N2 login_user  ──calls──> N5 create_session
```
Note: **N6 (`RateLimiter.allow`) has no edge to N1 or N2.** Nothing near login calls it today.
A pure call-graph search would miss it completely — this is exactly why Stage 6 (embeddings)
exists.

### Stage 3 — Rank (PageRank)
```
PageRank scores (0-1, higher = more central):
  N2 login_user          0.041   <- called by the route, calls 3 things
  N4 check_password      0.028   <- called from several auth-related places
  N3 get_user_by_email   0.019
  N5 create_session      0.017
  N6 RateLimiter.allow   0.006   <- rarely called, low centrality, but relevant by meaning
```
`login_user` ranks high because it sits at the centre of the login flow. `RateLimiter.allow`
ranks low on structure alone — it will only surface through semantic search, not through rank.

### Stage 4 — Repo map (skeletonization — signatures only, no bodies)
```json
[
  { "symbol": "login",              "file": "app.py:7",
    "sig": "def login()", "doc": null, "rank": 0.033 },
  { "symbol": "login_user",         "file": "services/auth.py:4",
    "sig": "def login_user(email, password)",
    "doc": "Check a user's password and start a session if it is correct.",
    "rank": 0.041 },
  { "symbol": "RateLimiter.allow",  "file": "services/api_rate_limiter.py:10",
    "sig": "def allow(self, key: str) -> bool",
    "doc": "Return True if 'key' is still under its hit limit for this window.",
    "rank": 0.006 }
]
```
The full repo map for all 40 files, signatures + one-line docstrings only, comes to about
**1,800 tokens** — versus about **60,000 tokens** for the raw source of the whole repo. That is
roughly **33x smaller**, in line with the 10–30x we expect from skeletonization.

### Stage 5 — Hierarchical summaries (cached, written once by Claude)
**Function-level summary (`login_user`):**
```json
{ "symbol": "login_user", "file": "services/auth.py",
  "summary": "Looks up a user by email, checks their password, and creates a session on success. Returns an error dict on bad credentials. Has no limit on how many attempts a caller can make.",
  "content_hash": "e91f2a" }
```
**File-level summary (`services/auth.py`):**
```json
{ "file": "services/auth.py",
  "summary": "Holds the login flow: looks up the user, verifies the password, and starts a session. No throttling or lockout logic lives here today." }
```
**Module-level summary (`services/`):**
```json
{ "module": "services/",
  "summary": "Business logic for auth, checkout, inventory, and the public API. api_rate_limiter.py provides a Redis-backed RateLimiter class, currently only wired into the public API routes." }
```
Notice the function summary for `login_user` already flags, in plain words, *"no limit on how
many attempts"* — this is exactly the gap the task wants filled, and it was written once,
offline, long before this task came in.

### Stage 6 — Embeddings (semantic index)
Each summary and function body becomes a chunk, embedded (with Voyage AI or
sentence-transformers) and stored in the vector index (Chroma/FAISS). A search for the meaning
of "limit how many times something can happen" returns:
```
Semantic search: "rate limiting login attempts"
  1. services/api_rate_limiter.py :: RateLimiter.allow      (score 0.87)
  2. services/api_rate_limiter.py :: RateLimiter.__init__   (score 0.81)
  3. services/auth.py             :: login_user              (score 0.74)
```
This is how `RateLimiter.allow` gets found for this task, even though the call graph (Stage 2)
never connects it to login. Structure and meaning cover two different kinds of relevance.

*(Stage 7, incremental update, is not shown here — it only matters on the next commit, after
this task's code change is merged.)*

---

## PHASE 2 — Online agentic retrieval (runs now, for this one task)

### Stage 1 — Receive the task
```
Task: "Add rate limiting to the login endpoint."
```
This one sentence is all the loop starts with.

### Stage 2 — Plan
The LLM reads the task plus the small, always-included top-level repo map (module names and
one-line summaries — about 300 tokens). It replies:
```
Plan:
- The login endpoint is likely in a routes file (app.py) calling into an auth module.
- Look at the login route and its login function/service.
- Check if a rate limiter already exists anywhere in the repo before building a new one.
```
Cost so far: about **350 tokens** (top-level map + task + plan reply).

### Stage 3 — Retrieve (first pass, small bundle)
Guided by the plan, the loop pulls the smallest useful slice:
```json
{
  "signatures": [
    { "symbol": "login", "file": "app.py:7", "sig": "def login()" },
    { "symbol": "login_user", "file": "services/auth.py:4",
      "sig": "def login_user(email, password)" }
  ],
  "summaries": [
    { "symbol": "login_user",
      "summary": "Looks up a user by email, checks their password, and creates a session on success. ... Has no limit on how many attempts a caller can make." }
  ]
}
```
No real code bodies yet — just signatures and the cached summary. Running total: about
**650 tokens**.

### Stage 4 — Check ("is this enough?")
```
LLM verdict: MORE: I know login_user has no rate limiting, but I need (1) the real code of
the login route and login_user, so I know exactly where to add the check, and (2) whether a
rate limiter already exists anywhere in this repo that I should reuse instead of writing a
new one.
```
Not enough yet — the loop expands.

### Stage 3 again — Retrieve (expand)
This time the loop fetches two more things, exactly as asked:
- **Graph neighbors:** the real bodies of `login` (app.py) and `login_user` (services/auth.py),
  since those are the two nodes the plan already pointed at.
- **Semantic match:** the vector index query "existing rate limiter" returns
  `RateLimiter.allow` and `RateLimiter.__init__` from `services/api_rate_limiter.py` — found by
  meaning, not by any call-graph edge.

```json
{
  "code": [
    { "symbol": "login", "file": "app.py:7-12", "body": "...(shown in Step 0 above)..." },
    { "symbol": "login_user", "file": "services/auth.py:4-10", "body": "...(shown in Step 0 above)..." },
    { "symbol": "RateLimiter", "file": "services/api_rate_limiter.py:1-16", "body": "...(shown in Step 0 above)..." }
  ]
}
```
Running total: about **1,900 tokens**.

### Stage 4 again — Check
```
LLM verdict: ENOUGH. I can see the login flow, and I can see a reusable RateLimiter class
already backed by Redis. I will call RateLimiter.allow(email) at the start of login_user.
```
The loop stops here — no third pass needed.

### Stage 5 — Assemble (final bundle)
De-duplicate (drop the earlier plain summary of `login_user` now that we have its real body),
rank by relevance, and trim to the token budget:
```json
{
  "task": "Add rate limiting to the login endpoint.",
  "bundle": [
    { "symbol": "login",        "file": "app.py:7-12",                      "kind": "code" },
    { "symbol": "login_user",   "file": "services/auth.py:4-10",            "kind": "code" },
    { "symbol": "RateLimiter",  "file": "services/api_rate_limiter.py:1-16","kind": "code" }
  ],
  "total_tokens": 2050
}
```

### Stage 6 — Serve
The MCP server returns this bundle to the coding assistant. The assistant now writes the
actual change — for example, wiring `RateLimiter.allow(f"login:{email}")` into `login_user`
before the password check, reusing the existing Redis-backed class instead of writing a new
one. That correct, reused-code answer was only possible because Stage 6 (embeddings) surfaced
`RateLimiter` even though the call graph never pointed to it.

---

## Token count comparison — naive vs. distilled

| Approach | What gets sent to the LLM | Tokens |
|---|---|---|
| **Naive** | The whole repo dumped in (or, more realistically, every file in `app.py`, `services/`, `db.py`, `utils/` "just in case") | ~60,000 |
| **Distilled (this tool)** | Top-level map (350) + first-pass signatures/summary (300) + expanded real code for 3 relevant symbols (1,400) | ~3,000 (2,050 kept in the final bundle; ~950 spent on the plan/check LLM calls along the way) |

Both approaches, given to Claude, arrive at the **same correct answer**: reuse
`RateLimiter.allow`, call it inside `login_user` before checking the password. The distilled
approach reaches it for about **1/20th the tokens**, because it never had to read the 37
unrelated files, and it stopped asking for more context the moment it had enough.

---

## The whole transformation on one line (for your team)

```
raw repo (~60,000 tokens)
  → Offline 1-3: parsed, graphed, ranked           (structure, not yet compact)
  → Offline 4:   repo map, no bodies                (~1,800 tokens for the whole repo)
  → Offline 5:   cached summaries                   (read a sentence, not the function)
  → Offline 6:   embeddings                         (finds RateLimiter even with no call-graph link)
  → Online 1-2:  task + plan                        (~350 tokens)
  → Online 3-4:  retrieve -> check -> expand -> check ("not enough" -> "enough", 2 passes)
  → Online 5-6:  minimal bundle served via MCP       (~2,050 tokens, same correct answer)
```
Nothing about the answer's correctness was sacrificed. Every extra token the naive approach
spent was on code the task never actually needed.
