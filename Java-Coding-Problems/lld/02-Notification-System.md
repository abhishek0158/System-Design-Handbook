# Notification System (LLD)

## Problem

Design a notification system for a backend application. The system must send a notification
to a user through one or more channels: Email, SMS, and Push (mobile push notification).

Each user has preferences. A preference says which channels the user allows. The message
content comes from a template, not from hard-coded text. When something happens in the
system (for example, "order shipped"), the system should trigger the right notification
automatically. A send can fail (network error, provider down). The system should retry a
failed send. New channels (for example, WhatsApp) should be easy to add later, without
changing existing code.

This is the **class-design (LLD)** version of the problem. We do not design servers, queues,
or databases here. We design clean Java classes and the patterns that connect them.

## Requirements & Clarifying Questions

**Functional requirements**
1. Send a notification to a user through Email, SMS, or Push.
2. Pick the channel(s) based on the user's saved preferences.
3. Build the message from a template, filled in with real data (name, order id, and so on).
4. Support system events (e.g., `ORDER_SHIPPED`) that trigger a notification automatically.
5. Retry a failed send a fixed number of times before giving up.
6. Let a developer add a new channel type without editing old, working code.

**Clarifying questions to ask the interviewer**
- Can one event trigger notifications on more than one channel at the same time? *(Assume yes.)*
- Is delivery guaranteed, or is "best effort with retry" enough? *(Assume best effort with retry.)*
- Do we need to persist notification history? *(Out of scope for this LLD; mention it as a
  follow-up.)*
- Should sending block the caller, or run in the background? *(We design it so it can run
  sync now and move to async later, with almost no code change. See the async note below.)*
- Can a user have no allowed channels at all? *(Yes — then we simply send nothing and log it.)*

## Design / Approach

We use four patterns, each solving one clear problem:

1. **Strategy** — `NotificationChannel` is an interface. `EmailChannel`, `SmsChannel`, and
   `PushChannel` are interchangeable strategies. `NotificationService` does not care which
   one it uses.
2. **Factory** — `NotificationChannelFactory` creates the right `NotificationChannel` object
   for a given `ChannelType`. It uses a registry (a map), so adding a new channel means
   *registering* it, not editing the factory's code. This gives us the **Open/Closed
   Principle**: open for extension, closed for modification.
3. **Decorator** — `RetryingChannel` wraps any `NotificationChannel` and adds retry behavior
   around it. The retry logic is written once and works for every channel.
4. **Observer** — `EventPublisher` is the subject. `NotificationService` and
   `AuditLogObserver` are observers. When a domain event happens (like "order shipped"), the
   publisher notifies every observer. `NotificationService` reacts by sending a notification.
   `AuditLogObserver` reacts by logging the event. Neither observer knows about the other.

```
                        +----------------------+
                        |    EventPublisher     |
                        |  (Observer: subject)  |
                        +----------+-----------+
                                   | notifies
                 -----------------+-----------------
                 |                                  |
                 v                                  v
      +----------------------+          +--------------------------+
      | NotificationService   |         |   AuditLogObserver        |
      | (implements Observer) |         |   (implements Observer)   |
      +----------+------------+          +--------------------------+
                 |
                 | 1. read allowed channels
                 v
      +--------------------------+
      |  UserPreferenceService    |
      +--------------------------+
                 |
                 | 2. render message
                 v
      +--------------------------+
      |     TemplateService       |
      +--------------------------+
                 |
                 | 3. get channel (Factory)
                 v
      +--------------------------+          <<interface>>
      | NotificationChannelFactory |  --->  NotificationChannel  (Strategy)
      +--------------------------+                 ^        ^        ^
                 |                                  |        |        |
                 | 4. wrap with retry (Decorator)   |        |        |
                 v                          EmailChannel SmsChannel PushChannel
      +--------------------------+
      |      RetryingChannel      |
      +--------------------------+
```

`NotificationService` is the one class that ties everything together: user preferences,
templates, the channel factory, and retry. Every other class has one small job. This keeps
each class easy to test on its own — a core LLD interview expectation at the 3–4 year level.

