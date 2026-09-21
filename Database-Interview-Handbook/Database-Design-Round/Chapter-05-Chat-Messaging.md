# Chapter 5 — Chat / Messaging

Design a chat system like WhatsApp or Slack: users send messages in 1:1 chats and group chats. This question tests how well you can model relationships and handle a table that grows very fast.

## Requirements & Clarifying Questions

Key features to support:
- 1:1 chats (two people) and group chats (many people).
- Send a text message. Load a conversation's message history, newest first.
- Show unread message counts per user, per conversation.
- Users can join or leave a group.

Questions to ask the interviewer:
- Do we need message editing or deletion? (Assume soft delete only, to keep it simple.)
- Do we need media (images, files) or just text? (Assume text now; media is a separate table with a URL, same pattern.)
- Do we need delivery receipts ("seen by") for group chats, or just unread counts? (This chapter covers unread counts. Per-user "seen" state for groups is a harder version of the same idea — see the scaling note.)
- Rough scale: assume millions of users, and a small fraction of very active group chats (hundreds of members, thousands of messages a day).

## The Schema

```sql
CREATE TABLE users (
  id            BIGSERIAL PRIMARY KEY,
  username      TEXT NOT NULL UNIQUE,
  ...
);

CREATE TABLE conversations (
  id            BIGSERIAL PRIMARY KEY,
  is_group      BOOLEAN NOT NULL DEFAULT FALSE,
  title         TEXT,               -- NULL for 1:1 chats
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversation_members (
  conversation_id     BIGINT NOT NULL REFERENCES conversations(id),
  user_id             BIGINT NOT NULL REFERENCES users(id),
  joined_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_read_message_id BIGINT,      -- points to a row in messages
  last_read_at        TIMESTAMPTZ,
  PRIMARY KEY (conversation_id, user_id)
);
CREATE INDEX ON conversation_members (user_id);

CREATE TABLE messages (
  id              BIGSERIAL PRIMARY KEY,
  conversation_id BIGINT NOT NULL REFERENCES conversations(id),
  sender_id       BIGINT NOT NULL REFERENCES users(id),
  body            TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ         -- soft delete
);
CREATE INDEX ON messages (conversation_id, created_at DESC, id DESC);
```

- `users` — one row per person. Kept thin here; auth and profile fields live elsewhere.
- `conversations` — one row per chat, group or 1:1. `is_group` tells you which kind it is.
- `conversation_members` — a junction table. It links users to conversations, and also stores per-user state for that conversation (when they joined, what they have read).
- `messages` — one row per message. The biggest table by far, and the one that needs the most care.

## Key Design Decisions — the Reasoning

**1. One model for both 1:1 and group chat.**
A junior design often makes two things: a `direct_messages` table for 1:1 chats, and a separate `group_messages` table for groups. This looks simple at first, but it doubles your work. You need two sets of queries, two pagination paths, and two places to add features like "mark as read." It also breaks the moment a product wants to turn a 1:1 chat into a group (add a third person) — a common real feature.

The better model: a **conversation** is just a set of members, whether it has 2 people or 200. `conversation_members` is a classic many-to-many junction table (see Chapter 2) between `users` and `conversations`. A 1:1 chat is a conversation with exactly two members. A group chat is a conversation with more. `messages` only ever needs to know the `conversation_id` — it does not care how many members that conversation has. One schema, one set of queries, for both cases.

**2. Index `messages` by `(conversation_id, created_at DESC, id DESC)`.**
The single most common query in a chat app is: "give me the last 30 messages in this conversation." Without the right index, Postgres would have to scan every message ever sent, across all conversations, and filter down to this one. That is a full table scan on the biggest table in the system — too slow.

The composite index `(conversation_id, created_at DESC, id DESC)` matches this read pattern exactly. Postgres can jump straight to the rows for one `conversation_id`, already sorted newest-first, and return the top 30 with almost no extra work. This is the general rule from Chapter 2: **index for the main read pattern.** We add `id` as a tie-breaker because two messages can have the same `created_at` timestamp (down to the millisecond, under load); `id` is always unique, so the sort order stays stable.

