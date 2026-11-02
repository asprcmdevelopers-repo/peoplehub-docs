# Import — help and FAQ

Plain-English answers about the **Import** group in Swagger: the two endpoints that **bring employee data into PeopleHub**.

- `POST /api/v1/import/employees` — ASP CORE's **push** of the greytHR employee list (normally every night).
- `POST /api/v1/import/excel` — an admin's **upload** of HR's Excel sheet, the fallback when the push is broken. In the Admin UI:
  **Imports → Excel upload**.

What happens *after* a run (reading a run's page, approve, replay, raw payload) is in the [import runs FAQ](import-runs-faq.md).

**Contents**

1. [The basics](#1-the-basics)
2. [Every push is the whole company](#2-every-push-is-the-whole-company)
3. [The push: `POST /import/employees`](#3-the-push-post-importemployees)
4. [The Excel upload: `POST /import/excel`](#4-the-excel-upload-post-importexcel)
5. [The answers](#5-the-answers)
6. [Everyday situations](#6-everyday-situations)
7. [Good to know](#7-good-to-know)
8. [Quick reference](#8-quick-reference)

---

## 1. The basics

**Where does PeopleHub's employee data come from?**
From greytHR, but PeopleHub never calls greytHR itself. **ASP CORE** already has the greytHR data, so ASP CORE **sends** (pushes) the
whole list to PeopleHub on a schedule. If that is broken, an admin can upload HR's Excel sheet instead.

**How are the two endpoints different?**

| | `POST /import/employees` | `POST /import/excel` |
|---|---|---|
| **Who calls it** | **ASP CORE**, on its schedule | **An admin**, when needed |
| **What is sent** | JSON: the employee list | An `.xlsx` workbook |
| **Key needed** | `import:write` — only the `asp-core` client has it | `admin` |
| **Extra protection** | Can be limited to ASP CORE's server addresses (`Import:AllowedSourceIps`) | — |
| **Run source / trigger** | `Core` / `Push` | `Excel` / `Upload` |
| **In the Admin UI** | — (machine to machine) | **Imports → Excel upload** |

**After that, are they treated the same?**
Yes. Once the file is read, both go through **exactly the same steps**:

1. The payload is **stored as received** (kept 30 days) and an **import run** is created.
2. Each row is read with the **field map** and **status mappings** (see the
   [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md)), then checked.

3. The result is compared with what PeopleHub already has: who is new, who changed, who is missing.
4. The **safety brake** looks at the result. If it looks like a broken list, **nothing** is applied.
5. Otherwise everything is saved in one go: employee records, **history**, and **events** for the apps (`employee.joined`,
   `updated`, `manager_changed`, `left`, `reactivated`).

6. New codes are checked against leavers for **rehire suggestions**.

Every push and every upload is a run you can open later under **Import runs**.

---

## 2. Every push is the whole company

**Does ASP CORE send only what changed?**
No — it sends **everyone, every time** (a *full snapshot*), including people who have left (with `active: false`). PeopleHub works
out the differences itself:

| In the list… | PeopleHub does | Apps receive |
|---|---|---|
| A code it has never seen | Creates the employee | `employee.joined` |
| A known code with different details | Updates the record (history shows each field) | `employee.updated` / `employee.manager_changed` |
| A known code, now inactive (`active: false` or a leaving status) | Marks them as left (reason `feed_inactive`) | `employee.left` |
| A former leaver, active again | Reactivates them | `employee.reactivated` |
| A known code with nothing changed | Nothing | Nothing |
| **Not in the list at all** | **Treats them as left** (reason `missing_from_feed`) | `employee.left` |

**Why is "missing means left" a rule?**
Because a full list is the simplest thing for ASP CORE to send, and it cannot miss a leaver: anyone greytHR no longer has is gone
from PeopleHub too.

**Who is never treated as left for being missing?**

- **Manual people** (`M-` codes) — imports never touch them (see the [manual people FAQ](manual-people-faq.md)).
- Anyone whose row was **skipped** because of an error — a typo in someone's email must not make them a leaver. They stay as they
  were until a later run reads their row correctly.

**What stops a broken list from "firing" half the company?**
The **safety brake**. A run is stopped when:

| Check | Default | Setting |
|---|---|---|
| It would deactivate more than this share of active employees | **5 %** | `Import:SafetyBrake:MaxDeactivatePercent` |
| The list is smaller than this share of the previous run's list | **80 %** | `Import:SafetyBrake:MinPayloadPercentOfPrevious` |

Then **nothing** is applied, the run becomes **Rejected**, the answer is `422`, and a Critical alert goes out. If the change was real
(a planned mass exit), an admin **approves** the run — see the [import runs FAQ](import-runs-faq.md#4-safety-brake-approve-and-replay).

---

## 3. The push: `POST /import/employees`

**What does ASP CORE send?**
A JSON array, one object per employee:

```json
[
  { "employeeCode": "ASP0001", "name": "Priya Raman", "email": "priya.raman@example.com",
    "department": "AR Follow-up", "designation": "Team Lead", "managerEmployeeCode": "ASP0100",
    "location": "Chennai", "shift": "General", "dateOfJoining": "2021-04-12", "active": true, "status": "Confirmed",
    "lastModified": "2026-10-31T18:20:00+05:30" },
  { "employeeCode": "ASP0045", "name": "Arun Kumar", "active": false, "status": "Resigned", "leavingDate": "2026-10-30" }
]
```

Only `employeeCode` and `name` are required (more in **strict mode**). The full field list and what ASP CORE should do with each
answer are in the [IT contract](../it-contract.md), the document given to the ASP CORE team.

**What if ASP CORE wraps the list, like `{ "employees": [ … ] }`?**
Fine — set **Field map → Array path** to `$.employees`. The default `$` means a bare array.

**What if ASP CORE uses other field names (`emp_id`, `doj` …)?**
Fine — add them to the **Field map**. ASP CORE does not need to change anything.

**What if ASP CORE sends salary, phone or bank details?**
They are **dropped** before anything is stored in an employee record, and a warning is logged in the run. Ask them to leave such
fields out anyway.

**How big may the list be?**
Up to **20 MB** (`Import:MaxPayloadBytes`) — far more than the company needs.

**Can anyone else push?**
Only a key with `import:write`. Give that scope to **ASP CORE's client only**. If `Import:AllowedSourceIps` lists addresses (single IPs
or ranges like `10.20.0.0/24`), calls from anywhere else get `403` even with the right key. Empty means any address.

---

## 4. The Excel upload: `POST /import/excel`

**When is it used?**
When the push cannot be fixed in time (ASP CORE down, export broken) and the data must be up to date today. HR gives the employee sheet;
an admin uploads it.

**How do I upload?**

- **Admin UI:** **Imports → Excel upload** → choose the `.xlsx` → *Sheet* empty for the first sheet → **Import**.
- **API:** `multipart/form-data` with the file in a field named `file`; optional `?sheet=Employees` to pick a worksheet.

**Which files are accepted?**
Only `.xlsx`. Save `.xls` or `.csv` files as `.xlsx` first. The first row must be the column headers; headers are matched like JSON
field names, and differing ones go under **Field map → Excel-only column headers**.

**Is the sheet also a full list?**
**Yes — this is the most important point.** Everyone with a greytHR code who is not in the sheet is treated as having left. Always
upload the **complete** employee sheet, never a list of a few changed people. (If you forget, the safety brake will normally reject
the run.)

**Why did the first upload say "Updated" for almost everyone?**
Each record changes from `source = Core` to `source = Excel`, which counts as a change. The next push from ASP CORE switches them back.
Details in the [import runs FAQ](import-runs-faq.md#6-excel-uploads).

**Is the original file kept?**
Yes, for 30 days, with a **Download uploaded workbook** button on the run's page.

**Upload errors:**

| Answer | Reason |
|---|---|
| `400` *No file uploaded* | No file in the field `file` |
| `400` *Unsupported file type* | Not an `.xlsx` |
| `400` *Payload could not be processed* | The workbook cannot be read, or the sheet name does not exist — the run is recorded as **Failed** with the reason |
| `413` *File too large* | Over 20 MB |

---

## 5. The answers

Both endpoints answer the same way:

| Answer | Run status | Meaning | What to do |
|---|---|---|---|
| `200` | **Succeeded** | Applied | Nothing; skipped rows (if any) are listed in `errors` |
| `422` | **Rejected** | The safety brake stopped it; nothing applied; Critical alert | ASP CORE: **do not retry**. Admin: check the run, approve only if genuine |
| `400` | **Failed** | Could not be read (not JSON, no array at the array path, unreadable workbook) | Fix the export or the array path, send again |
| `409` | **Failed** | Another import was still running (PeopleHub waits up to 30 s, `Import:LockTimeoutSeconds`) | Send again a few minutes later |
| `401` | — (no run) | Missing, wrong, expired or revoked key | Check the key |
| `403` | — (no run) | The key lacks the scope, or the address is not allowed | Check the scope / `AllowedSourceIps` |
| `413` | — (no run) | Too large | — |

**What is in a `200` answer?**

```json
{ "runId": 57, "status": "Succeeded", "source": "Core",
  "received": 412, "created": 3, "updated": 11, "deactivated": 2, "reactivated": 0, "skipped": 1,
  "errors":   [ { "row": 20, "employeeCode": "ASP0033", "field": "email", "code": "invalid_email", "message": "…" } ],
  "warnings": [ { "row": 31, "employeeCode": "ASP0020", "field": "status", "code": "unknown_status", "message": "…" } ],
  "brake": null }
```

A `422` has the same counts (what *would* have happened) plus `brake`: the reason (`deactivate_percent_exceeded` or
`payload_too_small`), the message, the percentage and the limit. A `400` or `409` is a problem answer with the `runId` when a run was
created. ASP CORE should log the `runId` — it is the number to quote when asking about a push.

**One bad row — does the whole push fail?**
No. That row is **skipped** and listed in `errors`; every other row is applied. Common reasons:

| Code | Meaning |
|---|---|
| `missing_required` | No `employeeCode` or `name` (or, in strict mode, another required field) |
| `duplicate_code` | The same code appears twice; the first row was used |
| `duplicate_email` | Two active people in the list have the same email |
| `invalid_email`, `invalid_boolean`, `invalid_type` | A value in the wrong format |
| `reserved_code_prefix` | The code starts with `M-`, which is reserved for manual people |
| `invalid_row` | The entry is not an object |

`unknown_status` is a **warning**, not an error: the person is imported with status `Unknown`. When more than 10 rows are skipped
(`Alerts:ErrorRowThreshold`), an *Import errors* alert is raised.

---

## 6. Everyday situations

**The night push did not arrive.**
After 26 hours (24 + 2 grace) the *No successful employee import* alert fires. Follow the
internal runbook: ask ASP CORE to check their job, or upload the Excel sheet.
The alert closes by itself once an import succeeds again.

**The push came back `422`.**
Open the run (**Import runs**) and look at the brake reason. A half-size list → ask ASP CORE to fix the export and push again. A real
mass exit → **Approve**.

**The push came back `400`.**
Open the run: the reason and the stored payload are there. Usually ASP CORE changed the shape (wrapped the list, renamed the array) →
fix the **array path** or field map, then **Replay** the run instead of waiting for the next night.

**A new joiner is missing after the push.**
Check the run's **Errors** for their code — the row was probably skipped. Ask HR to fix it in greytHR; the next push brings them.

**Someone who left is still active.**
If greytHR still sends them as active, PeopleHub shows what greytHR says. Fix it in greytHR (status or leaving date).

**ASP CORE wants to push more often (every few hours).**
No change needed on their side. Lower `Import:ExpectedIntervalHours` under **Settings** so the missed-run alert matches the new rhythm.

**HR wants to "just upload the 5 new joiners".**
Not with this endpoint — a sheet of 5 people means everyone else has left (the safety brake would reject it). Upload the full sheet, or
wait for the push.

---

## 7. Good to know

**Is it safe to send the same list twice?**
Yes. The second run finds nothing to change, so the counts are 0 and no events are sent. That is why ASP CORE may simply retry after a
timeout.

**Can two imports run at the same time?**
No. Only one run applies changes at a time; a second one waits up to 30 seconds and then answers `409`.

**Is a push recorded in the audit log?**
A push is recorded as an **import run** (who called, from which address, when, the payload). Excel uploads, approvals and replays
are also in the **Audit log** with the admin's name.

**What happens to the apps?**
They receive the events from a successful run within seconds (webhooks) or at their next poll (`/changes`). A rejected or failed run
sends nothing.

**Do timestamps without a time zone work?**
Yes — `lastModified` without an offset is read as India time (`Import:SourceTimeZone`, default `Asia/Kolkata`).

---

## 8. Quick reference

| Endpoint | In simple words |
|---|---|
| `POST /import/employees` | ASP CORE sends the full employee list (key with `import:write`) |
| `POST /import/excel` | An admin uploads HR's full `.xlsx` sheet (key with `admin`, or **Imports → Excel upload**) |

| Answer | Meaning |
|---|---|
| `200` | Applied (skipped rows in `errors`) |
| `422` | Safety brake — nothing applied; do not retry |
| `400` | Could not be read — fix and resend |
| `409` | Another import running — resend later |
| `401` / `403` / `413` | Key, scope or address problem / too large |

| Remember | |
|---|---|
| Every push and upload is the **whole company** | Missing = left (except manual people and skipped rows) |
| The safety brake | > 5 % leavers or < 80 % of the last list → rejected |
| Same list twice | Harmless |

See also: [client library FAQ](client-library-faq.md) · [import runs FAQ](import-runs-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) ·
[Employees and Changes FAQ](employees-and-changes-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) ·
[manual people FAQ](manual-people-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [IT contract](../it-contract.md).