## Java Solution

```java
// ---- Core message and channel abstraction ----

public enum ChannelType {
    EMAIL, SMS, PUSH
}

public final class Message {
    private final String subject; // empty for SMS/Push, used for Email
    private final String body;

    public Message(String subject, String body) {
        this.subject = subject;
        this.body = body;
    }

    public String getSubject() { return subject; }
    public String getBody() { return body; }
}

public class NotificationException extends RuntimeException {
    public NotificationException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Strategy interface: every channel implements this the same way.
public interface NotificationChannel {
    ChannelType getType();
    // "address" is an email address, a phone number, or a device token,
    // depending on the channel.
    void send(String address, Message message);
}

public class EmailChannel implements NotificationChannel {
    @Override
    public ChannelType getType() { return ChannelType.EMAIL; }

    @Override
    public void send(String address, Message message) {
        // In real code: call an email provider SDK (e.g., SES, SendGrid).
        System.out.println("[EMAIL to " + address + "] subject=" + message.getSubject()
                + " body=" + message.getBody());
    }
}

public class SmsChannel implements NotificationChannel {
    @Override
    public ChannelType getType() { return ChannelType.SMS; }

    @Override
    public void send(String address, Message message) {
        // In real code: call an SMS gateway (e.g., Twilio).
        System.out.println("[SMS to " + address + "] " + message.getBody());
    }
}

public class PushChannel implements NotificationChannel {
    @Override
    public ChannelType getType() { return ChannelType.PUSH; }

    @Override
    public void send(String address, Message message) {
        // In real code: call a push provider (e.g., FCM, APNs).
        System.out.println("[PUSH to device " + address + "] " + message.getBody());
    }
}
```

```java
// ---- Decorator: adds retry to any channel, without changing the channel ----

public class RetryingChannel implements NotificationChannel {
    private final NotificationChannel delegate;
    private final int maxAttempts;

    public RetryingChannel(NotificationChannel delegate, int maxAttempts) {
        this.delegate = delegate;
        this.maxAttempts = maxAttempts;
    }

    @Override
    public ChannelType getType() { return delegate.getType(); }

    @Override
    public void send(String address, Message message) {
        RuntimeException lastError = null;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                delegate.send(address, message);
                return; // success, stop here
            } catch (RuntimeException ex) {
                lastError = ex;
                System.out.println("Attempt " + attempt + " failed for "
                        + delegate.getType() + ": " + ex.getMessage());
            }
        }
        throw new NotificationException(
                "All " + maxAttempts + " attempts failed for " + delegate.getType(), lastError);
    }
}

// ---- Factory: registry-based, so new channels do not need code changes here ----

public class NotificationChannelFactory {
    private final Map<ChannelType, Supplier<NotificationChannel>> registry = new HashMap<>();

    public void register(ChannelType type, Supplier<NotificationChannel> supplier) {
        registry.put(type, supplier);
    }

    public NotificationChannel create(ChannelType type) {
        Supplier<NotificationChannel> supplier = registry.get(type);
        if (supplier == null) {
            throw new IllegalArgumentException("No channel registered for " + type);
        }
        return supplier.get();
    }
}
```

```java
// ---- Template engine: turns a template key + data into a real message ----

public final class MessageTemplate {
    private final String key;
    private final String subjectTemplate; // e.g. "Order {{orderId}} shipped"
    private final String bodyTemplate;    // e.g. "Hi {{name}}, your order shipped."

    public MessageTemplate(String key, String subjectTemplate, String bodyTemplate) {
        this.key = key;
        this.subjectTemplate = subjectTemplate;
        this.bodyTemplate = bodyTemplate;
    }

    public String getKey() { return key; }
    public String getSubjectTemplate() { return subjectTemplate; }
    public String getBodyTemplate() { return bodyTemplate; }
}

public class TemplateService {
    private final Map<String, MessageTemplate> templates = new HashMap<>();

    public void addTemplate(MessageTemplate template) {
        templates.put(template.getKey(), template);
    }

    public Message render(String templateKey, Map<String, String> params) {
        MessageTemplate template = templates.get(templateKey);
        if (template == null) {
            throw new IllegalArgumentException("Unknown template: " + templateKey);
        }
        String subject = fill(template.getSubjectTemplate(), params);
        String body = fill(template.getBodyTemplate(), params);
        return new Message(subject, body);
    }

    private String fill(String text, Map<String, String> params) {
        String result = text;
        for (Map.Entry<String, String> entry : params.entrySet()) {
            result = result.replace("{{" + entry.getKey() + "}}", entry.getValue());
        }
        return result;
    }
}
```

