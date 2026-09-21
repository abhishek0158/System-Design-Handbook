# Chapter 8 — Wallet / Ledger (money)

A wallet lets a user hold a balance and move money to other users. This is a common design
question, because it tests one thing above all: can you keep the numbers correct, forever, even
when things fail or retry.

## Requirements & Clarifying Questions

**Key features to support:**
- Each user has an account with a balance.
- A user can transfer money to another user.
- The system can show a user's current balance, fast.
- The system can show the full history of how a balance was reached (for support and audits).

**Clarifying questions to ask:**
- Is this a single currency, or multi-currency? (We assume each account holds one currency.
  A user with two currencies gets two account rows.)
- Do we handle real payment rails (banks, cards), or is this an internal wallet between users?
  (We assume internal wallet. External payment gateways are a separate, later topic.)
- Can a balance go negative (credit/overdraft), or must it always stay at or above zero? (We
  assume it must stay at or above zero, unless stated otherwise.)
- What happens if a transfer request is sent twice, by accident (a network retry)? This must not
  move money twice. (We design for this directly, below.)

**Rough scale:** Assume 5 million accounts and 2 million transfers a day. Balance reads (checking
your balance) happen far more often than transfers — maybe 50x more reads than writes.

## The Schema

```sql
CREATE TABLE accounts (
    account_id      BIGSERIAL PRIMARY KEY,
    owner_id        BIGINT NOT NULL,              -- the user who owns this account
    currency        TEXT NOT NULL,                -- e.g. 'USD', one currency per account
    balance_minor   BIGINT NOT NULL DEFAULT 0,    -- cached balance, in minor units (cents)
    version         BIGINT NOT NULL DEFAULT 0,    -- for optimistic locking, see below
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_balance_non_negative CHECK (balance_minor >= 0)
);
-- One row per (owner, currency). balance_minor is a CACHE of the ledger total, not the
-- source of truth. It exists so reads are fast.
CREATE INDEX idx_accounts_owner ON accounts(owner_id);

CREATE TABLE ledger_entries (
    entry_id         BIGSERIAL PRIMARY KEY,
    transfer_id      UUID NOT NULL,               -- groups the two rows of one transfer
    account_id       BIGINT NOT NULL REFERENCES accounts(account_id),
    amount_minor     BIGINT NOT NULL,              -- positive for credit, negative for debit
    entry_type       TEXT NOT NULL,                -- 'debit' or 'credit'
    idempotency_key  TEXT NOT NULL,                -- caller-supplied, unique per transfer
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
    -- ...more columns: description, related_transfer_id (for reversals)
);
-- Append-only. One transfer writes exactly TWO rows here: one debit, one credit.
-- Never updated, never deleted.
CREATE INDEX idx_ledger_account_created ON ledger_entries(account_id, created_at);
CREATE UNIQUE INDEX uq_ledger_idempotency ON ledger_entries(idempotency_key, entry_type);
```

**`accounts`** holds one row per user's balance in one currency. **`ledger_entries`** is the
append-only log of every money movement. Every real transfer is two rows in this table.

## Key Design Decisions — the Reasoning

**1. Double-entry: every transfer writes two ledger rows.**
Double-entry is an old accounting idea: money never appears or disappears, it only moves from
one place to another. So every transfer writes two rows in `ledger_entries` in the *same*
transaction: a **debit** (money leaves an account, negative amount) from the sender, and a
**credit** (money enters an account, positive amount) to the receiver. Both rows share the same
`transfer_id`.

Why this matters: at any time, you can add up all entries for a transfer, and they must sum to
zero. If they do not, you have a bug, and you can find it, because every movement is logged with
a matching pair. A design that only stores "account 5 now has $80" cannot tell you where the
missing $20 went. A ledger with matched debit/credit rows can. This is the main thing interviewers
check in this chapter: do you understand that money must move between two places, not just
appear or vanish at one place.

**2. The ledger is append-only and immutable. A mistake gets a reversing entry, never an edit.**
Append-only means rows are only inserted, never changed or deleted after that. Immutable means
once written, a row never changes.

Why: a ledger is a legal and financial record. If you could edit or delete a past entry, no one
could trust the history — a bug, or a bad actor, could quietly rewrite the past. Banks and
payment systems never do this. Instead, if entry was wrong, you add a **new** entry that reverses
it (a debit that cancels a credit, or the reverse), and optionally a follow-up correct entry. The
mistake stays visible in the log, along with its fix. This gives a full audit trail: anyone can
replay the entries and see exactly what happened, and when, and why (with a note field).

The trade-off: the ledger table only grows. It never shrinks. We accept this, because
correctness and trust matter more than storage cost (more on this in Scaling It).

**3. Money is an integer in minor units, never a float.**
Minor units means the smallest unit of a currency — cents for USD, paise for INR. `100` cents is
$1.00. We store `balance_minor` and `amount_minor` as integers (`BIGINT`), not `float` or
`double`.

