# Notification System — Simplified Tech Spec

**Author:** Joseph Davies  
**Version:** 1.0

---

## What This System Does

We need to send push notifications, SMS, and emails to over a million users. The system must be rock-solid: no message should ever get lost, and no user should ever receive the same message twice. If a provider like Twilio or SendGrid goes down, the system should keep working with a backup provider and never just silently give up.

---

## How It Works (The Big Picture)

We never let any part of our app talk directly to a notification provider. Instead, every notification is first put into a queue. From there, background workers pick it up, deliver it, and carefully record what happened.

Our App → Queue (per channel) → Workers → Providers (FCM, Twilio, SendGrid)
↓
Tracking Database (Postgres + Redis)


This means if we suddenly need to send a million marketing emails, push and SMS are unaffected—they run in separate queues. If a provider fails, only that channel’s queue slows down, and we switch to the backup automatically.

---

## The Key Pieces

### 1. Notification Gateway (API)
When another service wants to send a notification, it calls this lightweight API. The gateway:
- Checks the request makes sense (valid user, message length, etc.)
- Creates a unique `notification_id` (a random string) that will follow the message everywhere
- Writes a record in the database with the status `QUEUED`
- Drops the message into the right queue (push, sms, or email)
- Does **not** call any provider itself

### 2. Separate Queues for Each Channel
We use a robust message queue (like Amazon SQS or RabbitMQ). There are three distinct queues so that trouble in one channel never blocks the others:
- `notifications.push`
- `notifications.sms`
- `notifications.email`

### 3. Background Workers (the delivery team)
Workers are small programs that constantly listen to a queue. For each message, they:
- Look up the `notification_id` in a fast Redis cache to see if it was already sent (prevents duplicates)
- Mark the database record as `SENDING`
- Call the provider (e.g., SendGrid for email)
- On success: mark as `DELIVERED`
- On failure: retry a few times, waiting longer each time (so we don't hammer a struggling provider)
- If all retries fail, move the message to a "dead letter" area for human review

### 4. Delivery Tracker (Postgres + Redis)
- **PostgreSQL** stores the entire life story of every notification—when it was created, every status change, and the final outcome. This is our permanent audit trail.
- **Redis** holds a fast, temporary list of recently processed `notification_id`s. The workers check this list before calling any provider. Entries expire after a while because once a message is confirmed delivered, we don't need to check it forever.

### 5. Dead Letter Queue (emergency holding area)
Messages that fail even after all retries aren't thrown away. They go to a special queue. A separate monitor watches this queue and alerts the on-call engineer. Once the provider issue is fixed, we can replay those messages manually.

---

## How We Prevent Duplicate Messages

Every notification gets a unique ID at birth. Before a worker ever dials a provider, it quickly asks Redis: "Have I seen this ID recently?" If yes, the message is simply marked as done and thrown away (without calling the provider).

This simple check protects us from three real-world problems:
- The queue delivers the same message twice after a worker crash
- An upstream service accidentally sends the same request twice
- Two workers grab the same message at the same time (race condition)

---

## What Happens When a Provider Fails (Graceful Degradation)

Each channel has a primary provider and a fallback. The table below shows our defaults:

| Channel | Primary   | Fallback   |
|---------|-----------|------------|
| Push    | FCM       | OneSignal  |
| SMS     | Termii    | Twilio     |
| Email   | SendGrid  | AWS SES    |

Workers keep track of recent failures in Redis. If a provider racks up too many errors in a short time (like 10 failures in a minute), we automatically flip a switch and start using the fallback for new messages. The primary provider is tried again periodically until it recovers.

**For non-urgent messages** (marketing, tips): if both primary and fallback are down, the message just sits in the queue and waits.  
**For critical messages** (transaction alerts, one-time passwords): we try everything possible, and if nothing works, the team is paged immediately.

---

## How We Guarantee Reliability

- **No missed sends:** Messages stay in the queue until they are successfully processed, or they end up in the dead letter queue for a human to handle. Nothing is silently lost.
- **No duplicates:** The Redis check before every provider call guarantees each notification is sent exactly once.
- **No lost data on worker crashes:** A message is only removed from the queue after the worker has written a final delivery status to the database. If a worker dies mid-send, the queue will hand the message to another worker.
- **Full traceability:** For any notification, we can see exactly what happened and when—useful for debugging and compliance.

---

## How We Scale to Over a Million Users

- **Add more workers:** Workers are stateless; when traffic spikes, we simply run more of them. They can be scaled independently per channel.
- **Queues absorb bursts:** A sudden wave of 500,000 notifications doesn't crash anything. The queue holds them, and workers process them at a steady pace.
- **Rate limits:** Workers respect each provider’s sending limits so we never get banned or throttled.
- **Batching:** For push, we use FCM’s multicast to send one request for many users. For email, we use SendGrid’s batch API. This cuts down the number of external calls dramatically.

---

## What’s Not Included in Version 1

- User preferences and opt-outs
- An in-app notification center
- A dashboard showing delivery stats 

---