```java
// ---- User and preferences ----

public final class User {
    private final String id;
    private final String email;
    private final String phone;
    private final String deviceToken;

    public User(String id, String email, String phone, String deviceToken) {
        this.id = id;
        this.email = email;
        this.phone = phone;
        this.deviceToken = deviceToken;
    }

    public String getId() { return id; }

    // Picks the right "address" for a given channel type.
    public String addressFor(ChannelType type) {
        switch (type) {
            case EMAIL: return email;
            case SMS: return phone;
            case PUSH: return deviceToken;
            default: throw new IllegalArgumentException("Unsupported channel: " + type);
        }
    }
}

public final class UserPreference {
    private final Set<ChannelType> allowedChannels;

    public UserPreference(Set<ChannelType> allowedChannels) {
        this.allowedChannels = new HashSet<>(allowedChannels);
    }

    public Set<ChannelType> getAllowedChannels() {
        return Collections.unmodifiableSet(allowedChannels);
    }
}

public class UserPreferenceService {
    // ConcurrentHashMap: safe if preferences are read/written from several threads.
    private final Map<String, UserPreference> preferencesByUserId = new ConcurrentHashMap<>();

    public void setPreference(String userId, UserPreference preference) {
        preferencesByUserId.put(userId, preference);
    }

    public Set<ChannelType> getAllowedChannels(String userId) {
        UserPreference pref = preferencesByUserId.get(userId);
        return pref == null ? Collections.emptySet() : pref.getAllowedChannels();
    }
}
```

```java
// ---- Observer pattern: domain events drive notifications ----

public final class NotificationEvent {
    private final String userId;
    private final String templateKey;
    private final Map<String, String> params;

    public NotificationEvent(String userId, String templateKey, Map<String, String> params) {
        this.userId = userId;
        this.templateKey = templateKey;
        this.params = params;
    }

    public String getUserId() { return userId; }
    public String getTemplateKey() { return templateKey; }
    public Map<String, String> getParams() { return params; }
}

public interface NotificationObserver {
    void onEvent(NotificationEvent event);
}

public class EventPublisher {
    private final List<NotificationObserver> observers = new ArrayList<>();

    public void subscribe(NotificationObserver observer) {
        observers.add(observer);
    }

    public void publish(NotificationEvent event) {
        for (NotificationObserver observer : observers) {
            observer.onEvent(event);
        }
    }
}

public class AuditLogObserver implements NotificationObserver {
    @Override
    public void onEvent(NotificationEvent event) {
        System.out.println("[AUDIT] event for user=" + event.getUserId()
                + " template=" + event.getTemplateKey());
    }
}

// ---- The main service: wires strategy + factory + decorator + templates + preferences ----

public class NotificationService implements NotificationObserver {
    private final NotificationChannelFactory channelFactory;
    private final UserPreferenceService preferenceService;
    private final TemplateService templateService;
    private final Map<String, User> usersById;
    private final int maxRetryAttempts;

    public NotificationService(NotificationChannelFactory channelFactory,
                                UserPreferenceService preferenceService,
                                TemplateService templateService,
                                Map<String, User> usersById,
                                int maxRetryAttempts) {
        this.channelFactory = channelFactory;
        this.preferenceService = preferenceService;
        this.templateService = templateService;
        this.usersById = usersById;
        this.maxRetryAttempts = maxRetryAttempts;
    }

    @Override
    public void onEvent(NotificationEvent event) {
        notifyUser(event.getUserId(), event.getTemplateKey(), event.getParams());
    }

    public void notifyUser(String userId, String templateKey, Map<String, String> params) {
        User user = usersById.get(userId);
        if (user == null) {
            System.out.println("Unknown user: " + userId);
            return;
        }

        Set<ChannelType> allowedChannels = preferenceService.getAllowedChannels(userId);
        if (allowedChannels.isEmpty()) {
            System.out.println("User " + userId + " has no allowed channels. Skipping.");
            return;
        }

        Message message = templateService.render(templateKey, params);

        for (ChannelType type : allowedChannels) {
            String address = user.addressFor(type);
            NotificationChannel channel = channelFactory.create(type);
            NotificationChannel withRetry = new RetryingChannel(channel, maxRetryAttempts);
            try {
                withRetry.send(address, message);
            } catch (NotificationException ex) {
                // One channel failing should not stop the other channels.
                System.out.println("Give up on " + type + " for user " + userId
                        + ": " + ex.getMessage());
            }
        }
    }
}
```

