# Chapter 12 — Notifications System

Design a notifications system: "X liked your post," "Your order shipped," "Y commented on your photo." Almost every app has this. It tests fan-out, read state, and how to keep old data from breaking new reads.

## Requirements & Clarifying Questions

Key features to support:
- Send a notification to one user, or to many users at once (a "fan-out" event, like a post getting 500 likes).
- Show a user their notification list, newest first, with an unread count badge.
- Mark a notification as read (one, or all at once).
- Let a user turn notifications on or off, per notification type, per channel (in-app, email, push).
- Delete old notifications after some time.

Questions to ask the interviewer:
- Which channels do we support — in-app only, or also email and push? (Assume all three; the design must pick which channels to send on, per user preference.)
- Do read notifications ever need to be "unread" again? (Assume no, keep it simple: a flag, not a history.)
- How long do we keep notifications? (Assume 90 days in the main table, older ones archived or deleted.)
- Rough scale: assume tens of millions of users. A single popular event (a celebrity's post) can fan out to millions of notification rows in seconds. Writes will far outnumber reads.

## The Schema

```sql
CREATE TABLE notifications (
  id            BIGSERIAL PRIMARY KEY,
  user_id       BIGINT NOT NULL REFERENCES users(id),  -- the recipient
  type          TEXT NOT NULL,        -- e.g. 'like', 'comment', 'order_shipped'
  payload       JSONB NOT NULL,       -- self-contained data to render this notification
  is_read       BOOLEAN NOT NULL DEFAULT FALSE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON notifications (user_id, is_read, created_at DESC);

CREATE TABLE notification_preferences (
  user_id       BIGINT NOT NULL REFERENCES users(id),
  type          TEXT NOT NULL,        -- e.g. 'like', 'comment', 'order_shipped'
  channel       TEXT NOT NULL,        -- 'email', 'push', 'in_app'
  enabled       BOOLEAN NOT NULL DEFAULT TRUE,
  PRIMARY KEY (user_id, type, channel)
);
```

- `notifications` — one row per notification, per recipient. This is the main table, and it is write-heavy.
- `notification_preferences` — one row per `(user, type, channel)` combination. It answers: "should we send this user a push notification when someone comments?"

## Key Design Decisions — the Reasoning

**1. Fan-out on write: one row per recipient, not one shared row.**
A tempting shortcut: store one event row ("post 42 got a like from user 7") and let every viewer figure out, at read time, whether it applies to them and whether they have seen it. This is called **fan-out on read**. It sounds cheap because you write once. But it breaks the moment you need per-user state: has *this* user read it? Did *this* user turn off likes notifications? You would need a join and a lookup on every page load, for every user, forever.

The standard design is **fan-out on write**: when the event happens, write one `notifications` row *per recipient*. Each row has its own `is_read` flag and its own `user_id`. Now "get my unread notifications" is a single indexed query on one table, with no fan-out logic at read time. The trade-off: writes cost more, and a viral event (500 likes on one post) can create a burst of rows. This is the right trade, because notifications are read far more often (every app open) than they are triggered (once per event). You pay the fan-out cost once, at write time, so every later read stays cheap. Use a background job or message queue to do the fan-out asynchronously, so a viral post does not block the request that triggered it.

**2. Read state: an `is_read` flag, backed by an index on `(user_id, is_read, created_at)`.**
The two most common queries are: "show my notification list, newest first" and "how many are unread?" (the badge count). Both filter by `user_id`, and the second also filters by `is_read = false`. The composite index `(user_id, is_read, created_at DESC)` covers both:

```sql
-- unread count (the badge)
SELECT count(*) FROM notifications
WHERE user_id = :uid AND is_read = FALSE;

-- notification list, newest first
SELECT * FROM notifications
WHERE user_id = :uid
ORDER BY created_at DESC
LIMIT 30;
```

A single `BOOLEAN` flag is enough. We do not need a history of *when* it became read, or whether it went read-then-unread — the product does not need that. Keep the flag simple; add a `read_at` timestamp column only if a real requirement asks for it. "Mark all as read" becomes one cheap `UPDATE ... WHERE user_id = :uid AND is_read = FALSE`, covered by the same index.

**3. Store a self-contained `payload` (denormalized JSON), instead of joining to the source data.**
Say user A likes user B's photo. A normalized design would store `notifications(user_id, actor_id, photo_id)` and, at render time, join to the `users` table for the actor's name and avatar, and to the `photos` table for a thumbnail. This looks clean. But it breaks in an important way: if the actor later deletes their account, or the photo gets deleted, the join returns nothing — and the notification either disappears or renders broken, even though the event genuinely happened in the past.

The fix: copy the data the notification needs to render — the text, a link, an icon URL — straight into the `payload` JSONB column, at the moment the notification is created. For example:

```json
{
  "text": "Jane Doe liked your photo",
  "actor_name": "Jane Doe",
  "actor_avatar_url": "https://.../jane.jpg",
  "link": "/photos/9182",
  "icon": "heart"
}
```

Now the notification renders correctly forever, even if Jane deletes her account tomorrow or the photo is removed. This is the same idea as "price at time of purchase" from Chapter 2 (an order stores the price it charged, not a link to today's price) — a notification stores the facts as they were *at the time of the event*, not a live pointer to data that can change or vanish.

The trade-off: the data can go stale. If Jane changes her display name next week, this old notification still shows "Jane Doe," her old name. For a notification, that is fine — the reader cares what happened, not what is true right now. It also uses more storage than a foreign key, and `JSONB` fields are not as easy to query or index deeply as normal columns. Both trade-offs are worth it, because notifications are read-only history, not live state. If you ever need to query inside the JSON (say, "find all notifications about photo 9182"), add a `GIN` index on the column, or pull that one field into its own indexed column instead.

**4. Preferences per type and per channel — a small table, checked before sending.**
Users want control: "email me for orders, but do not push-notify me for likes." A single `notifications_enabled BOOLEAN` on the user row cannot express this. `notification_preferences` uses a composite key of `(user_id, type, channel)`, so each user can have a different setting per `(type, channel)` pair. Before sending on a channel, the fan-out job checks this table; if there is no row, default to `enabled = TRUE` (most products want notifications on by default). This table stays small — it grows with the number of `(type, channel)` pairs the product supports, not with the number of events — so it never needs the scaling care `notifications` needs.

**5. Retention: old notifications get archived or deleted.**
Nobody scrolls back 2 years into their notification history. Keeping every row forever wastes space and slows down the table for no benefit. The common pattern: run a scheduled job that deletes (or moves to a cheaper archive table) notifications older than a fixed window, for example 90 days. Deleting also naturally caps how large the "hot" table can grow, which matters a lot for the next section.

## Scaling It

`notifications` is the hardest table here: it is write-heavy (fan-out multiplies writes), and it grows fast (every like, comment, and system event adds rows). The path, in order:

1. **Index first** (done above) — `(user_id, is_read, created_at DESC)` covers both main queries.
2. **Queue the fan-out.** Never write hundreds of thousands of rows inside the request that triggered the event (a viral post). Push the event to a queue (Kafka, SQS, or similar) and let a worker do the fan-out writes in the background, in batches.
3. **Partition the table.** Once it holds billions of rows, split it. Partition by time (e.g., one partition per month) if most queries filter by recent time and old partitions can be dropped wholesale when retention expires. Partition by `user_id` (hash partitioning) if the goal is to spread load evenly and keep one user's notifications together. Time-based partitioning pairs naturally with the retention job in decision 5: dropping a whole old partition is far cheaper than deleting rows one by one.
4. **Archive aggressively.** Move anything past the retention window out of the hot table (to cold storage or a separate archive table) so the live table, and its indexes, stay small and fast.
5. **Cache the unread count.** Recomputing `COUNT(*) WHERE is_read = FALSE` on every page load is wasteful once a user has a lot of history. Many products keep a running unread counter (in Redis, or a counter column updated on write/read) instead of counting rows each time.

## Interview Tips & Common Mistakes

- Say "fan-out on write" out loud and explain why: it moves cost to write time so every read stays a single indexed query. This is the decision most interviewers probe first.
- Do not propose one shared row for a fan-out event. It cannot hold a per-user `is_read` flag or per-user preferences — say this explicitly.
- Justify the JSONB `payload` as a deliberate denormalization: the notification must still make sense after the source data is gone. Name the trade-off (staleness) instead of ignoring it.
- Mention the `(user_id, is_read, created_at)` index without being asked — it shows you know the two real queries (list, unread count) before being told them.
- If asked about scale, do not jump to sharding first. Walk the ladder: index, queue the fan-out, partition, archive, cache the count.
- Bring up retention on your own. A notifications table with no delete or archive policy is a common gap interviewers look for.
