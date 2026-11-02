# Dashboard and audit log — help and FAQ

Plain-English answers about **Admin: dashboard** ("is everything OK right now?") and **Admin: audit** ("who did what, and when?").
Each group has one endpoint. In the Admin UI they are the **Dashboard** (home page) and **Audit log**.

**Contents**

1. [Dashboard: what it is](#1-dashboard-what-it-is)
2. [Dashboard: what each number means](#2-dashboard-what-each-number-means)
3. [Dashboard: good day, bad day](#3-dashboard-good-day-bad-day)
4. [Audit log: what it is](#4-audit-log-what-it-is)
5. [Audit log: what is recorded](#5-audit-log-what-is-recorded)
6. [Audit log vs the other "who did what" lists](#6-audit-log-vs-the-other-who-did-what-lists)
7. [Quick reference](#7-quick-reference)

---

## 1. Dashboard: what it is

**What is `GET /api/v1/admin/dashboard`?**
The morning health check: one call that answers the questions an admin asks first — did last night's data arrive, is the next push
late, and is anything waiting for me? It only reads; it changes nothing. Scope: `admin`. The Admin UI home page shows the same numbers.

**Who uses it besides the Admin UI?**
Monitoring or scripts, for example a small job that checks `overdue` or `deadLetters` every morning and posts to Teams.

**Is it the same as `/health`?**
No. `GET /health` (no key needed) answers "is the **service** running?" (process and database). The dashboard answers "is the
**data** OK?".

---

## 2. Dashboard: what each number means

### Did the data arrive? (per source)

For **Core** (ASP CORE push) and **Excel** (HR upload), separately:

| Field | Meaning |
|---|---|
| `lastRun` | The most recent run of that source, **whatever its result** (e.g. #4 Rejected) |
| `lastSuccess` | The most recent run that **succeeded** (e.g. #5) |

If the two differ, the latest attempt did not work — open that run ([import runs FAQ](import-runs-faq.md)).

### Is the next push late?

| Field | Meaning |
|---|---|
| `lastSuccessAt` | When the latest successful import finished (Core or Excel, whichever is newer) |
| `nextExpectedPushBy` | The **latest** time the next successful import should have arrived: last success + 24 h (`Import:ExpectedIntervalHours`) + 2 h grace (`Import:AlertGraceHours`) |
| `overdue` | `true` once that time has passed: data is getting stale. The alert *"No successful employee import"* fires at the same moment. |

Example: last success Monday 01:30 → `nextExpectedPushBy` Tuesday 03:30. Nothing succeeded by then → `overdue: true`.
An Excel upload counts as a success too — that is why uploading HR's sheet clears "late" when the push is broken.

### How many people?

| Field | Meaning |
|---|---|
| `activeBySource` | Active people per source, e.g. `{ "Core": 58, "Excel": 2, "Manual": 3 }` |
| `inactive` | People who have left (old codes merged by a rehire link are not counted) |

A sudden jump is a reason to look at the latest run.

### What needs an admin's attention?

| Field | Meaning | Where to act |
|---|---|---|
| `unresolvedManagers` | People whose manager code or email matches nobody (e.g. ASP0030 → ASP9999) | **Employees** → filter *Unresolved manager*; usually HR fixes it in greytHR |
| `openRehireCandidates` | "Same person?" suggestions waiting for a decision (an alert fires after 7 days) | **Rehire review** |
| `pendingDeliveries` | Webhook deliveries waiting or retrying. A few for a moment is normal; many that stay means an app is struggling. | **Webhooks → Delivery log** |
| `deadLetters` | Webhook deliveries that failed 8 times | **Webhooks** → fix the app → *Replay dead letters* ([webhooks FAQ](webhooks-faq.md)) |
| `openAlertCount`, `openAlerts` | Unresolved alerts: the count and the latest 10 | **Alerts** |

---

## 3. Dashboard: good day, bad day

**What does a good day look like?**
Core `lastRun` = `lastSuccess` (last night), `overdue: false`, and `unresolvedManagers`, `openRehireCandidates`, `deadLetters`,
`openAlertCount` all 0 or small and known.

**What do bad days look like, and what do I do?**

| You see | It means | Do |
|---|---|---|
| Core `lastRun` is Rejected, `lastSuccess` is older | The safety brake stopped last night's push | Open the run ([import runs FAQ, section 4](import-runs-faq.md#4-safety-brake-approve-and-replay)) |
| Core `lastRun` is Failed | The payload could not be read | Open the run: failure reason and raw payload; usually fix the field map and replay |
| `overdue: true` and `lastRun` is old | ASP CORE did not call at all | Ask the ASP CORE team; meanwhile upload HR's Excel sheet |
| `deadLetters: 40` | An app has been unreachable for more than an hour | Delivery log → last error → app team → replay |
| `openRehireCandidates: 3` | Rehires waiting for a decision | Rehire review |
| `unresolvedManagers` keeps growing | A manager left or was re-coded in greytHR | Employees → *Unresolved manager*; ask HR |

---

## 4. Audit log: what it is

**What is `GET /api/v1/admin/audit`?**
The list of every **admin action** in PeopleHub, newest first: who did what, to what, and when. Scope: `admin`. In the Admin UI:
**Audit log**.

**Can entries be changed or deleted?**
Not from the Admin UI or the API. Entries are written at the moment of the action — for most actions in the same database step as the
change itself, so a change cannot happen without its entry. The nightly clean-up does **not** touch the audit log; it is kept
indefinitely.

**What does one entry show?**

| Column | Example |
|---|---|
| **When** | 01 Nov 2026 11:02 |
| **Who** (actor) | The admin's sign-in name, or the client name of the admin API key |
| **Action** | `client.created`, `key.revoked`, `alias.unlinked`, `run.approved` … |
| **Target** | What it was done to: `employee:ASP0890`, `client:…id…`, `webhook:…id…`, `run:4`, `alert:12`, `settings` |
| **Details** | The specifics, e.g. `{"aliasCode":"ASP0100","reason":"different people"}` or `{"outcome":"Succeeded"}` |

**How do I filter it?**

| Input | Meaning |
|---|---|
| `target` | Only entries whose target **starts with** this text: `employee:` all employee actions, `employee:ASP0890` one person, `client:` client apps and keys, `webhook:` subscriptions, `run:` import runs, `alert:` alerts, `settings` settings |
| `limit` | How many entries (200 by default, at most 1000) |

"Starts with" means `run:4` also shows `run:40`, `run:41`…; check the target column.

**Typical questions it answers**

- Who revoked Timesheet's key yesterday? → `target=client:`
- Who linked ASP0100 to ASP0890, and who undid it? → `target=employee:ASP0890`
- Who changed the safety brake to 10 %? → `target=settings`
- Who approved run 4, and did it work? → `target=run:4`
- Who added intern M-0003? → `target=employee:M-0003`
- Who closed the "missed run" alert? → `target=alert:`

---

## 5. Audit log: what is recorded

| Area | Actions |
|---|---|
| Import runs | `run.uploaded` (Excel upload: file and sheet), `run.replayed` (details give `replayOf`, the original run), `run.approved`. Each carries the **outcome** (`Succeeded`, `Rejected`, `Failed`, `Busy`) and the error if any — failed attempts are recorded too. A refused request (unknown run, not rejected, a later run already succeeded) changes nothing and is not recorded. |
| Client apps and keys | `client.created`, `client.updated` (scopes, active, speed limit), `client.deactivated`, `key.rotated`, `key.revoked`, `client.resync_requested` |
| Webhooks | `webhook.subscribed`, `webhook.unsubscribed`, `webhook.secret_rotated`, `webhook.replayed` (dead letters or one delivery) |
| Manual people | `manual.created`, `manual.updated`, `manual.deactivated`, `manual.reactivated` |
| Links | `alias.linked` (rehire *Same person*, contractor convert), `alias.unlinked` (with the reason), `rehire.dismissed` (*Different people*) |
| Configuration | `settings.saved`, `settings.reset` (includes the field map and the safety-brake limits), `status_mappings.saved`, `status_mappings.deleted` |
| Alerts | `alert.acknowledged`, `alert.resolved` (acknowledging an already acknowledged alert adds nothing) |

**Is ASP CORE's nightly push in the audit log?**
No — it is not an admin action. Every push is an import run: its page shows the caller (the API key's client name), the caller's IP,
the time and the result.

**Is an alert that resolves itself in the audit log?**
No. The alerts PeopleHub closes by itself ("No successful employee import", once an import succeeds again, and "Rehire candidates
waiting for review", once nothing old is open) are not an admin action. Only an admin's *Acknowledge* or *Resolve* is recorded.

**Do actions from before 1 Nov 2026 appear?**
Approvals, replays, Excel uploads and alert acknowledge/resolve were added to the audit log on 1 Nov 2026. For older ones, look on
the run's page (caller, *approved by*) or the alert (*acknowledged by*).

---

## 6. Audit log vs the other "who did what" lists

| List | Records | Kept | Where |
|---|---|---|---|
| **Audit log** | What **admins changed** | Indefinitely | **Audit log** |
| **Read audit** | What **apps read** | 1 year | **Client apps → app → Recent reads** |
| **Employee history** | What changed **in an employee's data**, and from which run or admin | 2 years | **Employees → person → History** |
| **Import run** | Who sent it (caller, IP), who approved it, which run it replayed | The run: indefinitely; raw payload 30 days | The run's own page |
| **Delivery log** | Every webhook delivery and its attempts | 90 days | **Webhooks → Delivery log** |

---

## 7. Quick reference

| Endpoint | In simple words |
|---|---|
| `GET /api/v1/admin/dashboard` | Is everything OK right now? (runs per source, overdue, head-count, things waiting) |
| `GET /api/v1/admin/audit?target=…&limit=…` | Who did what, newest first |

| I want to know… | Look at |
|---|---|
| Did last night's push work? | Dashboard → Core `lastRun` vs `lastSuccess` |
| Is the data stale? | Dashboard → `overdue` |
| What is waiting for me? | Dashboard → unresolved managers, rehire candidates, dead letters, alerts |
| Who changed this setting / key / link? | Audit log → filter by target |
| Who approved or replayed a run? | Audit log → `target=run:N` (or the run's page) |
| What did an app read? | Client apps → app → Recent reads |

See also: [client library FAQ](client-library-faq.md) · [import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) · [Employees and Changes FAQ](employees-and-changes-faq.md) ·
[clients and API keys FAQ](clients-and-api-keys-faq.md) · [webhooks FAQ](webhooks-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) · [manual people FAQ](manual-people-faq.md).