```java
// ---- Demo wiring ----

public class Demo {
    public static void main(String[] args) {
        NotificationChannelFactory factory = new NotificationChannelFactory();
        factory.register(ChannelType.EMAIL, EmailChannel::new);
        factory.register(ChannelType.SMS, SmsChannel::new);
        factory.register(ChannelType.PUSH, PushChannel::new);

        TemplateService templateService = new TemplateService();
        templateService.addTemplate(new MessageTemplate(
                "ORDER_SHIPPED",
                "Order {{orderId}} shipped",
                "Hi {{name}}, your order {{orderId}} has shipped."));

        UserPreferenceService preferenceService = new UserPreferenceService();
        preferenceService.setPreference("u1",
                new UserPreference(new HashSet<>(Arrays.asList(ChannelType.EMAIL, ChannelType.SMS))));

        Map<String, User> usersById = new HashMap<>();
        usersById.put("u1", new User("u1", "alice@example.com", "+911234567890", "device-token-1"));

        NotificationService notificationService = new NotificationService(
                factory, preferenceService, templateService, usersById, 3);

        EventPublisher publisher = new EventPublisher();
        publisher.subscribe(notificationService);
        publisher.subscribe(new AuditLogObserver());

        Map<String, String> params = new HashMap<>();
        params.put("orderId", "ORD-100");
        params.put("name", "Alice");

        publisher.publish(new NotificationEvent("u1", "ORDER_SHIPPED", params));
    }
}
```

## How It Works

1. Something happens in the system, for example an order ships. The code that handles the
   shipping calls `publisher.publish(new NotificationEvent(...))`.
2. `EventPublisher` loops over every subscribed observer and calls `onEvent`. This is the
   **Observer** pattern: the publisher does not know or care what the observers do.
3. `NotificationService.onEvent` looks up the user, then asks `UserPreferenceService` which
   channels this user allows (for example, Email and SMS, but not Push).
4. `TemplateService` fills in the template's placeholders (`{{orderId}}`, `{{name}}`) with
   real values and returns a `Message` object with a subject and a body.
5. For each allowed channel, `NotificationChannelFactory` builds the matching
   `NotificationChannel` object. This is the **Factory** pattern: the service asks for "an
   EMAIL channel" and does not need to know the class name `EmailChannel`.
6. The service wraps the channel in a `RetryingChannel`. This is the **Decorator** pattern:
   retry logic wraps around the real channel, without changing `EmailChannel`,
   `SmsChannel`, or `PushChannel` at all.
7. `RetryingChannel.send` calls the real channel. If it throws an exception, it tries again,
   up to `maxAttempts` times, before giving up.
8. Because `NotificationChannel` is an interface (the **Strategy** pattern), the service code
   that sends a message is exactly the same for every channel type. Only the object placed
   behind the interface changes.

## How to Extend (Follow-ups)

