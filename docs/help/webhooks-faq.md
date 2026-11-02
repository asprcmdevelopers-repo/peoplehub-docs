# Webhooks — help and FAQ

Plain-English answers about webhooks: what they are, how deliveries and retries work, the **Admin: webhooks** endpoints (any app's
subscriptions and the delivery log) and the **Webhooks** endpoints (an app managing its own subscriptions). In the Admin UI this is
under **Webhooks** and **Webhooks → Delivery log**.

**Contents**

1. [The basics](#1-the-basics)
2. [Deliveries, retries and dead letters](#2-deliveries-retries-and-dead-letters)
3. [Admin: webhooks — subscriptions](#3-admin-webhooks--subscriptions)
4. [Admin: webhooks — the delivery log](#4-admin-webhooks--the-delivery-log)
5. [Webhooks — an app managing its own subscriptions](#5-webhooks--an-app-managing-its-own-subscriptions)
6. [What the app receives](#6-what-the-app-receives)
7. [Everyday situations](#7-everyday-situations)
8. [Good to know](#8-good-to-know)
9. [Quick reference](#9-quick-reference)

---

## 1. The basics

**What is a webhook?**
PeopleHub **calls the app** the moment something changes, instead of the app asking every few minutes. The app gives PeopleHub a web
address (e.g. `https://timesheet.example.com/peoplehub/events`); for every event (*ASP0045 left*) PeopleHub sends an HTTP POST there,
usually within seconds.

**Which words do I need to know?**

| Word | Meaning |
|---|---|
| **Subscription** | "Send this app's events to this address." One app can have more than one. |
| **Delivery** | One event sent to one subscription. 3 apps subscribed and 1 event = 3 deliveries. |
| **Signing secret** | A shared password used to **sign** every delivery. The app checks the signature, so nobody else can send it fake events ("the CEO left"). |

**Do webhooks replace polling `/changes`?**
No. They carry the **same events** — webhooks just bring them faster. Polling stays on as the safety net: anything a webhook missed
arrives with the next poll, and nothing is applied twice (every event has a unique `eventId`). See the
[Employees and Changes FAQ](employees-and-changes-faq.md#6-the-changes-endpoint).

**Which apps can receive webhooks?**
Only client apps with the **`webhooks:subscribe`** scope ([clients and API keys FAQ](clients-and-api-keys-faq.md#2-scopes-what-an-app-may-do)).

---

## 2. Deliveries, retries and dead letters

**What happens to one delivery?**

```
event #103 (M-0001 joined)
   → PeopleHub POSTs it to https://timesheet.example.com/peoplehub/events, signed with the secret
   → Timesheet checks the signature, applies it, answers 200 OK   → delivery "Succeeded"
   → Timesheet down / answers 500 / no answer within 10 s         → delivery "Retrying"
```

**How often does PeopleHub retry?**
After 30 seconds, then 1, 2, 4, 8, 16 and 32 minutes (doubling each time): **8 attempts over about an hour** (`Webhooks:MaxAttempts`,
`BackoffSeconds`). Short network hiccups are also retried quickly within one attempt.

**What is a dead letter?**
A delivery that failed all 8 attempts. PeopleHub stops trying and raises an alert *"Webhook deliveries to timesheet are failing"*
(one alert per subscription, not one per delivery). Fix the app, then **replay** the dead letters.

**Is the event lost when a delivery fails?**
No. The app's next `/changes` poll picks it up anyway, and dead letters can be sent again.

**What do the delivery statuses mean?**

| Status | Meaning |
|---|---|
| Pending | Waiting to be sent (normally a few seconds) |
| Retrying | Failed; will try again later |
| Succeeded | The app answered 2xx |
| Skipped | Not sent: nothing this app may see changed (e.g. a department change for a basic-only app) |
| DeadLetter | Failed 8 times; stopped. Fix the app, then replay. |

---

## 3. Admin: webhooks — subscriptions

All need the `admin` scope; the Admin UI **Webhooks** page uses these.

### `GET /api/v1/admin/webhooks` — list subscriptions

Every subscription of every app: URL, event types, active or not, pending and dead-letter counts. Add `clientId` for one app only.

### `GET /api/v1/admin/webhooks/{id}` — one subscription

The same details for one subscription. Unknown id → `404`.

### `POST /api/v1/admin/webhooks` — subscribe an app

An admin registers the webhook **for** an app:

```json
{ "clientId": "…timesheet's id…", "url": "https://timesheet.example.com/peoplehub/events",
  "eventTypes": [], "description": "Timesheet UAT" }
```

- `eventTypes` empty = **all seven event types**; or list some, e.g. `["employee.joined", "employee.left"]`.
- Answer `201` with the subscription **and the signing secret** — **shown only once**. The app team stores it as
  `PeopleHub:WebhookSecret`.

| Error | When |
|---|---|
| `400` | URL not a full address; URL is `http://` (only `https://` is allowed, except in development with `Webhooks:AllowHttp`); unknown event type; the app lacks the `webhooks:subscribe` scope |
| `404` | The client app does not exist or is deactivated |

### `DELETE /api/v1/admin/webhooks/{id}` — stop a subscription

PeopleHub stops sending to that address (`204`). Deliveries not yet sent stay in the log but are no longer sent. The app still gets
its changes by polling.

### `POST /api/v1/admin/webhooks/{id}/secret` — new signing secret

For a leaked secret or a regular change. Returns the new secret (shown once). PeopleHub signs with it **from the very next delivery** —
there is no grace period as with API keys. The safe order:

1. *New secret* in the Admin UI.
2. The app team **adds** the new secret to `PeopleHub:WebhookSecrets` next to the old one (the client library accepts several) and
   restarts.

3. Later they remove the old one.

Deliveries rejected in between are retried for about an hour, and any dead letters can be replayed.

### `POST /api/v1/admin/webhooks/{id}/replay-dead-letters` — send all dead letters again

After the app is fixed: every dead letter of this subscription is queued again with a fresh 8 attempts. Answer `{ "replayed": 12 }`.
**Safe**: the app ignores events it has already applied (same `eventId`).

---

## 4. Admin: webhooks — the delivery log

### `GET /api/v1/admin/webhook-deliveries` — the delivery log

Newest first, 50 per page (at most 500). Filters: `status` (e.g. `DeadLetter`), `subscriptionId`, `employeeCode` ("did Timesheet
get ASP0045's leaving?"). Each entry shows the event, attempts, the last HTTP status code and the last error (e.g. *connection
refused*, *401 invalid signature*). Kept for **90 days** (`Retention:WebhookDeliveriesDays`).

### `GET /api/v1/admin/webhook-deliveries/{id}` — one delivery

All details of one delivery. Unknown id → `404`.

### `POST /api/v1/admin/webhook-deliveries/{id}/replay` — send one delivery again

Queues just this one again (`202`), whatever its status — even a succeeded one. Use it when an app owner says "we never processed
ASP0045's leaving". Safe for the same reason.

---

## 5. Webhooks — an app managing its own subscriptions

**What is the "Webhooks" heading in Swagger (without "Admin:")?**
The same subscriptions, but managed **by the app itself** with its own API key (scope `webhooks:subscribe`). An app can only see and
change **its own** subscriptions; admins see all of them on the Admin UI Webhooks page.

| Endpoint | In simple words |
|---|---|
| `GET /api/v1/webhooks` | List my app's subscriptions |
| `POST /api/v1/webhooks` | Subscribe my app: `{ "url": "…", "eventTypes": [], "description": "…" }`. Answer `201` with the secret (shown once). Same checks as the admin version (https, known event types). |
| `DELETE /api/v1/webhooks/{id}` | Stop one of my subscriptions (`204`); someone else's or unknown → `404` |

In the client library this is one line, run once:

```csharp
await peopleHubClient.SubscribeWebhookAsync(new Uri("https://myapp.example.com/peoplehub/events"));
```

**Admin or self-service — which should we use?**
Either; the result is the same subscription. Self-service suits app teams that set up their own environments; admin subscription suits
apps whose team prefers PeopleHub admins to do it. Rotating the secret and replaying dead letters are **admin-only**.

---

## 6. What the app receives

Every POST carries the same event as `/changes` (JSON), with these headers:

| Header | Meaning |
|---|---|
| `X-PeopleHub-Signature: t=…,v1=…` | The signature: an HMAC-SHA256 of `"{t}.{body}"` with the secret. The app rejects it if the signature is wrong or the time `t` is **more than 5 minutes** off, so a captured message cannot be re-sent later. |
| `X-PeopleHub-Event-Id` | The event's unique id (for de-duplication) |
| `X-PeopleHub-Event-Type` | e.g. `employee.left` |
| `X-PeopleHub-Sequence` | The event's number |
| `X-PeopleHub-Delivery-Id` | The delivery's number in the delivery log |
| `X-PeopleHub-Attempt` | 1 for the first attempt, 2 for the first retry, … |

The app must answer **2xx within 10 seconds** (`Webhooks:TimeoutSeconds`); anything else counts as failed and is retried.

`PeopleHub.Client` does all of this by itself — `app.MapPeopleHubWebhooks("/peoplehub/events")` checks the signature and time,
skips duplicates and calls the app's handler. If the handler throws, the client answers with an error so PeopleHub retries.

---

## 7. Everyday situations

**An app wants changes within seconds.**

1. Give its client the `webhooks:subscribe` scope (**Client apps → app → Edit**).
2. Subscribe it (**Webhooks → Add**), or let the app subscribe itself.
3. Hand over the secret safely; the app stores it as `PeopleHub:WebhookSecret` and restarts.
4. Check **Delivery log**: deliveries should be *Succeeded*.

**Alert "Webhook deliveries to X are failing".**

1. **Delivery log** → filter *DeadLetter* (and the subscription) → read the **last error**:

   | Last error | Usual cause |
   |---|---|
   | connection refused / timeout | The app is down or too slow (must answer within 10 s) |
   | 404 | Wrong URL, or the app no longer maps the webhook path |
   | 401 / invalid signature | The app has the wrong secret (e.g. after *New secret*) |
   | 500 | The app's handler crashed — see the app's logs |
2. The app team fixes it.
3. **Replay dead letters**. (Meanwhile the app has caught up by polling anyway.)

**"Did the app get this change?"**
**Delivery log** → filter by employee code (and subscription).

**The secret has leaked.**
**New secret** → the app adds it next to the old one → restarts → removes the old one later.

**The app moved to a new URL.**
Create a new subscription with the new URL, then stop the old one.

**The app is retired.**
Stop its subscriptions (deactivating the client app also stops it from using the API).

---

## 8. Good to know

- **Each app sees only what its key allows**, in webhooks too: hidden fields are removed, and deliveries with nothing visible are
  *Skipped*.

- **Order:** events are numbered; the client library ignores an older event that arrives after a newer one for the same person, so a
  retry arriving late cannot undo a newer change.

- **How long things are kept:** deliveries 90 days, events 180 days.
- **Every action is in the Audit log**: subscribe, unsubscribe, new secret, replays — with who did it.

---

## 9. Quick reference

| Endpoint | In simple words | Who |
|---|---|---|
| `GET /admin/webhooks` | All subscriptions (`clientId` to filter) | admin |
| `GET /admin/webhooks/{id}` | One subscription | admin |
| `POST /admin/webhooks` | Subscribe an app (secret shown once) | admin |
| `DELETE /admin/webhooks/{id}` | Stop a subscription | admin |
| `POST /admin/webhooks/{id}/secret` | New signing secret, used from the next delivery | admin |
| `POST /admin/webhooks/{id}/replay-dead-letters` | Send all dead letters again | admin |
| `GET /admin/webhook-deliveries` | Delivery log (status, subscription, employee code) | admin |
| `GET /admin/webhook-deliveries/{id}` | One delivery | admin |
| `POST /admin/webhook-deliveries/{id}/replay` | Send one delivery again | admin |
| `GET /webhooks` | My app's subscriptions | app (`webhooks:subscribe`) |
| `POST /webhooks` | Subscribe my app | app |
| `DELETE /webhooks/{id}` | Stop my subscription | app |

See also: [client library FAQ](client-library-faq.md) · [Employees and Changes FAQ](employees-and-changes-faq.md) · [clients and API keys FAQ](clients-and-api-keys-faq.md) ·
[import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) · [manual people FAQ](manual-people-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) ·
[onboarding → step 5](../onboarding.md#5-start-syncing-and-turn-on-webhooks).
