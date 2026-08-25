# Pub/Sub System (LLD)

## Problem

Design an in-process Publish-Subscribe (Pub/Sub) system. A **topic** is a named channel for
messages. A **publisher** sends a message to a topic. A **subscriber** registers interest in a
topic and receives every message sent to that topic after it registers. A subscriber can
**unsubscribe** at any time. Many publishers and many subscribers can act at the same time, on
many topics, so the system must be thread-safe.

This is the **class-design (LLD)** version of the problem. We build the core engine as a single
Java process — not a distributed system like Kafka. We do use ideas from Kafka to explain
follow-up questions at the end.

## Requirements & Clarifying Questions

**Functional requirements**
1. Create a topic by name. Publish messages to a topic.
2. A subscriber can subscribe to a topic and unsubscribe from a topic.
3. When a publisher publishes a message, every *current* subscriber of that topic receives it.
4. A subscriber that joins after a message was published does not get that old message.
5. One slow subscriber must not block a publisher or block other subscribers.
6. The system must work correctly with many threads calling publish/subscribe at once.

**Clarifying questions to ask the interviewer**
- Does a topic need to exist before anyone publishes to it? *(Assume no — publishing to an
  unknown topic auto-creates it, but say this out loud, since some interviewers want an
  explicit `createTopic` step instead.)*
- Is delivery synchronous (the publisher waits) or asynchronous (the publisher returns right
  away)? *(We build it asynchronous, using a worker pool, so a slow subscriber cannot block
  the publisher. This is a common follow-up, so we design for it from the start.)*
- Do we need message ordering? *(Yes, per topic — messages sent by the same publisher to the
  same topic should reach a subscriber in the order they were published. We do not promise
  order *between* different topics.)*
- Do we need at-least-once delivery, with retry, or is at-most-once (best effort) enough?
  *(We start with at-most-once, and show how to extend to at-least-once. This trade-off is
  explained in detail below.)*
- Do we need to persist messages, so a subscriber can replay old ones? *(Out of scope for this
  LLD. This is one of the biggest differences from Kafka — see the Kafka note at the end.)*

## Design / Approach

We use two patterns as the core of the design:

1. **Observer pattern.** A `Topic` is the *subject*. A `Subscriber` is the *observer*. The
   topic keeps a set of subscribers and notifies all of them when a new message arrives. The
   topic does not know or care what a subscriber does with a message.
2. **Thread pool (worker pool) for async delivery.** Instead of the publisher thread calling
   `subscriber.onMessage(...)` directly (which would block the publisher if the subscriber is
   slow), the topic hands each delivery off to a shared `ExecutorService`. The publisher's
   `publish()` call returns almost right away. Each subscriber gets its messages delivered in
   order, on a background thread, using a **per-subscriber queue** — this keeps ordering
   correct even though delivery is async.

Concurrency-safe building blocks:

- `ConcurrentHashMap<String, Topic>` maps topic name to the `Topic` object. Many threads can
  create or look up topics at the same time, without one big lock.
- Each `Topic` keeps its subscribers in a `CopyOnWriteArraySet<Subscriber>` (or a
  `ConcurrentHashMap`-backed set). Subscribe/unsubscribe are rare compared to publish, and
  publish only *reads* the set (iterates it), so a copy-on-write structure is a good fit: reads
  never block, and reads never see a `ConcurrentModificationException`.
- Each subscriber has its own single-threaded delivery queue (`BlockingQueue` drained by one
  worker at a time), so messages for that subscriber are always delivered in the order they
  were published, even though many topics share one thread pool.

```
                 +-------------------+
                 |   PubSubBroker     |
                 |-------------------|
                 | topics:            |
                 |  ConcurrentHashMap |
                 |  <String, Topic>   |
                 +---------+---------+
                           |
                           | getOrCreateTopic(name)
                           v
                 +-------------------+          <<interface>>
                 |       Topic        | ------> Subscriber (Observer)
                 |-------------------|                ^
                 | subscribers:       |                |
                 |  CopyOnWriteArraySet|      +--------+---------+
                 | publish(msg)        |      | ConcreteSubscriber|
                 +---------+----------+      |  (own queue +      |
                           |                  |   worker thread)   |
                           | for each sub:    +--------------------+
                           | submit delivery task
                           v
                 +-------------------+
                 |  ExecutorService   |
                 |  (worker pool)     |
                 +-------------------+
```

**Publisher** is not a separate class in code — any code holding a reference to a `Topic` (or
to the `PubSubBroker`) can call `publish`. This matches how most real Pub/Sub client libraries
work: "publisher" is a role, not a fixed type.

## Java Solution

