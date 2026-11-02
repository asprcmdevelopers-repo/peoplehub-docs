# Admin: employees and rehire review — help and FAQ

Plain-English answers about two Swagger groups that belong together:

- **Admin: employees** — the admin's view of every person in PeopleHub, and how to link an old code to the current one (or undo a
  wrong link). In the Admin UI: the **Employees** page and each person's page.

- **Admin: rehire review** — the suggestions PeopleHub makes when a new code looks like someone who left. In the Admin UI:
  **Rehire review**.

**Contents**

1. [The basics](#1-the-basics)
2. [The four employee endpoints](#2-the-four-employee-endpoints)
3. [Old codes (aliases) explained](#3-old-codes-aliases-explained)
4. [Rehire review](#4-rehire-review)
5. [Which one do I use?](#5-which-one-do-i-use)
6. [Everyday situations](#6-everyday-situations)
7. [Good to know](#7-good-to-know)
8. [Quick reference](#8-quick-reference)

---

## 1. The basics

**What are these groups for?**
Looking at people the way an admin needs to, and fixing the link between someone's old and new employee code. All endpoints need
the `admin` scope.

**Who is "the admin" in this guide?**
A **person** who signs in to the Admin UI (`/admin`) with their company account — usually someone in HR or IT who looks after the
employee master. Who may sign in is set by IT through the AD/Entra groups in `Admin:AllowedGroups`. A script can call the same
endpoints with an API key that has the `admin` scope; the audit log then shows the script's client name.

**PeopleHub** in this guide means the **software** itself, working automatically during imports. PeopleHub can *suggest* that two
codes are one person, but it **never links them by itself**: a person always decides.

**How is it different from the "Employees" endpoints the apps use?**

| | Apps (`/api/v1/employees`) | Admin (`/api/v1/admin/employees`) |
|---|---|---|
| Fields | Only what the app's scopes allow; the rest is `null` | **Everything**, including extra fields |
| Internal details | No | Manager code and email as greytHR sent them, whether the manager was found, why someone was deactivated, which code replaced an old one |
| History | No | Every change to every field, with who or what made it |
| Search | Filters such as department, active, modified since | Text search over code, **old code**, name, email, department; filter for unresolved managers |
| Changes data | No | Link and unlink old codes |

More on the apps' side: [Employees and Changes FAQ](employees-and-changes-faq.md).

**Can I change a person's name, department etc. here?**
No. greytHR people are changed in greytHR; the next push brings the change. Manually added people (contractors, interns) are edited
under **Manual people** (see the [manual people FAQ](manual-people-faq.md)).

---

## 2. The four employee endpoints

### `GET /api/v1/admin/employees` — search

Finds people by **code, old code, name, email or department**. Part of a word is enough: `pri` finds "Priya".

| Filter | Meaning |
|---|---|
| `search` | The text to look for |
| `source` | `Core` (greytHR through ASP CORE), `Excel` or `Manual` |
| `active` | `true` = only active people, `false` = only leavers; leave out for both |
| `unresolvedManager` | `true` = only people whose manager code or email did not match anyone |
| `includeSuperseded` | `true` = also show old codes that were merged into a newer code (hidden by default) |
| `page`, `pageSize` | 50 per page by default, at most 500 |

Answer: one page of people (sorted by name) plus the **total count**. In the Admin UI: **Employees**, with the same filters
(*Unresolved manager*, *Include old codes*).

**Example:** `?unresolvedManager=true&active=true` — active people whose manager in greytHR needs fixing.

### `GET /api/v1/admin/employees/{code}` — everything about one person

| Part | What it holds |
|---|---|
| **Record** | Every field, including extra fields |
| **Old codes (aliases)** | Each old code: kind (*Rehire* or *ManualConversion*), linked when, by whom, note |
| **History** | Every change: field, old value, new value, who or what changed it (an import run or `admin:<name>`), when. The history of the person's **old codes is included**, so a rehire's whole story is in one list. Newest 200 by default (`historyLimit`, at most 2000). |
| **Rehire suggestions** | Possible rehire / duplicate suggestions involving this person, with signal and status |
| **Direct reports** | People whose manager is this person (active first) |

You can give an **old code** too — you get the current person. Unknown code → `404`.

### `POST /api/v1/admin/employees/{code}/aliases` — "this old code is the same person"

```json
{ "aliasCode": "E1001", "note": "Rehired in 2026, confirmed by HR" }
```

`{code}` is the **current** code, `aliasCode` the **old** one. The note is optional. In the Admin UI: the person's page →
**Link an old code to this employee**.

What happens is described in [section 3](#3-old-codes-aliases-explained). It is refused when:

| Answer | Reason |
|---|---|
| `400` | A code linked to itself; the current code is a manually added person (it must be a greytHR code) |
| `404` | One of the codes does not exist |
| `409` | One of the codes is already linked to someone else, or the old code is **still active in greytHR** |

### `POST /api/v1/admin/employees/{code}/aliases/{aliasCode}/unlink` — undo a wrong link

```json
{ "reason": "Two different Priya Sharmas, linked by mistake" }
```

The safety net when someone said "same person" for two different people. **A reason is required.** In the Admin UI: the person's
page → **Old codes** → *Wrong link?* → reason → **Unlink**.

| Answer | Reason |
|---|---|
| `400` | No reason given |
| `404` | These two codes are not linked to each other |

---

## 3. Old codes (aliases) explained

**What is an old code?**
A code that used to belong to a person who now has a different code. Two common cases:

- **Rehire:** Priya left as `E1001` and came back as `E2050`.
- **Manual person converted:** a contractor added as `M-0007` joined greytHR as `E2051`.

**Why link them?**
So every app knows they are **one person**. Without the link, Timesheet would see a leaver `E1001` and a new joiner `E2050`, and
Priya's old timesheets would sit under a different person.

**What happens when I link `E1001` to `E2050`?**

- `E1001` now points to `E2050`: apps asking for `E1001` get `E2050`'s record.
- `E1001`'s history is kept and shows up on `E2050`'s page.
- Apps receive **one `employee.alias_linked` event** (not a "left" event for `E1001`) and merge their own data.
- An open rehire suggestion for this pair is closed as *Linked*.
- If `E1001` already had older codes of its own, they now point to `E2050` too. There is never a chain of old codes.

**Why must the old code be a leaver?**
So two people who are **both working** cannot be merged by mistake. Only a leaver's code can become an old code. A manually added
person (`M-xxxx`) is the exception: converting them is exactly the point.

**Why must the current code be a greytHR code?**
The current code is the one that lives on. A manual person can be the *old* code (converted to greytHR), never the *current* one.

**What happens when I unlink?**
The old code becomes its **own record again, exactly as it was before the link**:

- a leaver stays a leaver,
- a converted contractor becomes active again,
- older codes that came along with the link go back to it.

On top of that:

- the rehire suggestion becomes *Dismissed*, so it is not offered again as the same person,
- apps receive **`employee.alias_unlinked`** and split what they merged (the sample app puts the person on its review list),
- the unlink is kept with the original link details and the reason.

**Where else are codes linked?**
Most links are made elsewhere; the aliases endpoint is for the cases PeopleHub did not spot.

| Where | When |
|---|---|
| **Rehire review → Same person** ([section 4](#4-rehire-review)) | PeopleHub noticed a new code with a leaver's email (or name and department) |
| **Manual people → person → Convert** | A contractor or intern joined greytHR |
| **Employees → person → Link an old code** | The admin knows they are the same person, but no suggestion was made |

All three send apps the same `employee.alias_linked` event. A wrong link from any of them is undone on the Employees page with
**Unlink**.

---

## 4. Rehire review

**What is a rehire candidate?**
A **suggestion** made by PeopleHub: "this new code may be the same person as that old one". Nothing changes until an admin decides
*Same person* (link) or *Different people* (dismiss).

**When does PeopleHub make a suggestion?**
During an import (ASP CORE push or Excel upload), for every **new** code that arrives:

| Signal | Matches | Confidence |
|---|---|---|
| `Email` | Same email as someone who **left**, or as a **manually added person** (a contractor who joined greytHR) | High |
| `NameDepartment` | Same name and department as a leaver or a manual person — only checked when the email found nothing | Low |

Codes that are already linked to someone are not suggested. The same pair is suggested **only once**: a dismissed pair does not come
back on the next import.

**Why does a low-confidence match need an extra tick?**
Two different "Priya Sharma"s in one team happen. Before *Same person* on a name-and-department match, the admin must tick
**I checked both records: same person** (`confirmLowConfidence=true` in the API). Without it the link is refused (`400`).

**What if nobody looks at the suggestions?**
When a suggestion stays open longer than **7 days** (`Alerts:RehireReviewDays`, changeable under **Settings**), PeopleHub raises the
alert "Rehire candidates waiting for review". It closes by itself once nothing that old is open. The dashboard shows the number of
open suggestions too.

### The rehire endpoints

| Endpoint | What it does |
|---|---|
| `GET /api/v1/admin/rehire-candidates` | List suggestions: `status=Open` (default), `Linked` or `Dismissed` — so past decisions can be checked too (who, when) |
| `POST /api/v1/admin/rehire-candidates/{id}/link` | **Same person:** the previous code becomes an old code of the new one. Add `?confirmLowConfidence=true` for a low-confidence match. |
| `POST /api/v1/admin/rehire-candidates/{id}/dismiss` | **Different people:** the suggestion is closed. No employee data changes and apps get no event. |
| `POST /api/v1/admin/rehire-candidates` | The same two actions in one endpoint, with the action in the body: `{ "candidateId": 12, "action": "link" \| "dismiss", "confirmLowConfidence": false }`. A convenience for scripts, not a different feature. |

A suggestion that is no longer open (already linked or dismissed) → `404`. **Link** follows the same rules and does exactly the
same as the aliases endpoint ([section 3](#3-old-codes-aliases-explained)): apps receive `employee.alias_linked`, and the audit log
records `alias.linked` with the note "Rehire candidate N". **Dismiss** is recorded as `rehire.dismissed`.

Suggestions also appear on each person's page under **Possible rehire / duplicate**.

---

## 5. Which one do I use?

**Link: rehire review or aliases endpoint?**
The **result is identical**. What differs is who spotted the match:

| | Rehire review → `.../{id}/link` | Employees → `.../{code}/aliases` |
|---|---|---|
| **Who noticed** | **PeopleHub**, automatically during an import | **The admin** — PeopleHub made no suggestion (for example, the email changed) |
| **Who decides** | The admin | The admin |
| **What you send** | Only the suggestion number; PeopleHub knows both codes | Both codes: current and old |
| **Weak match check** | Low confidence needs the extra tick | None — it is the admin's own decision |
| **Data, events, audit** | Same | Same |

Rehire review is the **inbox of suggestions**; the aliases endpoint is the **manual way** for the cases the inbox missed.

**Dismiss or unlink?**
They are **not** opposites:

| | Rehire review → `.../{id}/dismiss` | Employees → `.../aliases/{old}/unlink` |
|---|---|---|
| **When** | **Before** any link: "these are different people" | **After** a link: "that link was a mistake" |
| **What changes** | Only the suggestion's status | The two records are split again; apps receive `employee.alias_unlinked` |
| **Reason** | Not needed | Required |
| **Audit** | `rehire.dismissed` | `alias.unlinked` |

Unlink lives only on the Employees page because it undoes **any** link — from rehire review, a manual conversion or the aliases
endpoint.

**The whole picture**

```
PeopleHub suggests ──► Rehire review ──► Same person (link) ──┐
                                    └──► Different people     │   (nothing changes)
                                                              ▼
Admin knows better ──► Employees ──► Link an old code ──────► LINKED ──► Employees ──► Unlink (undo)
Contractor joins ────► Manual people ──► Convert ─────────────┘
```

---

## 6. Everyday situations

**Who reports to this manager?**
**Employees → the manager → Direct reports.**

**When did this person's department change, and who changed it?**
**Employees → person → History.** "Run N" means an import changed it (open the run under **Import runs**); `admin:<name>` means
an admin did.

**Which people have a manager PeopleHub cannot find?**
**Employees → tick Unresolved manager.** The manager code or email from greytHR matched nobody. It is re-checked on every import;
if it stays, ask HR to fix the manager in greytHR.

**There is a rehire suggestion. How do I decide?**
Open both codes (the links on **Rehire review**) and compare name, email, department, date of joining and leaving date. If unsure,
ask HR. Same person → **Same person**; different people → **Different people**. Do not leave it open: after 7 days it raises an
alert.

**Someone came back with a new code, but there is no rehire suggestion.**
Usually because their email changed. Open the **new** code → **Link an old code** → the old code, with a note → **Link**.

**An app says it has never heard of code `E1001`.**
Search for `E1001` with **Include old codes** ticked. If it is an old code, the app should use the current code shown; apps that
ask PeopleHub for `E1001` get the current record anyway.

**Two different people were linked by mistake.**
Open the current person → **Old codes** → reason → **Unlink**. Then tell the app owners: each app decides how to split what it
merged.

**Someone was linked to the wrong current code.**
Unlink it first, then link it to the right code.

---

**I clicked "Different people" by mistake.**
A dismissed pair is not suggested again. If they are the same person, link them by hand: new code → **Link an old code**.

---

## 7. Good to know

**Is it recorded?**
Yes, in the **Audit log**:

- `alias.linked` (target `employee:<current code>`) — who linked, the kind (rehire or manual conversion) and the note; a link from
  rehire review has the note "Rehire candidate N",

- `alias.unlinked` (target `employee:<current code>`) — who linked it originally and when, who unlinked, and the reason,
- `rehire.dismissed` (target `rehire:<id>`) — who said "different people" for which two codes.

Search and viewing change nothing, so they are not audited. See the [dashboard and audit FAQ](dashboard-and-audit-faq.md).

**Do imports undo a link?**
No. Imports never remove a link: the old code stays linked to the current person until an admin unlinks it.

**How soon do apps hear about a link?**
The event is written at once. Webhook subscribers get it within seconds; polling apps on their next `GET /changes`.

**How much history is shown?**
The Admin UI shows the newest 500 changes; the API 200 by default, up to 2000 with `historyLimit`. Nothing is deleted.

---

## 8. Quick reference

| Endpoint | In simple words |
|---|---|
| `GET /admin/employees` | Search everyone, with admin filters |
| `GET /admin/employees/{code}` | Everything about one person: old codes, full history, rehire suggestions, team |
| `POST /admin/employees/{code}/aliases` | "This old code is the same person" (admin's own decision) |
| `POST /admin/employees/{code}/aliases/{old}/unlink` | "That link was wrong — split them again" |
| `GET /admin/rehire-candidates` | PeopleHub's suggestions (open, linked or dismissed) |
| `POST /admin/rehire-candidates/{id}/link` | Suggestion accepted: same person |
| `POST /admin/rehire-candidates/{id}/dismiss` | Suggestion rejected: different people |
| `POST /admin/rehire-candidates` | Link or dismiss, with the action in the body |

| Situation | Action |
|---|---|
| Open rehire suggestion | Rehire review → compare both records → Same person / Different people |
| Rehire not spotted by PeopleHub | New code → Link an old code |
| Wrong link | Current code → Old codes → Unlink (with reason) |
| Manager not found | Employees → Unresolved manager → ask HR to fix greytHR |
| Who changed a field | Person → History |
| Old code unknown to an app | Search with Include old codes |

See also: [client library FAQ](client-library-faq.md) · [Employees and Changes FAQ](employees-and-changes-faq.md) (what apps receive) · [import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) ·
[clients and API keys FAQ](clients-and-api-keys-faq.md) · [webhooks FAQ](webhooks-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) · [manual people FAQ](manual-people-faq.md).
