# Chapter 20 — Design Case Studies (worked)

This chapter works through six classic design questions end to end. For each one we set
requirements, sketch the entities, write the schema, add indexes, and defend the trade-offs.
The goal is not a perfect design. The goal is a design you can explain and defend, which is
what the design round tests. Say your assumptions out loud, reason about trade-offs, and be
clear when the honest answer is "it depends" (see Chapter 18 for the general approach).

## Case Study 1 — E-commerce

Reuse the shared E-commerce schema from the brief. The interesting parts are **price at the
time of order** and **stock (inventory) correctness** under many buyers at once.

**Requirements**

- Store customers, products, and orders made of many items.
- An order must keep the price the customer actually paid, even if the product price changes later.
- Two customers must not oversell the last unit in stock.
- Read a customer's order history fast.

Clarifying questions to ask the interviewer:

- Do we track stock per product only, or per warehouse or variant (size, color)?
- Can prices change over time, and must old orders show the old price? (Almost always yes.)
- Do we need to hold stock in a cart before payment (a reservation), or only reduce it at checkout?
- What order volume and read/write mix should I design for?

**Entities & Relationships**

```
customer 1 ---- * order 1 ---- * order_item * ---- 1 product
                                                    product 1 ---- 1 inventory
```

One customer has many orders. One order has many order_items. Each order_item points to one
product. Each product has one inventory row that holds the current stock count.

**Schema**

```sql
CREATE TABLE customers (
    customer_id  BIGSERIAL PRIMARY KEY,
    name         TEXT NOT NULL,
    country      TEXT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
    product_id   BIGSERIAL PRIMARY KEY,
    name         TEXT NOT NULL,
    category     TEXT,
    price        NUMERIC(12,2) NOT NULL CHECK (price >= 0)  -- current list price
);

CREATE TABLE inventory (
    product_id   BIGINT PRIMARY KEY REFERENCES products(product_id),
    stock        INTEGER NOT NULL CHECK (stock >= 0)        -- CHECK blocks overselling
);

CREATE TABLE orders (
    order_id     BIGSERIAL PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(customer_id),
    order_date   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status       TEXT NOT NULL DEFAULT 'PENDING',
    total_amount NUMERIC(12,2) NOT NULL DEFAULT 0
);

CREATE TABLE order_items (
    order_id     BIGINT NOT NULL REFERENCES orders(order_id),
    product_id   BIGINT NOT NULL REFERENCES products(product_id),
    quantity     INTEGER NOT NULL CHECK (quantity > 0),
    unit_price   NUMERIC(12,2) NOT NULL,   -- price COPIED at order time
    PRIMARY KEY (order_id, product_id)
);
```

The key decision is `unit_price` on `order_items`. We copy the product price into the order
line when the order is placed. We do **not** join to `products.price` at read time. If we did,
a later price change would silently rewrite order history. Money must be frozen at sale time.
`NUMERIC` (fixed-point decimal) is used for money, never `FLOAT`, because floating point rounds
in ways that lose cents.

**Indexes**

```sql
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date DESC);
CREATE INDEX idx_order_items_product  ON order_items (product_id);
```

The first index serves "show me this customer's recent orders", the most common read. The
composite order `(customer_id, order_date DESC)` lets one index filter and sort at once. The
second index helps "which orders included product X" for reporting. The `order_items` primary
key `(order_id, product_id)` already covers "give me all items in this order".

**Normalization / Denormalization decisions**

- The schema is normalized: product facts live once in `products`, order facts in `orders`.
- `unit_price` looks like duplication but is **not** a normalization mistake. It is a
  point-in-time fact, different from the current price. Storing it is correct.
- `total_amount` on `orders` is a denormalized cache of `SUM(quantity * unit_price)`. We keep
  it so order lists do not re-sum items every time. The risk is drift: it can go out of sync
  with the lines. We keep it correct by only writing it inside the same transaction that writes
  the items.

**Handling stock correctly**

The classic bug: two buyers both read `stock = 1`, both think they can buy, both decrement, and
stock goes to -1. Fix with a conditional update inside the checkout transaction:

```sql
BEGIN;
UPDATE inventory SET stock = stock - 1
 WHERE product_id = 42 AND stock >= 1;
-- if 0 rows updated, stock ran out -> ROLLBACK and tell the user
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (1001, 42, 1, (SELECT price FROM products WHERE product_id = 42));
COMMIT;
```

