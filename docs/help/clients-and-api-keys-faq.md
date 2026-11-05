# Clients and API keys — help and FAQ

Plain-English answers about **Admin: clients and API keys**: what a client app is, what scopes and keys are, the nine admin endpoints,
and what to do in everyday situations (new app, key change, leaked key, retired app). In the Admin UI all of this is under
**Client apps**.

**Contents**

1. [The basics](#1-the-basics)
2. [Scopes: what an app may do](#2-scopes-what-an-app-may-do)
3. [API keys](#3-api-keys)
4. [The nine endpoints](#4-the-nine-endpoints)
5. [Everyday situations](#5-everyday-situations)
6. [Good to know](#6-good-to-know)
7. [Quick reference](#7-quick-reference)

---

## 1. The basics

**What is a client app?**
Every program that talks to PeopleHub is registered as a **client app**: Timesheet, the training portal, ASP CORE itself, an admin
script. Each client app has:

- a **name** (e.g. `timesheet`) and an optional description,
- **scopes** — what it is allowed to do,
- one or more **API keys** — the "password" the app sends in the `X-Api-Key` header,
- optionally its own **speed limit** (otherwise the default of 600 calls a minute).

**Why does every app need its own client?**
So PeopleHub knows **who** is calling: each app sees only what its scopes allow, every read is recorded against that app, and one
app's key can be changed or switched off without touching the others.

---

## 2. Scopes: what an app may do

**Which scopes exist?**

| Scope | Allows | Typical app |
|---|---|---|
| `employees:read:basic` | code, greytHR ID (`hrmsId`), name, email, active, status, shift, source, old codes | every reading app |
| `employees:read:org` | + department, designation, manager, location, grade, level; `/manager`, `/reports`, `/departments` | Timesheet (approvals) |
| `employees:read:dates` | + date of joining, leaving date | HR reports |
| `employees:read:extra` | + extra fields (Gender…) | only if really needed |
| `webhooks:subscribe` | register its own webhook | apps that want changes within seconds |
| `import:write` | **push** employee data | **ASP CORE only** |
| `admin` | the admin API (everything under "Admin:" in Swagger) | admin scripts, rarely |

**How do I choose?**
Give each app **only what it needs**. A canteen app needs `basic`, not dates or extra fields. Fields an app may not see come back as
`null`, and change events about only those fields are not sent to it at all (details in the
[Employees and Changes FAQ](employees-and-changes-faq.md#2-rules-for-every-read-endpoint)).

**Can I change an app's scopes later?**
Yes, at any time (*Edit* / `PUT`). The key stays the same, and the change takes effect within seconds.

---

## 3. API keys

**What does a key look like?**
`ph_dev_3f9a1c2b7d4e8f60_9c1e…` (long):

| Part | Meaning |
|---|---|
| `ph` | PeopleHub |
| `dev` / `uat` / `prod` | The environment (`ApiKeys:Environment`). A UAT key used against Prod fails visibly instead of silently reading the wrong hub. |
| `3f9a1c2b7d4e8f60` | A short ID (the *prefix*) PeopleHub uses to find the key. It is shown in the Admin UI. |
| the rest | The secret |

**Why is the key shown only once?**
PeopleHub does **not store the key itself**, only a scrambled fingerprint of it (a salted SHA-256 hash). It can check a key the app
sends, but it cannot show it again. If a key is lost, nobody can look it up — **rotate** and get a new one.

**Can a client have more than one key?**
Yes, for a short time: during a rotation the old and the new key both work (7 days by default). Normally each client has one active
key.

**How do I know a key is still in use?**
Each key shows **last used** (refreshed about once a minute while it is used). A key nobody has used for weeks can usually be revoked.

---

## 4. The nine endpoints

All need the `admin` scope. The Admin UI (**Client apps**) uses exactly these. `{id}` is the client app's id (a GUID).

### `GET /api/v1/admin/clients` — list all client apps

Every app with its scopes, active flag, speed limit and keys. For each key: its short ID, created, expires, revoked, **last used**.
Secrets are never shown.

### `GET /api/v1/admin/clients/{id}` — one client app

The same details for one app. Unknown id → `404`.

### `POST /api/v1/admin/clients` — create a client app

```json
{ "name": "timesheet", "description": "Timesheet UAT",
  "scopes": ["employees:read:basic", "employees:read:org", "webhooks:subscribe"],
  "rateLimitPerMinute": null }
```

Answer `201` with the app **and its first key**. **Copy the key now** — it is never shown again.
`rateLimitPerMinute: null` means the default (600 a minute).

| Error | When |
|---|---|
| `400` | No name or longer than 100 characters; no scopes; an unknown scope (the message lists the valid ones); a speed limit of 0 or less |
| `409` | An app with this name already exists |

### `PUT /api/v1/admin/clients/{id}` — change a client app

Change the **description**, **scopes**, **active** flag or **speed limit**. Only the fields you send change. The key stays the same.
Example: Timesheet now also needs joining dates → send `scopes` with `employees:read:dates` added.

### `DELETE /api/v1/admin/clients/{id}` — deactivate a client app

For an app that is retired. The app is switched off and **all its keys are revoked at once**; its next call gets `401`. Answer `204`.
Despite "DELETE", the app is **not deleted**: it stays in the list as inactive, so the read history still shows whose reads they were.

### `POST /api/v1/admin/clients/{id}/keys` — rotate (issue a new key)

- You get a **new key** (shown once).
- The **old key keeps working for 7 more days** (`ApiKeys:RotationGraceDays`), so the app team can put the new key in their
  configuration and redeploy without downtime.

- After 7 days the old key stops by itself — or revoke it earlier.

Answer: `{ "clientId": "…", "apiKey": "ph_…", "oldKeysValidForDays": 7 }`. A deactivated app → `404`.

### `DELETE /api/v1/admin/clients/{id}/keys/{keyId}` — revoke one key now

The key stops working **immediately** (`204`). Use it after a rotation once the app runs with the new key, or **at once when a key
has leaked**.

### `POST /api/v1/admin/clients/{id}/resync` — ask the app to reload everything

PeopleHub notes the request (`202`). The app's next `GET /changes` says `resyncRequired: true` and the app reloads the full employee
list; the note is cleared when that reload starts. Use it when an app's copy looks wrong as a whole (restored from an old backup,
rows deleted by mistake). More in the [Employees and Changes FAQ](employees-and-changes-faq.md#4-full-resync).

### `GET /api/v1/admin/clients/{id}/audit` — what this app has read

Newest first, 200 by default (`limit` to change): which endpoint, which filters, how many records came back, when. Kept for
**1 year** (`Retention:ReadAuditDays`). In the Admin UI: **Client apps → app → Recent reads**.

---

## 5. Everyday situations

**A new app wants to connect. What do I do?**

1. **Client apps → Add a client app**: name, description, only the scopes it needs.
2. Copy the key.
3. Give it to the app team through a safe channel (a password manager or secret store, **not** email or chat).
4. They store it in their secret store or environment variable, never in source code.
5. Point them to [onboarding.md](../onboarding.md).

**An app needs one more field.**
**Client apps → app → Edit** → add the scope. No new key needed.

**Regular key change, or a developer who knew the key leaves.**

1. **Rotate key** → copy the new key → give it to the app team.
2. The app team deploys the new key (they have 7 days; the old one keeps working).
3. **Revoke** the old key, or let it expire by itself.

**A key has leaked** (pasted in chat, committed to Git, sent by email).

1. **Revoke** it **at once** — it is useless from the next request.
2. **Rotate** to get a new key; the app team deploys it. (The app is down between revoke and deploy; that is the price of a leak.)
3. Check **Recent reads** for calls you don't recognise.

**An app is retired.**
**Deactivate** the client. All its keys stop; its history stays.

**An app's copy of employees looks wrong.**
**Request full resync**.

**Is this app still used?**
Look at the keys' **last used** and the app's **Recent reads**.

**Changing ASP CORE's push key.**
Same as a regular rotation on the `asp-core` client; agree the switch-over time with the ASP CORE team so the night push does not use
the old key after it was revoked.

---

## 6. Good to know

**One app, one client, per environment.**
Timesheet gets one client on the Dev hub, one on UAT, one on Prod. Never share a key between two apps: you could not tell who read
what, and you could not switch one off without breaking the other.

**ASP CORE is a client too.**
With only `import:write`. On top of the key, the push can be limited to ASP CORE's server addresses (`Import:AllowedSourceIps`). That
setting applies to the push only, not to reading apps.

**How is the very first key created?**
When nothing exists yet, with the Migrator's `create-client` command (see the README):

```bash
dotnet run --project src/PeopleHub.Migrator -- create-client --name asp-core --scopes import:write --env dev --connection "Host=…;Database=…;Username=…;Password=…"
```

After that, everything is done in the Admin UI.

**Admin keys vs admin sign-in.**
People use the Admin UI with their company sign-in (AD/Entra groups in `Admin:AllowedGroups`). A client with the `admin` scope is only
for scripts that call the admin API; create one rarely and keep its key especially safe.

**Is anything here recorded?**
Yes. Every action — create, change, deactivate, rotate, revoke, resync — is in the **Audit log** with who did it and when.

**What does an app see when its key is wrong?**
`401 Unauthorized` (missing, unknown, expired or revoked key, or a deactivated client). `403 Forbidden` means the key is fine but lacks
the scope for that endpoint. `429 Too Many Requests` means it is over its speed limit.

---

## 7. Quick reference

| Endpoint | In simple words |
|---|---|
| `GET /admin/clients` | List all client apps and their keys |
| `GET /admin/clients/{id}` | One client app |
| `POST /admin/clients` | Create a client app + first key (shown once) |
| `PUT /admin/clients/{id}` | Change description, scopes, active, speed limit |
| `DELETE /admin/clients/{id}` | Deactivate the app and revoke all its keys |
| `POST /admin/clients/{id}/keys` | Rotate: new key, old one valid 7 more days |
| `DELETE /admin/clients/{id}/keys/{keyId}` | Revoke one key now |
| `POST /admin/clients/{id}/resync` | Make the app reload everything |
| `GET /admin/clients/{id}/audit` | What the app has read |

| Situation | Action |
|---|---|
| New app | Create with minimal scopes → hand over the key safely |
| Needs another field | Edit → add scope |
| Planned key change | Rotate → deploy → revoke old |
| Key leaked | Revoke now → rotate → deploy |
| App retired | Deactivate |
| App's data looks wrong | Request full resync |

See also: [client library FAQ](client-library-faq.md) · [Employees and Changes FAQ](employees-and-changes-faq.md) · [webhooks FAQ](webhooks-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) · [manual people FAQ](manual-people-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) · [import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) ·
[onboarding](../onboarding.md).