- **Add a WhatsApp channel.** Write `WhatsAppChannel implements NotificationChannel`. Call
  `factory.register(ChannelType.WHATSAPP, WhatsAppChannel::new)`. No existing class changes.
  This is the Open/Closed Principle in action.
- **Different retry rules per channel.** Give `RetryingChannel` a backoff strategy (for
  example, wait 1s, then 2s, then 4s between attempts) using its own small `BackoffPolicy`
  interface, instead of retrying immediately.
- **Per-channel formatting.** Some channels need a short message (SMS has a length limit).
  Add a `MessageFormatter` per channel, applied after `TemplateService.render` and before
  `send`.
- **Notification history / read receipts.** Add a `NotificationRecord` saved to a database
  after each `send`, so support staff can see what was sent and when.
- **Async sending and a queue.** Right now, `notifyUser` sends on the caller's thread. In a
  real system, put the `NotificationEvent` on a message queue (like Kafka, RabbitMQ, or SQS)
  instead of calling `NotificationService` directly. A separate pool of worker threads (or a
  separate service) reads from the queue and calls the same `NotificationService` code. The
  class design in this document does not change — only what calls `notifyUser` changes, from
  "the event publisher, in-process" to "a queue consumer." This is the usual bridge between an
  LLD answer and an HLD (distributed) answer for the same problem.
- **Rate limiting per user.** Add a check before sending: has this user already received N
  notifications in the last hour? This fits well as another decorator around
  `NotificationChannel`, next to `RetryingChannel`.

## Complexity & Thread-Safety Notes

- Sending one notification to `k` allowed channels costs `O(k)` calls, each independent.
  A slow or failing channel does not block the others in this design (each is tried and
  caught in its own `try` block).
- Retry adds up to `maxAttempts` calls per channel in the worst case. This is a constant
  factor, not a growth in complexity, but it does add latency if the code runs synchronously.
- `UserPreferenceService` uses `ConcurrentHashMap`, so many threads can read and update
  preferences safely at the same time, without an external lock.
- `NotificationChannelFactory`'s registry map is filled once at startup and then only read.
  If channels are registered later while the app is running, use `ConcurrentHashMap` there
  too, or make registration happen only during startup, before any thread reads it.
- `NotificationChannel` implementations should be stateless (no shared mutable fields), so
  the same instance can be reused safely across threads. The sample `EmailChannel`,
  `SmsChannel`, and `PushChannel` have no state, so this holds.
- `EventPublisher.publish` calls each observer directly and one at a time (synchronous). If
  one observer is slow, it delays all observers after it. In a real system, publishing to a
  queue (see the async note above) removes this problem.

## Interview Tips & Common Mistakes

- Do not hard-code `if (channelType == EMAIL) { ... } else if (...)` inside
  `NotificationService`. This breaks the Open/Closed Principle. Use the Strategy + Factory
  pair shown here instead, and say so out loud — interviewers listen for this.
- Do not put retry logic inside each channel class. Repeating the same retry loop in
  `EmailChannel`, `SmsChannel`, and `PushChannel` violates DRY (Don't Repeat Yourself). Use
  one Decorator instead.
- Do not confuse this LLD with the HLD (system design) version of "Notification System."
  In an LLD/machine-coding round, the interviewer wants classes, interfaces, and patterns —
  not Kafka topics or database schemas. Mention the async/queue idea briefly at the end, as
  a bridge, but do not spend the round drawing boxes and arrows for servers.
- Keep `Message`, `User`, and `NotificationEvent` immutable (final fields, no setters).
  This avoids bugs where one part of the code changes a message after another part already
  read it.
- Explain the Observer pattern clearly: `EventPublisher` (subject) does not know about
  `NotificationService` or `AuditLogObserver` by name. It only knows the
  `NotificationObserver` interface. This is what makes it easy to add more observers later
  (for example, a `MetricsObserver` that counts events) without touching `EventPublisher`.
- If asked "what if sending must never block the caller," say clearly: move the call from
  direct method call to a queue, and let a separate worker do the sending. Do not try to
  build a full async framework live in the interview — naming the idea and showing where it
  plugs in is enough at the 3–4 year level.