The `WHERE stock >= 1` makes the check and the update one atomic step. The row lock the
`UPDATE` takes serializes the two buyers, so only one wins. The `CHECK (stock >= 0)` is a final
safety net. This is cleaner than reading, deciding in application code, then writing — that
read-then-write pattern has a race (see Chapter 11 on locking).

If the interviewer adds a cart that must **hold** stock before payment, do not decrement real
stock yet. Add a `reservations(product_id, quantity, expires_at)` row and count reserved units
against available stock. A background job releases expired holds. This keeps a slow checkout
from locking real inventory, and it separates "reserved" from "sold". Mention this only if asked
— it adds real complexity.

**Scaling**

- Reads dominate (browsing beats buying). Add **read replicas** for product and catalog reads.
- Cache hot product pages in Redis; invalidate on price or stock change.
- Orders keep growing, so **partition** `orders` and `order_items` by `order_date` (range
  partitioning). Old months move to cheaper storage and queries touch fewer partitions.
- If you shard, shard by `customer_id` so one customer's orders sit together.

**SQL vs NoSQL choice**

Choose **SQL (PostgreSQL)**. Orders need multi-row transactions (stock + items + total in one
atomic step) and foreign keys keep the data consistent. This is the textbook case where ACID
guarantees are worth more than raw write scale. A product **catalog** with flexible attributes
could sit in a document store, but the ordering core stays relational.

## Case Study 2 — Social news feed

Users follow other users and post updates. Each user sees a feed of posts from people they
follow, newest first. The hard part is building the feed fast for millions of users.

**Requirements**

- Users follow other users (a directed relationship).
- Users create posts.
- Show a user's home feed: recent posts from everyone they follow.
- Handle "celebrities" — accounts with millions of followers.

Clarifying questions:

- How many users, and what is the follower distribution? (A few huge accounts change the design.)
- Is the feed strictly time-ordered, or ranked by an algorithm? (Ranking changes the read path.)
- How fresh must the feed be — instant, or is a few seconds of delay fine?
- Read/write ratio? Feeds are read far more than posts are written.

**Entities & Relationships**

```
user 1 ---- * post
user * ---- * user     (follows: follower_id -> followee_id)
```

`follows` is a many-to-many self-relationship on users, modeled as its own table.

**Schema**

```sql
CREATE TABLE users (
    user_id    BIGSERIAL PRIMARY KEY,
    handle     TEXT UNIQUE NOT NULL,
    name       TEXT
);

CREATE TABLE follows (
    follower_id BIGINT NOT NULL REFERENCES users(user_id),
    followee_id BIGINT NOT NULL REFERENCES users(user_id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (follower_id, followee_id)
);

CREATE TABLE posts (
    post_id    BIGSERIAL PRIMARY KEY,
    author_id  BIGINT NOT NULL REFERENCES users(user_id),
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Materialized feed, used only in the fan-out-on-write approach
CREATE TABLE feed_entries (
    user_id    BIGINT NOT NULL,   -- whose feed this row belongs to
    post_id    BIGINT NOT NULL,
    author_id  BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, created_at, post_id)
);
```

**Indexes**

```sql
CREATE INDEX idx_posts_author_time ON posts (author_id, created_at DESC);
CREATE INDEX idx_follows_follower  ON follows (follower_id);   -- who do I follow
CREATE INDEX idx_follows_followee  ON follows (followee_id);   -- who follows me
```

The `follows` table needs indexes in **both** directions. Fan-out-on-read needs "who do I
follow"; fan-out-on-write needs "who follows me" to know where to push a new post.

**Fan-out-on-read vs fan-out-on-write**

This is the core question. Two ways to build the feed:

*Fan-out-on-read* (build the feed when the user opens the app):

```sql
SELECT p.*
FROM posts p
JOIN follows f ON f.followee_id = p.author_id
WHERE f.follower_id = :me
ORDER BY p.created_at DESC
LIMIT 50;
```

Writing a post is cheap: one insert. But every feed read joins and sorts across everyone you
follow. Reads are expensive and get slower as people follow more accounts.