```java
// ---- Core message type ----

public final class Message {
    private final String topic;
    private final Object payload;
    private final long publishedAtMillis;

    public Message(String topic, Object payload) {
        this.topic = topic;
        this.payload = payload;
        this.publishedAtMillis = System.currentTimeMillis();
    }

    public String getTopic() { return topic; }
    public Object getPayload() { return payload; }
    public long getPublishedAtMillis() { return publishedAtMillis; }

    @Override
    public String toString() {
        return "Message{topic=" + topic + ", payload=" + payload + "}";
    }
}

// Observer interface: every subscriber implements this the same way.
public interface Subscriber {
    String getId();
    void onMessage(Message message);
}
```

```java
// ---- Per-subscriber delivery queue: keeps order, isolates slow subscribers ----

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicBoolean;

public final class SubscriberMailbox {
    private final Subscriber subscriber;
    private final BlockingQueue<Message> queue = new LinkedBlockingQueue<>();
    private final AtomicBoolean running = new AtomicBoolean(false);
    private final java.util.concurrent.ExecutorService workerPool;

    public SubscriberMailbox(Subscriber subscriber, java.util.concurrent.ExecutorService workerPool) {
        this.subscriber = subscriber;
        this.workerPool = workerPool;
    }

    // Called by a Topic when a new message arrives for this subscriber.
    public void enqueue(Message message) {
        queue.offer(message);
        scheduleDrainIfNeeded();
    }

    // Only one worker thread drains this mailbox at a time (running flag).
    // This guarantees in-order delivery for this subscriber, even though the
    // shared thread pool has many worker threads overall.
    private void scheduleDrainIfNeeded() {
        if (running.compareAndSet(false, true)) {
            workerPool.submit(this::drain);
        }
    }

    private void drain() {
        try {
            Message message;
            while ((message = queue.poll()) != null) {
                try {
                    subscriber.onMessage(message);
                } catch (RuntimeException ex) {
                    // A subscriber's bug must not crash the delivery worker
                    // or block other subscribers.
                    System.out.println("Subscriber " + subscriber.getId()
                            + " threw an error: " + ex.getMessage());
                }
            }
        } finally {
            running.set(false);
            // Edge case: a message may have been enqueued right after the
            // while-loop's poll() returned null but before running was reset.
            // Re-check and restart draining if so.
            if (!queue.isEmpty()) {
                scheduleDrainIfNeeded();
            }
        }
    }
}
```

```java
// ---- Topic: the Observer "subject" ----

import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.Map;

public final class Topic {
    private final String name;
    private final ExecutorService workerPool;
    // CopyOnWriteArraySet: publish() only iterates (read); subscribe/unsubscribe
    // are writes. This trade-off fits Pub/Sub, where publish is far more common.
    private final Set<Subscriber> subscribers = new CopyOnWriteArraySet<>();
    private final Map<String, SubscriberMailbox> mailboxes = new ConcurrentHashMap<>();

    public Topic(String name, ExecutorService workerPool) {
        this.name = name;
        this.workerPool = workerPool;
    }

    public String getName() { return name; }

    public void subscribe(Subscriber subscriber) {
        subscribers.add(subscriber);
        mailboxes.putIfAbsent(subscriber.getId(), new SubscriberMailbox(subscriber, workerPool));
    }

    public void unsubscribe(Subscriber subscriber) {
        subscribers.remove(subscriber);
        mailboxes.remove(subscriber.getId());
        // Any messages already queued for this subscriber are simply dropped
        // once removed. This is a design choice — see "Complexity & Thread-Safety".
    }

    // Called by a publisher. Fans the message out to every current subscriber.
    // This method returns quickly: it only enqueues work, it never runs a
    // subscriber's onMessage() itself.
    public void publish(Message message) {
        for (Subscriber subscriber : subscribers) {
            SubscriberMailbox mailbox = mailboxes.get(subscriber.getId());
            if (mailbox != null) {
                mailbox.enqueue(message);
            }
        }
    }

    public int subscriberCount() {
        return subscribers.size();
    }
}
```

```java
// ---- Broker: entry point, owns the topic registry and the shared thread pool ----

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

public final class PubSubBroker {
    private final Map<String, Topic> topics = new ConcurrentHashMap<>();
    private final ExecutorService workerPool;

    public PubSubBroker(int workerThreadCount) {
        this.workerPool = Executors.newFixedThreadPool(workerThreadCount);
    }

    // computeIfAbsent on ConcurrentHashMap is atomic: two threads racing to
    // create the same topic name will both get the same Topic instance.
    public Topic getOrCreateTopic(String topicName) {
        return topics.computeIfAbsent(topicName, name -> new Topic(name, workerPool));
    }

    public void subscribe(String topicName, Subscriber subscriber) {
        getOrCreateTopic(topicName).subscribe(subscriber);
    }

    public void unsubscribe(String topicName, Subscriber subscriber) {
        Topic topic = topics.get(topicName);
        if (topic != null) {
            topic.unsubscribe(subscriber);
        }
    }

    // The publisher's method. Any code holding the broker (or a Topic
    // reference) plays the role of "publisher" — there is no separate class.
    public void publish(String topicName, Object payload) {
        Topic topic = getOrCreateTopic(topicName);
        topic.publish(new Message(topicName, payload));
    }

    public void shutdown() {
        workerPool.shutdown();
    }
}
```

