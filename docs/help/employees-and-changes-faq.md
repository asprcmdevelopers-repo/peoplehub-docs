# Employees and Changes endpoints — help and FAQ

Plain-English answers about how apps **get** employee data from PeopleHub: the endpoints under **Employees** and **Changes** in
Swagger, how they work together with webhooks, and the questions that usually come up. Examples use the local demo data
(employees `ASP0001…`, manual people `M-0001…`, import runs #1–#6).

**Contents**

1. [The big picture: how apps get data](#1-the-big-picture-how-apps-get-data)
2. [Rules for every read endpoint](#2-rules-for-every-read-endpoint)
3. [The six Employees endpoints](#3-the-six-employees-endpoints)
4. [Full resync](#4-full-resync)
5. [Match: finding the employee behind an existing app user](#5-match-finding-the-employee-behind-an-existing-app-user)
6. [The Changes endpoint](#6-the-changes-endpoint)
7. [How often should an app poll?](#7-how-often-should-an-app-poll)
8. [Manual people (interns, contractors) and the push](#8-manual-people-interns-contractors-and-the-push)
9. [Trying it in Swagger](#9-trying-it-in-swagger)
10. [Quick reference](#10-quick-reference)

---

## 1. The big picture: how apps get data

**ASP CORE pushes data to PeopleHub. How do the apps get it — through the Employees endpoints?**
Partly. Apps use **three** ways together:

```
ASP CORE ──push──► PeopleHub ──┬── 1. Employees endpoints   app asks: "give me everyone" / "who is ASP0007?"
                               ├── 2. GET /changes          app asks: "what changed since event 101?"
                               └── 3. Webhooks              PeopleHub calls the app: "ASP0045 just left"
```

| Way | Used for |
|---|---|
| **Employees endpoints** | The **first full load** (copy everyone on day one) and occasional live lookups (one person, their manager, their team). |
| **`GET /changes`** | Keeping the copy up to date: the app asks every few minutes for the changes since the last one it handled. |
| **Webhooks** | The same changes, sent by PeopleHub within seconds. If the app is down, PeopleHub retries for about an hour. |

The Employees endpoints alone are not enough: they show how things are **now**, not what **changed**. To notice that ASP0045
resigned, an app would have to reload all employees and compare. That is what `/changes` and webhooks are for.

**Does every app call these endpoints itself?**
No. Apps add the **`PeopleHub.Client`** package, which:

1. does the full load on first start (Employees endpoints),
2. then polls `/changes` and receives webhooks,
3. keeps a **local copy** of employees in the app's own database,
4. calls the app's own code for each event — `OnJoinedAsync` (create a login), `OnLeftAsync` (disable it), `OnManagerChangedAsync`
   (move approvals), and so on.

So day to day an app reads its **own local copy** — fast, and still working when PeopleHub is down. The sample app
(http://localhost:5190 locally) works exactly this way: after the Excel upload (run #6), ASP0061 appeared on `/users` and ASP0045 was
disabled.

**Does PeopleHub create logins in the apps?**
No. PeopleHub only tells the app that someone joined. Whether they can log in depends on the app: its `OnJoinedAsync` must create the
account, and if the app signs in with company AD accounts, IT must also have created the AD account.

---

## 2. Rules for every read endpoint

**Which key do I need?**
Each app has its **own API key** (Admin UI → **Client apps**). What the key may see depends on its scopes:

| Scope | Fields the app sees |
|---|---|
| `employees:read:basic` | code, greytHR ID (`hrmsId`), name, email, active, status, shift (e.g. General, US Night), source, old codes (aliases) |
| `employees:read:org` | + department, designation, manager, location |
| `employees:read:dates` | + date of joining, leaving date |
| `employees:read:extra` | + extra fields (e.g. Grade) |

A field the key cannot see comes back as `null`. Two apps asking for the same person can get different answers — on purpose.

**Is it recorded who read what?**
Yes. Every call is audited: **Client apps → app → Recent reads** shows what the app read and how many records it got.

**Is there a limit on how often an app may call?**
Yes, per app: **600 requests a minute** by default (can be set per app). Above that the app gets `429 Too Many Requests` and should
wait a moment.

---

## 3. The six Employees endpoints

### `GET /api/v1/employees` — the list of employees

"Give me everybody", one page at a time. Used for the **first full load**. Scope: basic.

| Input | Meaning |
|---|---|
| `page`, `pageSize` | Which page and how many per page (200 by default, at most 1000). |
| `active` | `true` = only current employees, `false` = only leavers, empty = everyone. |
| `department` | Only one department. Needs the `org` scope (403 without it). |
| `source` | `Core` (ASP CORE), `Excel` (HR upload) or `Manual` (added in PeopleHub). |
| `modifiedSince` | Only people changed after this date and time. |
| `includeSuperseded` | Also return old records retired by a rehire link (hidden by default). |

The answer has `items`, `page`, `pageSize`, `totalCount` and a **`cursor`**. The app saves the cursor and then continues with
`GET /changes?cursor=…`, so nothing that changes during or after the load is missed.

If nothing has changed since the app last asked, PeopleHub answers **`304 Not Modified`** with no body (it compares the ETag the app
sends back) — a quick "you already have this".

### `GET /api/v1/employees/{employeeCode}` — one person

"Tell me about ASP0007." Scope: basic. Unknown code → `404 Employee not found`.
Asking for an **old code** (e.g. ASP0100, who rejoined as ASP0890) returns the **current** record; the `Content-Location` header tells
the current code. Apps that still store old codes keep working.

### `GET /api/v1/employees/{employeeCode}/manager` — the person's manager

"Who does ASP0007 report to?" Returns the manager's full record. Scope: org. There are three different `404` answers:

| Answer | Meaning |
|---|---|
| `employee_not_found` | The person does not exist. |
| `no_manager` | The person has no manager (e.g. the CEO). |
| `manager_unresolved` | A manager was given but matches nobody (run #3: ASP0030's manager ASP9999). Checked again on every import. |

### `GET /api/v1/employees/{employeeCode}/reports` — the person's team

"Who reports directly to this manager?" Direct reports only, not the whole tree. `active=true` leaves out leavers. Scope: org.
Typical use: approvals, "my team" pages.

### `POST /api/v1/employees/match` — find the employee behind an existing app user

Used once, when an app that already has users joins PeopleHub. See [section 5](#5-match-finding-the-employee-behind-an-existing-app-user).
Scope: basic.

### `GET /api/v1/departments` — departments and head-count

Every department with `activeCount` and `totalCount`. Scope: org. Also answers `304` when nothing changed. Typical use: a department
drop-down.

---

## 4. Full resync

**What is a resync?**
Each app keeps its own copy of the employee list, kept correct by applying changes one by one. Sometimes the copy goes wrong as a
whole — the app's database was restored from an old backup, or rows were deleted by mistake — and nobody knows what is missing. The
simple fix is: **throw the copy away and copy everyone again.** That is a resync.

**How does an admin ask an app to resync?**
The admin cannot reach into the app, so PeopleHub leaves it a note:

1. Admin UI → **Client apps → the app → Request full resync**. PeopleHub notes "this app must reload".
2. The next time the app asks `GET /changes`, the answer says `resyncRequired: true`: "don't continue, reload everything".
3. The app (the client library does it by itself) loads `GET /employees` page 1, 2, … and rebuilds its copy, then continues from the
   cursor that answer gives.

**What does "a full load clears the resync request" mean?**
When the app starts that full reload, PeopleHub removes the note — otherwise every later `/changes` would keep saying
`resyncRequired: true` and the app would reload again and again. Only a **real full reload** counts: page 1 with no filters. A call
like `GET /employees?department=Billing` reads one department, not a full copy, so the note stays.

In one line: the admin says "reload", the app reloads, PeopleHub ticks the request off.

**Is there another reason for `resyncRequired: true`?**
Yes: the app was away too long. PeopleHub keeps events for **180 days**; an app coming back with an older cursor cannot continue and
must reload everything.

---

## 5. Match: finding the employee behind an existing app user

**What problem does it solve?**
Take Timesheet today: it already has hundreds of users, but it was built before PeopleHub, so its users have **no employee code**:

| Timesheet user id | Name | Email | Department |
|---|---|---|---|
| 17 | Priya Sharma | priya.s@example.com | Billing |
| 18 | Ravi Kumar | *(empty)* | Coding |

If Timesheet simply connects and loads all employees, it creates **Priya Sharma a second time** — the old user 17 plus a new one from
PeopleHub — and her old timesheets stay on the old one. To avoid duplicates, **before connecting** Timesheet must find out "my user 17 is
PeopleHub employee ASP0012" and store that code on user 17. Match is the helper that finds it.

**What does it do?**
You send what you know about **one** app user (any of `employeeCode`, `email`, `name`, `department`; at least one), and PeopleHub
answers with up to 20 guesses, best first:

```json
POST /api/v1/employees/match
{ "name": "Priya Sharma", "email": "priya.s@example.com", "department": "Billing" }
```
```json
[ { "employee": { "employeeCode": "ASP0012", "name": "Priya Sharma", … }, "confidence": "High", "signal": "Email" } ]
```

"I found ASP0012, and I'm sure, because the **email** is the same." How sure depends on what matched:

| Matched on | Confidence | Why |
|---|---|---|
| Employee code (old codes included) | High | It is the ID |
| Email | High | Each person has their own email |
| Name + department | Medium | Usually right, but two Priya Sharmas can work in Billing |
| Name only | Low | Could easily be someone else |

For Ravi (no email) PeopleHub might return two Ravi Kumars, both Low — a person has to decide.

**Who calls it?**
Nobody by hand: the **`peoplehub-backfill`** tool calls it for every user.

1. **match** — the app exports its users to `users.csv`; the tool asks Match for each and writes `review.csv`
   (*user 17 → ASP0012, High, Email*; *user 18 → ASP0230?, Low, Name*).

2. **review** — someone from the app team checks the list: yes to the sure ones, checks or corrects the unsure ones.
3. **apply** — the tool writes SQL (`UPDATE users SET employee_code = 'ASP0012' WHERE id = 17`), and the team runs it on the app's
   database.

Now every user carries their code, and when the app connects, PeopleHub's Priya is recognised as the existing user 17 — no
duplicate. Match only **suggests**; a person confirms. New apps without existing users never need it. Apps whose users already
have codes (Timesheet) run it once as a **check**: `review.csv` shows each user's `currentCode` next to the `proposedCode`, and
`apply` writes SQL only for the codes that change (for example an old code of someone rehired). Steps in
[onboarding.md](../onboarding.md#4-backfill-existing-users-only-if-the-app-already-has-users).

---

## 6. The Changes endpoint

### What is `GET /api/v1/changes`?

"What has changed since I last asked?" The `/employees` list is read once; after that the app needs only the **changes**. PeopleHub
keeps a **numbered list of changes** (events), and each app reads just the new ones. Scope: basic.

### The numbered list

Every change, whatever caused it, becomes one **event** with a number that only goes up:

```
#101  ASP0007  employee.updated        designation: Associate → Team Lead   (night push, run #3)
#102  ASP0012  employee.left                                                (night push, run #3)
#103  M-0001   employee.joined         new intern                           (admin, 11:00)
#104  ASP0890  employee.alias_linked   old code ASP0100                     (admin, rehire review)
```

Events come from **imports** (push, Excel upload, replay, approve) **and admin actions** (manual people, rehire link, unlink) — all
in the same list. A run rejected by the safety brake applies nothing, so it adds no events.

### How an app uses it — the bookmark

The number the app keeps is the **cursor**, like a bookmark: "I have read up to 102; what comes after?"

```
Day 1, 10:00  GET /employees (all pages)        → copies everyone, gets cursor = 100
       10:05  GET /changes?cursor=100           → events #101, #102  → applies them, saves 102
       10:10  GET /changes?cursor=102           → nothing new         → keeps 102
       11:05  GET /changes?cursor=102           → event #103 (intern) → creates the intern's account, saves 103
```

### What you send

| Input | Meaning |
|---|---|
| `cursor` | The last event number the app has handled (0 = from the start). |
| `limit` | At most how many events to return (500 by default, 1000 at most). |

### What you get back

```json
{
  "events": [ { "eventId": "…", "sequence": 103, "type": "employee.joined", "employeeCode": "M-0001",
                "occurredAt": "…", "runId": null, "changes": [], "employee": { …full current record… } } ],
  "nextCursor": 103,
  "hasMore": false,
  "resyncRequired": false
}
```

| Field | Meaning |
|---|---|
| `events` | The changes, oldest first — apply them in this order. |
| `nextCursor` | The new bookmark: save it, send it as `cursor` next time. |
| `hasMore` | `true` = more events are waiting than `limit` allowed; ask again straight away (e.g. a big push gives 600 updates in two pages). |
| `resyncRequired` | `true` = stop and reload everything ([section 4](#4-full-resync)). |

Each event has `eventId` (unique), `sequence` (its number), `type`, `employeeCode`, `occurredAt`, `runId` (the import run that caused it;
empty for admin actions), `changes` (fields that changed, old → new), `employee` (the full record **as it is now**) and, for link /
unlink events, `aliasCode`.

### The seven event types

| Type | Meaning | What an app typically does |
|---|---|---|
| `employee.joined` | New person | Create an account |
| `employee.updated` | Some fields changed | Update the copy; maybe change a role |
| `employee.left` | Became inactive | Disable the account |
| `employee.reactivated` | Active again | Re-enable the account |
| `employee.manager_changed` | New manager | Move pending approvals |
| `employee.alias_linked` | Old and new code are one person (rehire) | Merge the two users |
| `employee.alias_unlinked` | That link was a mistake | Split them again, or flag for checking |

### Does every app see every event?

No — events are filtered by the app's key like the Employees endpoints. Hidden fields are removed from `employee` and from `changes`.
If an `updated` event changed **only** hidden fields (a department change, for a basic-only app), the app does not get that event at
all. The bookmark still moves past it, so the app never gets stuck.

### Changes vs webhooks

Same events, two ways of delivery:

| | `/changes` (polling) | Webhooks |
|---|---|---|
| Who starts it | The app asks | PeopleHub sends |
| How fast | At the next poll (5 minutes by default) | Within seconds |
| If the app was down | Catches up on its next poll | Retries for about an hour, then dead letter |

Apps normally use **both**: webhooks for speed, polling as the safety net. Nothing is applied twice — every event has a unique
`eventId` and number.

### Is this the same as the import run's "changes"?

No. `GET /api/v1/admin/import-runs/{id}/changes` is an **admin** view of what one run did. `GET /api/v1/changes` is the running list
**apps** read: every run plus every admin action.

---

## 7. How often should an app poll?

**ASP CORE pushes only once a night. Is polling every 5 minutes needed?**
Not always — but it is not wasted either:

1. **The night push is not the only source of changes.** Admin actions during the day create events at once:

   | Daytime action | Event apps receive |
   |---|---|
   | Add a contractor or intern (Manual people) | joined |
   | Edit or deactivate a manual person | updated / left |
   | Convert a contractor to a greytHR code | alias_linked |
   | Rehire review: *Same person* | alias_linked |
   | Unlink a wrong link | alias_unlinked |
   | Excel upload (when the push failed) | joined / updated / left |
   | Replay a run, approve a rejected one | joined / updated / left |
   | An extra push from ASP CORE (urgent fix) | any |

   An intern added at 11:00 reaches Timesheet by about 11:05 with a 5-minute poll — not the next day. And the IT contract allows
   "nightly **or more often**", so apps need no change if ASP CORE later pushes more often.

2. **An empty poll costs almost nothing**: `{"events":[],"nextCursor":…,"hasMore":false}`, one quick lookup. 5 minutes is 288 calls a
   day per app, far below the limit of 600 a minute.

**Does the intern case need an ASP CORE push?**
No. Manual people are added **directly in PeopleHub**; the event is written the moment the admin saves. No push, no import run.

**Can the interval be longer?**
Yes — a setting in each app, no code change:

```json
"PeopleHub": { "LocalSync": { "PollInterval": "00:30:00" } }
```

| The app… | Suggested poll interval |
|---|---|
| has **webhooks** on | **30–60 minutes** — webhooks bring changes in seconds; polling only catches what a webhook missed |
| has **no webhooks**, and daytime changes matter (Timesheet, approvals) | **5–15 minutes** |
| has no webhooks and only needs the nightly data (a report, a directory) | **1–6 hours** |

---

## 8. Manual people (interns, contractors) and the push

**Someone added directly in PeopleHub is not in ASP CORE's push. Will PeopleHub deactivate them as "missing"?**
No. The rule "missing from the list = has left" applies **only to people from an HR feed**. Every person has a **source**:

| Source | Came from | Does a push touch them? |
|---|---|---|
| `Core` | ASP CORE push (greytHR) | Yes: updated by each push; deactivated if missing from it |
| `Excel` | HR's Excel upload | Yes, the same as Core |
| `Manual` | Admin UI → **Manual people** (interns, contractors, vendor staff) | **Never** |

Imports never touch manual people: they are **not deactivated** when missing from a push or Excel sheet, **not updated** by an
import, and **not counted** by the safety brake (the 5 % is worked out over Core and Excel people only).

**Could a greytHR record overwrite a manual person by accident?**
No. Manual people get codes starting with **`M-`** (`M-0001`, `M-0002`…). A pushed row with an `M-` code is skipped with an error
("prefix reserved for manually added people").

**When does a manual person become inactive?**
Only when an admin deactivates them in **Manual people** (e.g. the internship ended).

**What if the intern later joins greytHR?**
The next push creates them as a new Core employee (say ASP0950, same email). **Rehire review** then suggests "M-0001 and ASP0950 may be
the same person"; the admin clicks *Same person* (or uses **Manual people → Convert**). M-0001 becomes an old code of ASP0950, apps
receive `employee.alias_linked` and merge, and from then on the person is managed by the push like everyone else.

Everything about adding, editing, deactivating and converting manual people is in the [manual people FAQ](manual-people-faq.md).

---

## 9. Trying it in Swagger

1. Admin UI → **Client apps** → create an app with the read scopes you want and copy its key. Admin keys are for the admin endpoints;
   these endpoints are for apps.

2. Swagger (`/swagger`) → **Authorize** → paste the key.
3. **Employees:** try `GET /employees`, then `GET /employees/ASP0007`. To see scopes at work, make two keys (only `basic`, and
   everything) and compare the answers.

4. **Changes:** call `GET /changes` with `cursor=0` — every event from the start (runs #2–#6). Call again with the `nextCursor` you got:
   `"events": []`. Add a manual person in the Admin UI and call again: one `employee.joined` event.

---

## 10. Quick reference

| Endpoint | Question it answers | Scope |
|---|---|---|
| `GET /employees` | Everybody (paged, filterable)? Gives the starting cursor. | basic |
| `GET /employees/{code}` | Who is this person? (old codes work) | basic |
| `GET /employees/{code}/manager` | Who is their manager? | org |
| `GET /employees/{code}/reports` | Who is in their team? | org |
| `POST /employees/match` | Which employee is my existing user? (backfill, once) | basic |
| `GET /departments` | Which departments, how many people? | org |
| `GET /changes?cursor=N` | What changed since event N? | basic |

| I want to… | Do this |
|---|---|
| Make an app reload everything | **Client apps → app → Request full resync** |
| See what an app has read | **Client apps → app → Recent reads** |
| Get changes faster than polling | Turn on webhooks ([onboarding.md](../onboarding.md#5-start-syncing-and-turn-on-webhooks)) |
| Add someone who is not in greytHR | **Manual people → Add person** |

See also: [client library FAQ](client-library-faq.md) · [import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) · [clients and API keys FAQ](clients-and-api-keys-faq.md) · [webhooks FAQ](webhooks-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) · [manual people FAQ](manual-people-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) · [onboarding](../onboarding.md) (connecting an app).