*Fan-out-on-write* (push the post into each follower's feed at post time): when a user posts,
insert one `feed_entries` row per follower. Reading the feed is then a simple, fast lookup:

```sql
SELECT * FROM feed_entries
WHERE user_id = :me
ORDER BY created_at DESC
LIMIT 50;
```

Reads are very fast. But a post by someone with 1M followers writes 1M rows. Writes are
expensive.

Rule of thumb: feeds are read far more than written, so **fan-out-on-write wins for most
users**. The read path is the one you must keep fast. A useful way to say it in the interview:
fan-out-on-write moves the cost from read time to write time, and since reads are far more
frequent, paying at write time is the cheaper trade overall.

**The celebrity problem**

Fan-out-on-write breaks for celebrities: one post = millions of inserts, a write storm. The
standard fix is a **hybrid**:

- Normal users: fan-out-on-write. Push their posts into follower feeds.
- Celebrities: do **not** fan out. Their posts stay only in `posts`.
- At read time, a user's feed = their materialized `feed_entries` **plus** a live query for
  posts from the few celebrities they follow, merged and sorted.

This keeps writes bounded and reads fast. Say this trade-off out loud — interviewers probe it.

**Normalization / Denormalization decisions**

- `follows` and `posts` are normalized.
- `feed_entries` is heavy denormalization: the same post is copied into many feeds. We accept
  the storage cost and duplication to make reads O(1). We also copy `author_id` and
  `created_at` into it so the read needs no join at all.

**Scaling**

- Store `feed_entries` in a fast key-value or wide-column store (Redis, Cassandra) keyed by
  `user_id`. It is append-and-read, not relational.
- Cap each feed (keep only the newest ~1000 entries) so it does not grow forever.
- Shard `posts` and `follows` by `user_id`.

**SQL vs NoSQL choice**

**Mixed.** Keep `users`, `follows`, and `posts` in SQL — they are relational and need
integrity. Keep the materialized `feed_entries` in NoSQL (a wide-column or key-value store)
because it is huge, denormalized, and only needs "get latest N by key". This split — SQL for
the source of truth, NoSQL for the read-optimized view — is a strong, defensible answer.

## Case Study 3 — Chat / messaging

Users exchange messages inside conversations. A conversation can be one-to-one or a group. The
tricky parts are **read state** and **unread counts** per user.

**Requirements**

- Users belong to conversations; a conversation has many members.
- Messages are ordered inside a conversation.
- Track, per user, which messages are read and how many are unread.
- Load recent messages in a conversation fast.

Clarifying questions:

- One-to-one only, or group chats too? (Group changes the membership model.)
- Do we need per-message read receipts ("seen by"), or only a per-user unread count?
- Message scale per conversation? (Millions of rows means partitioning.)
- Do we need edit and delete, and must we keep history?

**Entities & Relationships**

```
conversation 1 ---- * participant * ---- 1 user
conversation 1 ---- * message
```

A conversation has many participants (join table to users) and many messages.

**Schema**

```sql
CREATE TABLE conversations (
    conversation_id BIGSERIAL PRIMARY KEY,
    is_group        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE participants (
    conversation_id BIGINT NOT NULL REFERENCES conversations(conversation_id),
    user_id         BIGINT NOT NULL REFERENCES users(user_id),
    last_read_msg   BIGINT,      -- id of the last message this user has read
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE messages (
    message_id      BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(conversation_id),
    sender_id       BIGINT NOT NULL REFERENCES users(user_id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Indexes**

```sql
CREATE INDEX idx_messages_conv_id ON messages (conversation_id, message_id DESC);
CREATE INDEX idx_participants_user ON participants (user_id);
```

The first index serves the main read: "latest N messages in this conversation", newest first.
We sort by `message_id` (a monotonic id) instead of `created_at`, so ties from equal timestamps
still order correctly. The second index serves "list all my conversations".

**Read state and unread counts — the key decision**

The naive design stores one row per (user, message) marking read/unread. That is huge: in a
group of 100, every message writes 100 read-state rows. Avoid it.

Better: store one **`last_read_msg`** watermark per participant. A message is unread for a user
if its `message_id` is greater than that user's `last_read_msg`. Marking a conversation read is
a single update:

```sql
UPDATE participants SET last_read_msg = :latest_msg_id
 WHERE conversation_id = :c AND user_id = :me;
```

Unread count is then a range count:

```sql
SELECT count(*) FROM messages
 WHERE conversation_id = :c AND message_id > :last_read_msg;
```

Since ids are ordered, this uses the index and is fast. This turns per-message state into one
number per participant. It works because chat read state is almost always "read up to here",
not random messages read out of order. For a total unread badge across all chats, sum each
conversation's unread count, or keep a small cached counter per participant that you bump on new
messages and reset to zero on read.

If you truly need per-message "seen by" receipts, add a small `message_reads(message_id,
user_id, read_at)` table, but only for that feature, and expect it to be large.

**Normalization / Denormalization decisions**

- Normalized core: conversations, participants, messages.
- `last_read_msg` is a denormalized summary. It replaces a whole table of per-message state
  with one column. This is a deliberate simplification, not a mistake.
- Optionally cache `last_message_at` and an unread count per participant so the conversation
  list loads without counting. Update it when a message arrives.

**Scaling**

- Messages are the biggest table. **Partition** `messages` by `conversation_id` (hash) or by
  time, so each conversation's history stays local and old data can age out.
- Reads (open chat, scroll up) dominate; add read replicas.
- Shard by `conversation_id` so a whole conversation lives on one shard and reads hit one node.

**SQL vs NoSQL choice**

Either works, and you should say why. **SQL** is a fine default at moderate scale: strong
ordering, easy joins, transactions for "insert message + bump counters". At very high write
volume (think WhatsApp scale), messages are often moved to a **wide-column store like
Cassandra**, partitioned by conversation and clustered by message id, because that write rate
and time-series read pattern fit it well. State the scale that flips your choice.

## Case Study 4 — Booking / reservation system

Users book a resource (a hotel room, a meeting room, a seat) for a time range. The single
hardest requirement is **no double-booking**: the same room must never be held by two bookings
whose times overlap.

**Requirements**

- Resources (rooms) can be booked for a start–end time range.
- Two confirmed bookings for the same room must never overlap in time.
- Show a room's bookings and check availability for a range.

Clarifying questions:

- Are slots fixed (e.g. 30-minute blocks) or free-form time ranges? (This changes the constraint.)
- Can a booking be cancelled, and does a cancelled booking free the slot?
- Do we hold a slot temporarily during checkout (a soft hold with a timeout)?
- Time zones — do we store everything in UTC? (Yes, store `TIMESTAMPTZ`.)

**Entities & Relationships**

```
room 1 ---- * booking * ---- 1 user
```

One room has many bookings over time. Each booking belongs to one user and one room.

**Schema (free-form time ranges)**

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;   -- lets us mix = and range in one constraint

CREATE TABLE rooms (
    room_id   BIGSERIAL PRIMARY KEY,
    name      TEXT NOT NULL
);

CREATE TABLE bookings (
    booking_id BIGSERIAL PRIMARY KEY,
    room_id    BIGINT NOT NULL REFERENCES rooms(room_id),
    user_id    BIGINT NOT NULL REFERENCES users(user_id),
    during     TSTZRANGE NOT NULL,      -- the booked time range [start, end)
    status     TEXT NOT NULL DEFAULT 'CONFIRMED',
    -- Stop overlaps for the SAME room, at the database level:
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

**The double-booking fix — the key decision**

The interviewer wants to know how you *guarantee* no overlap, not just check for it. Three
options, weakest to strongest:

1. **Check-then-insert in app code.** Query for overlaps, if none, insert. This has a race:
   two requests both see "free" and both insert. Wrong on its own.
2. **Row lock (`SELECT ... FOR UPDATE`)** on the room row before checking, so requests for the
   same room serialize. This works but you must remember to lock every time.
3. **A database exclusion constraint** (shown above). `EXCLUDE USING gist (room_id WITH =,
   during WITH &&)` tells Postgres: reject any new row where `room_id` is equal **and** the
   `during` range overlaps (`&&`) an existing row. The database enforces it for every writer,
   with no app-code race. This is the strongest answer.

An overlap check reads: two ranges `[s1,e1)` and `[s2,e2)` overlap when `s1 < e2 AND s2 < e1`.
The `&&` operator does exactly this. Using half-open ranges `[start, end)` means a booking that
ends at 10:00 does not clash with one that starts at 10:00.

For **fixed slots**, the design is simpler — model each slot as a row and use a plain unique
constraint:

```sql
CREATE TABLE slot_bookings (
    room_id  BIGINT NOT NULL,
    slot_ts  TIMESTAMPTZ NOT NULL,   -- start of a fixed 30-min slot
    user_id  BIGINT NOT NULL,
    PRIMARY KEY (room_id, slot_ts)   -- one booking per room per slot
);
```

The primary key `(room_id, slot_ts)` makes double-booking impossible: the second insert fails.
Turning time into discrete slots turns a hard overlap problem into a simple uniqueness problem.
Say this trade-off: fixed slots are simpler but less flexible.

**Indexes**

The exclusion constraint creates a GiST index on `(room_id, during)`, which also serves
availability queries. For "show my bookings" add:

```sql
CREATE INDEX idx_bookings_user ON bookings (user_id);
```

**Normalization / Denormalization decisions**

Fully normalized; no duplication needed. Availability is derived from bookings, not stored, to
avoid a second source of truth that could disagree. If availability lookups get hot, cache them
and rebuild on write.

**Scaling**

- Usually read-heavy (many people check availability, fewer book). Add read replicas for checks.
- Shard by `room_id` (or by property/hotel) so one resource's bookings and its overlap check
  stay on one node. The exclusion constraint is local to a node, so keeping a room on one shard
  is what makes it work.
- Use short-lived **soft holds** (a row with a status and expiry) during checkout so a slot is
  not double-offered while a user pays.

**SQL vs NoSQL choice**

Choose **SQL (PostgreSQL)**. The core requirement — a correctness constraint that spans rows
(no overlap) — is exactly what relational constraints and transactions give you. Most NoSQL
stores cannot enforce "no overlapping range for this key" in the engine. This is a clear SQL win.

## Case Study 5 — URL shortener

Map a short code (like `aX9fQ2`) to a long URL. A visit to the short link redirects to the long
URL. This system is extremely **read-heavy**: many redirects, far fewer creates.

**Requirements**

- Create a short code for a long URL.
- Given a short code, return the long URL fast (the redirect).
- Codes must be short, unique, and hard to guess in bulk.
- Optionally track click counts.

Clarifying questions:

- Do we allow custom aliases (user picks the code)?
- Should codes expire? Do we need per-user ownership and analytics?
- Expected scale — how many creates per day, how many redirects per second?
- Is it fine if two long URLs get two different codes? (Usually yes; de-dup is optional.)

**Entities & Relationships**

```
url_mapping (short_code -> long_url)   [core]
click_event * ---- 1 url_mapping       [optional analytics]
```

**Schema**

```sql
CREATE TABLE url_mapping (
    id          BIGSERIAL PRIMARY KEY,     -- internal numeric id
    short_code  TEXT UNIQUE NOT NULL,      -- the public code, e.g. 'aX9fQ2'
    long_url    TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    click_count BIGINT NOT NULL DEFAULT 0
);
```

**How to generate the code — the key decision**

Two main strategies:

1. **Counter + Base62 encode.** Take the auto-increment `id` (a number) and encode it in
   Base62 (`0-9A-Za-z`). Id 125 becomes `cb`, and so on. This guarantees uniqueness with no
   collision check, because each id is unique. It is the cleanest answer. Downside: codes are
   sequential and guessable (`...b`, `...c`), which leaks how many links exist. Fix by hashing
   the id or seeding the counter, or by permuting the id space.
2. **Random code + uniqueness check.** Generate a random 6–7 char Base62 string, insert, and
   rely on the `UNIQUE` constraint to reject the rare collision (retry on conflict). Codes are
   unguessable. 62^7 is a huge space, so collisions are rare at first but rise as it fills.

Say the trade-off: counter is collision-free but guessable; random is unguessable but needs a
retry. Base62 with 7 chars gives 62^7 ≈ 3.5 trillion codes, plenty for most systems.

For custom aliases, insert the chosen code and let the `UNIQUE` constraint reject duplicates.

A common follow-up: "how do you avoid a central counter becoming a bottleneck?" Hand each
application server a **range** of ids (say 1000 at a time) from a central allocator. Each server
then encodes ids from its own range with no per-write coordination, and refills when its range
runs low. This keeps code generation fast even across many servers.

**Indexes**

```sql
-- short_code UNIQUE already creates the index we need for the redirect lookup.
```

The redirect only ever looks up by `short_code`, and the `UNIQUE` constraint already builds that
index. No other index is needed for the core path. That simplicity is the point.

**Normalization / Denormalization decisions**

One table is enough; there is nothing to normalize. `click_count` is a denormalized counter kept
on the row for a fast total. If you also keep detailed `click_event` rows for analytics, the
counter becomes a cache you can rebuild by counting events. At high click rates, do not
`UPDATE ... SET click_count = click_count + 1` on every hit — that row becomes a write hotspot.
Instead count clicks in a stream or cache and flush periodically.

**Scaling**

- This is the read-heavy poster child. Put a **cache (Redis) in front**: `short_code -> long_url`.
  Most redirects never touch the database. This is the single most important scaling move.
- Add read replicas for cache misses.
- The mapping table is simple and can be **sharded by `short_code`** (hash) easily, because
  every read is a single-key lookup with no joins or ranges.
- Use a CDN / edge layer for the redirect if global latency matters.

**SQL vs NoSQL choice**

This one leans **NoSQL / key-value**, and it is worth saying why. The access pattern is a pure
key lookup (`short_code -> long_url`) with no joins, no transactions, no ranges. A key-value
store (Redis, DynamoDB) fits perfectly and scales reads cheaply. SQL also works fine and gives
you the easy `UNIQUE` constraint and Base62-from-id trick, so it is a reasonable default at small
scale. The honest answer: start with SQL for simplicity, move the hot path to a key-value cache
and store as reads grow.

## Case Study 6 — Wallet / ledger

Track money in user accounts and every movement between them. Here **correctness beats
everything**: money must never be created, lost, or left in a half-state. This is where
interviewers push hardest on transactions and data modeling.

**Requirements**

- Each account has a balance.
- Record every money movement (deposit, transfer, withdrawal).
- Balances must always be correct and auditable — you can prove how each balance was reached.
- Never lose or double-count money, even with crashes or concurrent transfers.

Clarifying questions:

- Single currency or many? (Many means currency per account and no cross-currency mixing.)
- Do we need a full audit trail (regulators usually require it)?
- Can balances go negative (overdraft), or never?
- What consistency do we need — strong (a bank) almost always, not eventual.

**Entities & Relationships**

```
account 1 ---- * ledger_entry * ---- 1 transaction
```

One transaction (e.g. a transfer) has two or more ledger entries. Each entry touches one
account. This is **double-entry** bookkeeping: every movement is recorded twice — once as money
out of one account, once as money into another — and the two sides must sum to zero.

**Schema**

```sql
CREATE TABLE accounts (
    account_id  BIGSERIAL PRIMARY KEY,
    owner_id    BIGINT NOT NULL,
    currency    CHAR(3) NOT NULL,          -- 'USD', 'INR'
    balance     BIGINT NOT NULL DEFAULT 0  -- cached balance, in MINOR units (cents)
);

CREATE TABLE transactions (
    txn_id      BIGSERIAL PRIMARY KEY,
    kind        TEXT NOT NULL,             -- 'TRANSFER', 'DEPOSIT'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    idempotency_key TEXT UNIQUE            -- stops the same request applying twice
);

CREATE TABLE ledger_entries (
    entry_id    BIGSERIAL PRIMARY KEY,
    txn_id      BIGINT NOT NULL REFERENCES transactions(txn_id),
    account_id  BIGINT NOT NULL REFERENCES accounts(account_id),
    amount      BIGINT NOT NULL,           -- signed, in minor units: +credit, -debit
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**The key decisions**

*Integer minor units.* Store money as an integer count of the smallest unit — cents, paise — in
`BIGINT`, not `FLOAT` and not decimals with rounding surprises. $12.34 is stored as `1234`.
Integer math is exact, so no cent is ever lost to floating-point rounding. Format to dollars
only when showing it to a user.

*Append-only, double-entry ledger.* `ledger_entries` is **append-only**: you never update or
delete a row. A correction is a new, reversing entry, not an edit. Every transaction inserts
balanced entries that sum to zero. A $100 transfer from A to B is:

```sql
BEGIN;
INSERT INTO transactions (kind, idempotency_key) VALUES ('TRANSFER', :key)
    RETURNING txn_id;   -- say it returns 500
INSERT INTO ledger_entries (txn_id, account_id, amount) VALUES
    (500, :A, -10000),   -- debit A by $100.00
    (500, :B, +10000);   -- credit B by $100.00
UPDATE accounts SET balance = balance - 10000 WHERE account_id = :A AND balance >= 10000;
-- if 0 rows updated, A lacks funds -> ROLLBACK
UPDATE accounts SET balance = balance + 10000 WHERE account_id = :B;
COMMIT;
```

Both entries and both balance updates happen in **one transaction** (atomicity from Chapter 9).
If anything fails, the whole transfer rolls back — money is never half-moved. The two entries
sum to zero, so total money in the system is conserved.

*Balance: derived vs cached.* The true balance is `SUM(amount)` of an account's ledger entries.
That is the source of truth and is always auditable. But summing all history on every read is
slow. So we keep a **cached `balance` column** and update it in the same transaction as the
entries. The invariant to state: `accounts.balance` must always equal the sum of that account's
`ledger_entries.amount`. You can verify it any time:

```sql
SELECT a.account_id, a.balance, COALESCE(SUM(l.amount), 0) AS computed
FROM accounts a
LEFT JOIN ledger_entries l ON l.account_id = a.account_id
GROUP BY a.account_id, a.balance
HAVING a.balance <> COALESCE(SUM(l.amount), 0);   -- any row here = a bug
```

*Idempotency.* Payment requests get retried (network timeouts). The `UNIQUE idempotency_key`
means a retried request cannot apply the same transfer twice — the second insert fails and the
transfer is not repeated.

*Consistent lock ordering.* When two transfers touch the same two accounts at once, they can
deadlock if each locks the accounts in a different order. Always lock (and update) accounts in a
fixed order — for example, lowest `account_id` first. This simple rule prevents the deadlock. It
is a favorite follow-up, so state it before you are asked.

**Indexes**

```sql
CREATE INDEX idx_ledger_account_time ON ledger_entries (account_id, created_at DESC);
CREATE INDEX idx_ledger_txn          ON ledger_entries (txn_id);
```

The first serves an account statement ("my recent movements"). The second gathers both sides of
one transaction. `idempotency_key`'s unique constraint already indexes the retry check.

**Normalization / Denormalization decisions**

- Fully normalized ledger, and it must stay that way for audit.
- `accounts.balance` is the one deliberate denormalization: a cached aggregate of the ledger.
  It is safe only because it is updated inside the same transaction and can be re-derived from
  the append-only entries at any time.

**Scaling**

- Writes need strong consistency, so keep a **single primary** for writes; do not accept
  eventual consistency for money.
- Add read replicas for statements and reports (reads can tolerate small lag).
- Shard by `account_id` when needed, but a transfer that crosses shards then needs a
  distributed transaction or a two-step (saga) flow — call this out as the hard part.
- The ledger only grows, so **partition** it by time and archive old periods.

**SQL vs NoSQL choice**

Choose **SQL (PostgreSQL)**, firmly. Money needs ACID transactions, strong consistency, and
constraints — exactly relational strengths. Eventual consistency can create or lose money, which
is unacceptable here. This is the clearest SQL case in the chapter: never reach for an eventually
consistent store for a ledger.

## Common Traps & Mistakes

- **Joining to the live price for old orders.** Order history must store the price paid
  (`unit_price`), not read today's `products.price`. A price change must not rewrite past orders.
- **Read-then-write races on stock or balances.** Do not read a value, decide in app code, then
  write. Use a conditional `UPDATE ... WHERE stock >= 1` (or a row lock) so the check and write
  are one atomic step.
- **Using `FLOAT` for money.** Floating point loses cents to rounding. Use `NUMERIC`, or better,
  integer minor units (cents) for a ledger.
- **Checking for booking overlap in app code.** Two requests can both see "free". Enforce it in
  the database with an exclusion constraint or a proper lock, not with a check-then-insert.
- **A read/write row per user per message.** Read state does not need a row per message. Store
  one `last_read_msg` watermark per participant and count by id range.
- **Fan-out-on-write for celebrities.** Pushing one post to millions of feeds is a write storm.
  Use the hybrid: fan-out for normal users, live-merge for celebrity accounts.
- **A single hot counter.** Incrementing one `click_count` or global counter row on every event
  serializes all writers. Batch, shard the counter, or count in a stream.
- **Forgetting idempotency on money.** Retried payment requests double-charge without a unique
  idempotency key.
- **Denormalized caches that can drift.** A cached `total_amount` or `balance` is fine only if
  you update it in the same transaction as the source rows, and can re-derive it to verify.
- **Not stating assumptions or the "it depends".** The design round rewards reasoning. Ask the
  clarifying questions, say the trade-off, and name the scale at which your choice would flip.
```