```java
// ---- Demo wiring ----

public class Demo {
    public static void main(String[] args) throws InterruptedException {
        PubSubBroker broker = new PubSubBroker(4);

        Subscriber logger = new Subscriber() {
            @Override public String getId() { return "logger-1"; }
            @Override public void onMessage(Message message) {
                System.out.println(Thread.currentThread().getName()
                        + " [logger-1] got: " + message);
            }
        };

        Subscriber slowAnalytics = new Subscriber() {
            @Override public String getId() { return "analytics-1"; }
            @Override public void onMessage(Message message) {
                try { Thread.sleep(200); } catch (InterruptedException ignored) { }
                System.out.println(Thread.currentThread().getName()
                        + " [analytics-1] processed: " + message);
            }
        };

        broker.subscribe("orders", logger);
        broker.subscribe("orders", slowAnalytics);

        // Many publishers, from many threads, at the same time.
        Runnable publishTask = () -> {
            for (int i = 0; i < 3; i++) {
                broker.publish("orders", "order-" + Thread.currentThread().getId() + "-" + i);
            }
        };
        Thread p1 = new Thread(publishTask, "publisher-A");
        Thread p2 = new Thread(publishTask, "publisher-B");
        p1.start();
        p2.start();
        p1.join();
        p2.join();

        // logger keeps up quickly; slowAnalytics lags behind, but neither
        // blocks the publisher threads above, and neither blocks the other.
        Thread.sleep(2000);
        broker.shutdown();
    }
}
```

## How It Works

1. `PubSubBroker` holds one `ConcurrentHashMap<String, Topic>`. `getOrCreateTopic` uses
   `computeIfAbsent`, which is atomic — even if two publisher threads ask for a brand-new
   topic name at the exact same time, only one `Topic` object is created and both threads get
   the same one.
2. `subscribe` adds the `Subscriber` to the topic's `CopyOnWriteArraySet` and creates a
   `SubscriberMailbox` for it. This is the Observer pattern: the topic (subject) now knows
   about one more observer, through the `Subscriber` interface only — it does not know the
   concrete class.
3. `publish` builds a `Message` and calls `Topic.publish`. The topic loops over its current
   subscriber set and hands the message to each subscriber's mailbox. This loop is fast: it
   never calls `onMessage` directly, so a publisher thread never blocks on subscriber code.
4. Each `SubscriberMailbox` has its own `BlockingQueue`. `enqueue` adds the message and, if no
   worker is currently draining this mailbox, submits a `drain` task to the shared
   `ExecutorService`. The `running` flag (an `AtomicBoolean`) guarantees only one thread drains
   one mailbox at a time — this is what keeps per-subscriber ordering correct, even with a
   shared pool of many worker threads.
5. `drain` pulls messages off the queue one at a time and calls `subscriber.onMessage`. If the
   subscriber throws an exception, we catch it, log it, and keep draining — one bad subscriber
   must not stop delivery to that same subscriber's next message, or to any other subscriber.
6. `unsubscribe` removes the subscriber from the set and drops its mailbox. Any message
   already sitting in that mailbox's queue is discarded — the subscriber will not see it,
   which matches requirement 3 ("current subscribers only").

## How to Extend (Follow-ups)

- **Wildcard or hierarchical topics** (for example, `orders.*` matches `orders.created` and
  `orders.shipped`). Add a `TopicMatcher` that a subscriber registers instead of a single
  topic name. This is a common "level up" question in interviews.
- **At-least-once delivery.** Right now, a message lost mid-`onMessage` is gone for good
  (at-most-once). To get at-least-once, add an acknowledgment step: `onMessage` returns a
  boolean or throws, and the mailbox re-queues the message (with a retry limit) instead of
  just logging and moving on.
- **Message persistence / replay.** Store each `Message` in an append-only log per topic
  before fan-out, so a new subscriber can "replay from offset N" instead of only getting
  messages published after it joined. This is the biggest step toward Kafka-like behavior.
- **Backpressure.** If a subscriber's queue grows without limit, switch `LinkedBlockingQueue`
  to a bounded queue and pick a policy: drop the oldest message, drop the newest, or block
  the publisher (with a timeout) until space frees up.
- **Filtering per subscriber.** Let a subscriber pass a `Predicate<Message>` at subscribe
  time, checked before enqueueing, so it only receives messages it cares about.