**3. Read state: store `last_read_message_id` per member, not a read flag per message.**
A naive design adds a `message_reads` table with one row per `(message_id, user_id)` pair, marked when a user reads a message. This blows up fast: a group with 200 members and 10,000 messages a day means 2,000,000 read-rows a day, just for bookkeeping. Most of that data is never queried directly — the app only ever asks "how many unread messages does this user have?"

The cheap answer: store one value per member — `last_read_message_id` (or `last_read_at`) — on the `conversation_members` row. When a user opens a conversation, update that one field to the newest message's ID. To compute unread count, run:

```sql
SELECT count(*) FROM messages
WHERE conversation_id = :cid AND id > :last_read_message_id;
```

This is one row updated per "read" action, instead of one row per message per member. It trades a small COUNT query (cheap, backed by the index above) for a huge write-amplification problem. The trade-off: you lose fine detail, like "exactly which messages did this user read." Most chat apps do not need that. If a product truly needs per-message "seen by" receipts (common in small group chats, like iMessage's "Read" tag), that is a deliberate, separate feature — not the default.

**4. Message ordering and pagination: keyset pagination, not `OFFSET`.**
When a user scrolls up to load older messages, do not use `LIMIT 30 OFFSET 3000`. `OFFSET` forces Postgres to walk through and discard the first 3,000 rows every time, and it gets slower as the user scrolls further back. It is also wrong if new messages arrive while paging, shifting every offset.

Use **keyset pagination** instead: remember the `created_at` and `id` of the last message you saw, and ask for the next page with:

```sql
SELECT * FROM messages
WHERE conversation_id = :cid
  AND (created_at, id) < (:last_seen_created_at, :last_seen_id)
ORDER BY created_at DESC, id DESC
LIMIT 30;
```

This uses the same `(conversation_id, created_at, id)` index directly, so every page costs the same, no matter how deep you scroll. This is a reusable pattern (see Chapter 2) for any "load more, newest first" feed, not just chat.

## Scaling It

The `messages` table is the bottleneck. A busy chat product can generate billions of rows a year. The scaling path, roughly in order:

1. **Index first** (done above) — this alone carries you a long way, because almost every query filters by `conversation_id`.
2. **Cache the last page.** The most recent 20–30 messages of an active conversation are read far more often than they are written. Cache them (Redis is a common choice) so most "open chat" requests never touch Postgres.
3. **Read replicas** for the heavy read load (loading history) once one primary database cannot keep up.
4. **Shard `messages` by `conversation_id`.** Once the table is too big for one machine, split it across multiple database servers, using a hash of `conversation_id` to decide which server holds which conversation's messages. This keeps all of one conversation's messages together, so the common query (load one conversation's history) still hits only one shard. Sharding by `id` instead would scatter one conversation's messages across every shard — the worst possible layout for this workload.
5. **Archive old messages.** Most users only look at recent history. Move messages older than, say, 12 months to cheaper, slower storage (a separate "cold" table or object storage), and only load them on request. This keeps the "hot" table small and fast.

## Interview Tips & Common Mistakes

- Do not design two separate tables for 1:1 and group chat. Say out loud: "I will model both as a conversation with members, so one schema handles both." This is the single most-tested decision in this chapter.
- Do not propose a per-message read receipt for every member as the default. It is correct to mention it as an option, but explain the write-amplification cost and prefer `last_read_message_id`.
- Always mention the `(conversation_id, created_at, id)` index without being asked. It shows you know the main read pattern before you know every requirement.
- If asked about pagination, say "keyset, not offset" and explain why offset gets slow and unstable as new rows come in.
- If the interviewer pushes on scale, walk the ladder: index, cache, replica, shard — do not jump straight to "shard everything," it sounds like you skipped the cheaper steps.
