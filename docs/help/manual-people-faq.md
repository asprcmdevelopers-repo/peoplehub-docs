# Manual people — help and FAQ

Plain-English answers about **Admin: manual people**: people who work for the company but are **not in greytHR** — contractors,
interns, vendor staff, consultants — and are added to PeopleHub by hand. In the Admin UI this is the **Manual people** page.

**Contents**

1. [The basics](#1-the-basics)
2. [The six endpoints](#2-the-six-endpoints)
3. [When the person joins greytHR](#3-when-the-person-joins-greythr)
4. [Everyday situations](#4-everyday-situations)
5. [Good to know](#5-good-to-know)
6. [Quick reference](#6-quick-reference)

---

## 1. The basics

**Why add people by hand?**
Everyone who uses a company app needs to be in PeopleHub: an intern logs into Timesheet, a vendor's staff member takes a course in
the training portal. If they are not in greytHR, ASP CORE's push never brings them, so an admin adds them under **Manual people**.

**How are they different from greytHR people?**

| | greytHR people | Manual people |
|---|---|---|
| Code | From greytHR (`E1050`, `ASP0001`…) | Given by PeopleHub: `M-0001`, `M-0002`… (prefix `ManualPeople:IdPrefix`) |
| Source | `Core` or `Excel` | `Manual` |
| Changed by | The push (or HR's Excel upload) | An admin, on the Manual people page |
| Deactivated when | Missing from a push, or a leaving status | Only when an admin deactivates them |
| Counted by the safety brake | Yes | No |

**Do the imports ever touch them?**
No. They are not updated, not deactivated when missing from a push or Excel sheet, and not counted by the safety brake. A pushed row
with an `M-` code is skipped ("prefix reserved for manually added people"), so greytHR can never overwrite one by accident.

**What do apps see?**
Normal employees: the same endpoints, the same fields, the same events (`employee.joined`, `updated`, `manager_changed`, `left`,
`reactivated`). Their `source` is `Manual`, so an app can treat them differently if it wants to. Events are written the moment the
admin saves — no push, no import run.

**Who can do this?**
An admin (a person signed in to the Admin UI, or a script with an `admin` key). All endpoints need the `admin` scope.

---

## 2. The six endpoints

### `GET /api/v1/admin/manual-people` — list

Manual people only, 100 per page. Filters: `search` (code, name, email, department), `active`, `page`. In the Admin UI: **Manual
people** (tick *Show inactive* to see leavers too).

### `POST /api/v1/admin/manual-people` — add a person

```json
{ "name": "Ravi Kumar", "email": "ravi.kumar@example.com", "department": "AR Follow-up",
  "designation": "Intern", "managerEmployeeCode": "E1050", "location": "Chennai",
  "shift": "General", "dateOfJoining": "2026-11-03", "status": "Active" }
```

| Field | Notes |
|---|---|
| `name` | **Required** |
| `email` | Optional, but needed for most app logins. Stored in lower case. |
| `managerEmployeeCode` or `managerEmail` | The manager, by code or (if no code) by email; found the same way as for greytHR people |
| `dateOfJoining`, `leavingDate` | Start and (planned) end date; the end date cannot be before the start |
| `status` | `Active` (default) or `NoticePeriod` = active; `Resigned`, `Absconded`, `Terminated`, `Unknown` = inactive |
| `department`, `designation`, `location`, `shift`, `extra` | Optional |

PeopleHub gives the next code (e.g. `M-0007`) and answers `201` with the new record. Apps receive `employee.joined`.
In the Admin UI: **Manual people → Add person**.

| Error | When |
|---|---|
| `400` | No name; an invalid email or manager email; end date before start date |
| `409` | An **active** employee already uses this email — usually the person is already in greytHR (see [section 3](#3-when-the-person-joins-greythr)) |

### `PUT /api/v1/admin/manual-people/{code}` — edit

**The whole record is replaced** with what you send: a field you leave out becomes empty (only `extra` is kept when not sent). The
Admin UI form is filled in with the current values, so this matters only for scripts: read the record first, change it, send it all.
Apps receive `employee.updated` (or `manager_changed`, `left`, `reactivated` if that is what changed).

| Error | When |
|---|---|
| `400` | Same checks as adding |
| `404` | Unknown code, or a greytHR person (they are changed in greytHR) |
| `409` | Already converted to a greytHR code (edit that record instead), or the email is used by another active employee |

### `POST /api/v1/admin/manual-people/{code}/deactivate` — the person left

```json
{ "leavingDate": "2026-12-31" }
```

The leaving date is optional (today if left out). Status becomes `Resigned`, reason "manual". Apps receive `employee.left` — Timesheet,
for example, blocks the login. Nothing is deleted. In the Admin UI: the person's page → **Status** → **Deactivate**.

### `POST /api/v1/admin/manual-people/{code}/reactivate` — the person is back

The **same `M-` code** is active again; the leaving date is cleared. Apps receive `employee.reactivated`. Refused (`409`) if another
active employee now uses their email.

### `POST /api/v1/admin/manual-people/{code}/convert` — the person joined greytHR

```json
{ "greytHrCode": "E2051" }
```

The `M-` code becomes an **old code** of the greytHR code. Details in the next section.

Deactivate, reactivate and convert answer `404` for an unknown code, a greytHR person or a person already converted.

---

## 3. When the person joins greytHR

**Example:** intern Ravi (`M-0007`) is hired permanently and gets `E2051` in greytHR.

1. **Wait until `E2051` is in PeopleHub** — it arrives with the next push. Convert needs both codes to exist.
2. **Link the two codes** (one of the three ways below).
3. From then on:
   - `M-0007` is an old code of `E2051` and no longer active,
   - apps receive one **`employee.alias_linked`** event and merge Ravi's intern history into `E2051`,
   - apps asking for `M-0007` get `E2051`,
   - Ravi is a normal greytHR employee, kept up to date by the push.

**Is there more than one way to do it?**
Yes — three endpoints give **exactly the same result** (same rules, same event, same audit entry `alias.linked`, link kind
*ManualConversion*):

| Endpoint | How you call it | When to use it |
|---|---|---|
| `POST /admin/manual-people/M-0007/convert` | `{ "greytHrCode": "E2051" }` | The normal way, from **Manual people** |
| `POST /admin/rehire-candidates/{id}/link` | Only the suggestion number | PeopleHub already **suggested** the match on **Rehire review** (usual when the email is the same) |
| `POST /admin/employees/E2051/aliases` | `{ "aliasCode": "M-0007", "note": "…" }` | The general "link an old code" endpoint, from **Employees** |

The small differences:

- **Convert** checks that the code really is a manual person (a greytHR code by mistake → `400`); its note is always "Converted from
  a manually added person".

- **Aliases** lets you write your own note.
- **Rehire link** asks for the extra *I checked both records* tick when the match was only by name and department.

**Which one should I use?**
If **Rehire review** already shows the suggestion, click **Same person** there — the suggestion is closed at the same time.
Otherwise use **Convert**. Whichever you use, an open suggestion for the same pair is closed too, so it never has to be done twice.

**When is it refused?**

| Answer | Reason |
|---|---|
| `400` | The code is not a manual person (convert only); the greytHR code is itself a manual person |
| `404` | The greytHR code does not exist yet (wait for the push) |
| `409` | One of the codes is already linked to someone else |

**I converted to the wrong code.**
**Employees → the greytHR person → Old codes → Unlink**, with a reason. `M-0007` becomes its own record again, **active as before**,
and apps receive `employee.alias_unlinked`. Then convert to the right code. See the
[admin employees and rehire FAQ](admin-employees-and-rehire-faq.md#3-old-codes-aliases-explained).

---

## 4. Everyday situations

**A new intern starts on Monday.**
**Manual people → Add person**: name, email, department, manager, start date, (planned) end date. Apps can see them within minutes
(webhook apps within seconds).

**The internship ended.**
Person → **Deactivate** with the end date. Their login is blocked by the apps; their record and history stay.

**The contract was extended / the person moved department.**
Person → change the fields → **Save**. For a new end date, change *End date*; the person stays active.

**The person came back after a break.**
Person → **Reactivate**. Same `M-` code, same history.

**Adding someone fails with "An active employee already uses this email".**
They are already in PeopleHub as an active employee — usually in greytHR. Search for the email on **Employees**. Do not add them twice.

**The intern joined greytHR.**
Wait for the push to bring the greytHR code, then follow [section 3](#3-when-the-person-joins-greythr).

**Can I delete a manual person added by mistake?**
No — nothing is deleted, so history and app data stay explainable. **Deactivate** them instead.

---

## 5. Good to know

**Is it recorded?**
Yes, in the **Audit log** (target `employee:M-xxxx`, or the greytHR code for a conversion): `manual.created`, `manual.updated`,
`manual.deactivated`, `manual.reactivated`, and `alias.linked` for a conversion. Every field change is also in the person's
**History**, shown as `admin:<name>`. See the [dashboard and audit FAQ](dashboard-and-audit-faq.md).

**Can two active people share an email?**
No. Adding, editing or reactivating a manual person is refused when another **active** employee (manual or greytHR) has the same
email. Inactive people may share it — a former intern's email can come back for their greytHR record.

**Can a manual person be a manager?**
Yes. Anyone can give their `M-` code as the manager, and apps see the reporting line as usual.

**Are the numbers ever reused?**
No. Each new person gets the next free number; numbers of deactivated or converted people are not given out again.

---

## 6. Quick reference

| Endpoint | In simple words |
|---|---|
| `GET /admin/manual-people` | List people added by hand |
| `POST /admin/manual-people` | Add a contractor, intern, vendor staff or consultant (gets an `M-` code) |
| `PUT /admin/manual-people/{code}` | Edit (send the whole record) |
| `POST /admin/manual-people/{code}/deactivate` | They left |
| `POST /admin/manual-people/{code}/reactivate` | They are back |
| `POST /admin/manual-people/{code}/convert` | They joined greytHR: link the `M-` code to the greytHR code |

| Situation | Action |
|---|---|
| New intern or contractor | Manual people → Add person |
| Contract ended | Person → Deactivate |
| Came back | Person → Reactivate |
| Joined greytHR | Wait for the push → Rehire review *Same person*, or Convert |
| Converted to the wrong code | Employees → greytHR person → Old codes → Unlink |
| "Email already used" | Search Employees for the email; do not add twice |

See also: [client library FAQ](client-library-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) · [Employees and Changes FAQ](employees-and-changes-faq.md#8-manual-people-interns-contractors-and-the-push) ·
[import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md).