- **Metrics.** Add counters for messages published per topic and mailbox queue depth per
  subscriber — useful for answering "how do you know a subscriber is falling behind?"

## Complexity & Thread-Safety Notes

- `publish` to a topic with `k` subscribers is `O(k)` to fan out (one `enqueue` call per
  subscriber). Each `enqueue` is `O(1)`.
- Delivery work is `O(1)` per message per subscriber, done on a background thread, so
  `publish` itself never waits on subscriber code — this is what "a slow subscriber does not
  block the publisher" means in this design.
- `ConcurrentHashMap` for topics allows many threads to create and read topics at once, with
  fine-grained internal locking (not one global lock).
- `CopyOnWriteArraySet` for subscribers means `publish`'s iteration never throws
  `ConcurrentModificationException`, even if another thread subscribes or unsubscribes at the
  same moment. The trade-off is that `subscribe`/`unsubscribe` copy the whole underlying array,
  which is `O(n)` — acceptable because subscribe/unsubscribe are rare compared to publish.
- The `AtomicBoolean running` flag in `SubscriberMailbox` is the key thread-safety idea: it
  makes sure exactly one worker thread drains a given subscriber's queue at a time, so
  messages for that subscriber are delivered strictly in the order they were enqueued — even
  though the thread pool has multiple worker threads shared across many subscribers.
- **Ordering guarantee**: messages to the *same* subscriber, from the *same* topic, arrive in
  publish order. There is **no** guarantee of order *across* different topics, and no
  guarantee of order between two different subscribers (they run independently, possibly on
  different worker threads, at different speeds).
- **At-most-once vs at-least-once**: the base design here is at-most-once — if delivery fails
  (subscriber throws, or the process dies mid-delivery), the message is not retried, so a
  subscriber might silently miss it. At-least-once needs acknowledgment plus retry (or a
  persisted log to replay from), which costs more complexity and possible duplicate delivery
  (a subscriber might then see the same message twice, so it must be idempotent). Always state
  in an interview which one you are building, and why — do not assume "exactly once," which is
  hard to achieve in a real distributed system, and is more of a marketing term than a strict
  guarantee even in Kafka.

## Interview Tips & Common Mistakes

- Do not deliver messages by calling `subscriber.onMessage(...)` directly from inside
  `publish`. That makes the publisher's thread block until every subscriber finishes, which
  fails the "slow subscriber must not block anyone" requirement. Say this is why delivery
  routes through a worker pool instead.
- Do not use a plain `HashSet` or `ArrayList` for the subscriber list. Under concurrent
  subscribe/unsubscribe and publish, plain collections throw
  `ConcurrentModificationException` or lose updates. Use `ConcurrentHashMap`,
  `CopyOnWriteArraySet`, or explicit locking, and explain the trade-off you picked.
- Do not share one queue across all subscribers of a topic. If you do, one slow subscriber
  blocks delivery to every other subscriber, because they all wait behind the same queue. Give
  each subscriber its own mailbox/queue, as shown above.
- Watch out for the "lost wakeup" bug in the mailbox: if you check `queue.isEmpty()` and then
  set `running = false` in the wrong order, a message enqueued in between might never get
  drained. The pattern shown here (drain in a loop, then re-check after `running.set(false)`
  in the `finally` block) avoids this. Mention this edge case — it shows depth.
- Clearly separate "topic" (the Observer subject, holding a subscriber set) from "message
  routing" (the mailbox and worker pool). Interviewers sometimes ask for a different delivery
  strategy (for example, synchronous for unit tests) — a clean seam here makes that easy.
- If asked "how is this different from Kafka," answer briefly, do not over-explain:
    - **Topics** here match Kafka topics in name and idea (a named stream of messages).
    - **Partitions**: Kafka splits one topic into many partitions, each an ordered log, spread
      across machines, so many consumers can read the same topic in parallel while keeping
      order *within* a partition. Our design has no partitions — one topic is one unordered set
      of subscribers, each with its own ordered stream.
    - **Consumer groups**: in Kafka, subscribers in the same consumer group *share* the
      messages of a topic (each message goes to only one consumer in the group, for horizontal
      scaling). In our design, every subscriber gets every message (fan-out, not load-sharing).
      Adding consumer-group behavior would need a new concept: a "group," where the broker picks
      one member per message instead of notifying all members.
    - **Persistence**: Kafka stores messages on disk and lets consumers replay from an offset.
      Our design is in-memory and delivers only to currently subscribed listeners — closer to a
      live event bus than a durable log.
    - Say: "this in-process design solves the same core idea — decoupled publishers and
      subscribers — but Kafka adds durability, partitioned parallelism, and consumer groups for
      scale across many machines." That one sentence usually satisfies the follow-up.