Why: floats store numbers in binary fractions, and many decimal amounts (like 0.10) cannot be
stored exactly in binary. Add up enough of them, and small errors creep in — a cent goes missing
or appears from nowhere. In a wallet, that is unacceptable; the books must balance exactly, down
to the last cent. Integers have no rounding error under addition and subtraction. (`NUMERIC` with
a fixed number of decimal places is another safe option in Postgres, but plain integers in minor
units are simpler and just as safe, and are what most payment systems use.)

**4. The ledger is the source of truth. `accounts.balance_minor` is a cache, updated in the same
transaction.**
Source of truth means: if there is ever a disagreement, this is the value we trust. Here, that is
the sum of all `ledger_entries` rows for an account. In theory, you could compute a balance by
summing every entry for that account, every time someone asks. But that gets slow once an account
has thousands of entries.

So we keep `accounts.balance_minor` as a fast cache of that sum. The rule that keeps it correct:
every time we insert ledger entries for an account, we update that account's `balance_minor` in
the *exact same database transaction*. A transaction is a group of operations that either all
succeed together, or all fail together (see Chapter 2). If the ledger insert and the balance
update are in the same transaction, they can never go out of sync — either both happen, or
neither does. Reads stay fast (`SELECT balance_minor FROM accounts WHERE ...`), and writes stay
correct, because the cache can always be rebuilt from the ledger if you ever doubt it.

**5. A transfer is one atomic transaction, guarded by an idempotency key.**
Atomic means all-or-nothing: a transfer's steps (debit one account, credit another, update both
balances) either all happen, or none happen. If a server crashes halfway, we must not end up with
money removed from the sender but never added to the receiver. Wrapping all four writes (two
ledger inserts, two balance updates) in one database transaction guarantees this.

But atomicity alone does not stop a different problem: **retries**. If a client's network call
times out, it often retries the same request — but the first call may have actually succeeded on
the server. Without protection, this retry would create a second transfer and move the money
twice. An idempotency key fixes this: the caller generates one unique key per transfer attempt
(often a UUID) and sends it with the request. The server checks: "have I already processed this
key?" The `uq_ledger_idempotency` unique index enforces this at the database level — a second
insert with the same key fails, so the transaction can detect the duplicate and safely return the
original result instead of moving money again. This pattern (idempotency key + unique constraint)
is the standard fix for "retried request must not repeat a side effect," and it comes up often in
interviews beyond just wallets.

## Scaling It

**Reads are the hot path — they are already fast.** Checking a balance is just one indexed read
on `accounts`, by design (that is the whole point of the cache column). This scales well with
normal read replicas if traffic grows very high.

**Writes: partition by account to reduce contention.** Many transfers touching different
accounts can run in parallel. The only lock contention happens when many transfers hit the *same*
account at once (a popular merchant account, for example). If that becomes a bottleneck, you can
shard accounts across multiple database instances by `account_id` (see Chapter 2 on sharding), so
each shard only handles a slice of accounts and their transfers.

**The ledger grows forever — archive it.** `ledger_entries` never shrinks, so after a few years it
can hold billions of rows. Most reads only need recent history (say, the last 90 days) for a
statement view. Partition the table by `created_at` (monthly partitions), and move old partitions
to cheaper cold storage once they are old enough to rarely be read. Keep the partitions searchable
for audits, just not in the hot, expensive storage tier.

**Balance disputes: rebuild from the ledger.** If a cached balance is ever in doubt, you can always
recompute it by summing `ledger_entries` for that account. This is the safety net double-entry
gives you, and it is worth mentioning out loud in an interview: the cache can be wrong or lost,
but the ledger cannot lie, because it is append-only and every entry is paired.

## Interview Tips & Common Mistakes

- **Always say "double-entry" and explain it simply.** Many candidates jump straight to a single
  `balance` column and a single `UPDATE`. Say clearly: every movement is two rows, a debit and a
  credit, and they must sum to zero.
- **Never suggest editing or deleting a ledger row.** If asked "how do you fix a mistake," the
  answer is a reversing entry, not an `UPDATE` or `DELETE`. This is the number one thing
  interviewers listen for in a correctness-focused chapter like this one.
- **Never use `float` for money.** Say "integer minor units" or `NUMERIC` out loud, even if not
  asked. Using `float` is treated as a serious red flag in this round.
- **Do not forget the idempotency key.** If asked "what if the client retries the request," a
  design with no idempotency key will double-move money. Mention the unique constraint on the
  key as the enforcement mechanism, not just "the client should not retry."
- **Do not skip the transaction boundary.** Say explicitly which statements are inside the same
  atomic transaction (both ledger inserts and both balance updates). "I update the balance
  separately afterward" is a common mistake that breaks correctness under crashes.
- **Say the ledger is the source of truth, and the balance column is a cache.** This shows you
  understand *why* both exist, not just that both exist.
