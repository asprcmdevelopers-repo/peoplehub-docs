# Mappings, settings and alerts — help and FAQ

Plain-English answers about three Swagger groups an admin uses to **tune** PeopleHub without a redeploy and to **hear about
problems**:

- **Admin: mappings** — how incoming data is read: the **status mappings** (greytHR's status words → PeopleHub's statuses) and the
  **field map** (incoming field names → PeopleHub fields). In the Admin UI: **Status mappings** and **Field map**.

- **Admin: settings** — thresholds and intervals that can be changed while PeopleHub runs. In the Admin UI: **Settings**.
- **Admin: alerts** — the problems PeopleHub has noticed, and how to acknowledge and resolve them. In the Admin UI: **Alerts**.

All endpoints need the `admin` scope; every change is recorded in the **Audit log**.

**Contents**

1. [Status mappings](#1-status-mappings)
2. [Field map](#2-field-map)
3. [Settings](#3-settings)
4. [Alerts](#4-alerts)
5. [Everyday situations](#5-everyday-situations)
6. [Good to know](#6-good-to-know)
7. [Quick reference](#7-quick-reference)

---

## 1. Status mappings

**What problem do they solve?**
greytHR describes an employee's state in its own words: *Confirmed*, *Probation*, *On Notice*, *Relieved*… Apps should not have to
know all of them. PeopleHub turns each word into one of **six fixed statuses**:

| PeopleHub status | Meaning | Active? |
|---|---|---|
| `Active` | Working | Yes |
| `NoticePeriod` | Resigned but still working out the notice | Yes |
| `Resigned` | Left | No |
| `Absconded` | Stopped coming without notice | No |
| `Terminated` | Let go | No |
| `Unknown` | greytHR sent a word PeopleHub has no mapping for | See below |

A **status mapping** is one line: *raw word → PeopleHub status*, for example `Relieved → Resigned`.

**Which mappings exist from the start?**

| greytHR sends | Becomes |
|---|---|
| Active, Confirmed, Probation | `Active` |
| Notice Period, On Notice | `NoticePeriod` |
| Resigned, Relieved | `Resigned` |
| Absconded, Absconding | `Absconded` |
| Terminated, Dismissed | `Terminated` |

They are a starting point; refine them once IT confirms the exact values greytHR sends. Matching ignores upper/lower case, spaces
and punctuation: `On Notice`, `on-notice` and `ONNOTICE` are the same word. A word that is exactly a PeopleHub status (`Resigned`)
works even without a mapping.

**What happens to a word with no mapping?**
The employee is **still imported** (never rejected for this), with status `Unknown`. The run shows a warning `unknown_status`, and
the word appears under **Status mappings → Seen but not mapped**, with how many employees have it.

**Does the status decide whether someone is active?**
Only when greytHR sends **no active flag**:

- If the row has an `active` field (`true`/`false`, `yes`/`no`, `1`/`0`), **that flag decides** — the status is just information.
- If not, the person is inactive when their raw word or PeopleHub status is in `Import:InactiveStatuses` (default: Resigned,
  Absconded, Terminated, Inactive), otherwise active. So an `Unknown` status counts as **active** until it is mapped.

### The status mapping endpoints

| Endpoint | What it does |
|---|---|
| `GET /api/v1/admin/status-mappings` | `mappings` (every line, with who changed it and when) and `unmapped` (raw words currently stored as Unknown, with employee counts) |
| `PUT /api/v1/admin/status-mappings` | Add or change lines: `[ { "rawStatus": "Sabbatical", "canonical": "Active" } ]`. Several at once is fine; an existing word is overwritten. An empty raw word → `400`. |
| `DELETE /api/v1/admin/status-mappings/{rawStatus}` | Remove a line (`204`; unknown word → `404`) |

**When does a change take effect?**
From the **next import**. People already stored as `Unknown` stay so until data is processed again: **replay** the last run
(Import runs → run → *Replay with current mapping*) to apply it straight away.

---

## 2. Field map

**What problem does it solve?**
ASP CORE may call the code `emp_id` and the name `full_name`; PeopleHub calls them `employeeCode` and `name`. The **field map** says
which incoming name means which PeopleHub field, so nobody has to change ASP CORE's export.

**Which PeopleHub fields are there?**
`employeeCode`, `name`, `email`, `department`, `designation`, `managerEmployeeCode`, `managerEmail`, `location`, `shift`,
`dateOfJoining`, `active`, `status`, `leavingDate`, `lastModified`.

**How is each incoming field treated?**

| Incoming field | Becomes |
|---|---|
| Same name as a PeopleHub field (case, spaces and `_` ignored: `Employee_Code` = `employeeCode`) | That field — no mapping needed |
| Listed in the field map | The PeopleHub field it maps to |
| Mapped to `-` | **Dropped** (thrown away) |
| On the sensitive list (salary, phone, PAN, Aadhaar, bank account, date of birth …) | **Always dropped**, whatever the map says |
| Anything else | Kept in **`extra`** (apps with the `employees:read:extra` scope can see it) |

**What are the other mapping settings?**

| Setting | Meaning | Default |
|---|---|---|
| **Excel-only column headers** (`ExcelFieldMap`) | Extra lines used **only for Excel uploads**, on top of the field map — HR's sheet headers (`Emp Code`, `Date of Joining`) | none |
| **Array path** (`ArrayPath`) | Where the list of employees sits in ASP CORE's JSON: `$` = the body is the list; `$.employees` or `$.data.items` when it is wrapped | `$` |
| **Date formats** (`DateFormats`) | Accepted date formats, tried in order | `yyyy-MM-dd`, `dd/MM/yyyy`, `dd-MM-yyyy`, `dd-MMM-yyyy`, `dd MMM yyyy`, `d/M/yyyy` |
| **Strict mode** (`Strict`) | Off: only code and name are required; a bad optional value (wrong date) is dropped with a warning. **On:** email, department, designation, date of joining and status-or-active are also required, a bad value rejects the row, and unknown fields are listed as a warning. Switch on once the mapping is agreed with IT. | off |

### The field map endpoints

| Endpoint | What it does |
|---|---|
| `GET /api/v1/admin/field-map` | The map in use now: field map, Excel-only headers, array path, date formats, strict, and the list of PeopleHub fields |
| `POST /api/v1/admin/field-map/preview` | **Try** a draft against a stored payload; **writes nothing**. Body: `{ "runId": 6, "draft": { "fieldMap": { "emp_id": "employeeCode" } } }`. Leave out `runId` for the latest payload; leave out a draft part to keep the current one. |
| `PUT /api/v1/admin/field-map` | **Save**. Send only the parts you change (`fieldMap`, `excelFieldMap`, `arrayPath`, `dateFormats`, `strict`); a part you send **replaces** the old one completely. Mapping to an unknown PeopleHub field → `400` with the list of valid ones; nothing is saved. |

**What does the preview show?**

- every **incoming field**, what it **becomes** (a field, `(extra)`, `(ignored)`, `(dropped: sensitive)`) and in how many rows,
- how many rows are valid and how many would be skipped,
- the first 100 **problems** (row, field, message),
- a **sample** of 20 mapped rows,
- or an error, e.g. the array path is wrong.

`doj → (extra)` in the preview is the classic sign of a forgotten line.

**When does a saved map take effect?**
From the **next import**. Use **Replay** to apply it to a stored payload now (raw payloads are kept 30 days). The field map is
stored as a setting, so **Settings → Import:FieldMap → Reset** returns to the appsettings version.

More examples (the `emp_id` story, skipped rows): [import runs FAQ](import-runs-faq.md).

---

## 3. Settings

**What are runtime settings?**
Most of PeopleHub's behaviour is configured in `appsettings.json`, which needs a redeploy to change. A chosen set of values can also
be changed **while it runs**: the new value is stored in the database and **overrides** the file until it is **reset**.

**Which settings can be changed?**

| Group | Setting | Meaning | Default |
|---|---|---|---|
| Import | `Import:ExpectedIntervalHours` | How often ASP CORE should push | 24 |
| Import | `Import:AlertGraceHours` | Extra hours before the missed-run alert | 2 |
| Import | `Import:ManagerMatching` | How managers are found: `CodeThenEmail`, `CodeOnly`, `EmailOnly` | CodeThenEmail |
| Safety brake | `Import:SafetyBrake:Enabled` | Reject suspicious runs instead of applying them | true |
| Safety brake | `Import:SafetyBrake:MaxDeactivatePercent` | Reject a run that would deactivate more than this % of active employees | 5 |
| Safety brake | `Import:SafetyBrake:MinPayloadPercentOfPrevious` | Reject a run smaller than this % of the previous run | 80 |
| Mapping | `Import:Strict`, `Import:ArrayPath`, `Import:DateFormats`, `Import:FieldMap`, `Import:ExcelFieldMap` | The field map settings ([section 2](#2-field-map)) | |
| Mapping | `Import:InactiveStatuses` | Statuses that mean inactive when no active flag is sent | Resigned, Absconded, Terminated, Inactive |
| Retention | `Import:RawRetentionDays` | Days raw payloads are kept for replay | 30 |
| Alerts | `Alerts:ErrorRowThreshold` | Alert when a run skips more than this many rows | 10 |
| Alerts | `Alerts:RehireReviewDays` | Alert when a rehire suggestion stays open longer than this | 7 |
| Webhooks | `Webhooks:MaxAttempts` | Delivery attempts before a dead letter (1–50) | 8 |
| Webhooks | `Webhooks:BackoffSeconds` | First retry delay, doubling each time (0–3600) | 30 |

**What can never be changed here?**
Security settings stay in the file on purpose: the push IP allow-list, the sensitive-field list, keys, secrets, sign-in groups,
alert email and Teams addresses.

### The settings endpoints

| Endpoint | What it does |
|---|---|
| `GET /api/v1/admin/settings` | Every setting: key, group, description, **value in use**, whether it is **overridden**, and by whom and when |
| `PUT /api/v1/admin/settings` | Override one or more: `{ "Import:SafetyBrake:MaxDeactivatePercent": 8 }`. **All or nothing:** if one value is invalid (wrong type, out of range, unknown key), nothing is saved and `400` lists the problems. |
| `DELETE /api/v1/admin/settings/{key}` | **Reset**: remove the override, the appsettings value applies again (`204`; not an editable key → `404`) |

**When does a change take effect?**
At once on the server that saved it; other PeopleHub servers pick it up within a minute.

**Overridden vs default.**
The Settings page shows the source of each value. *Overridden* means someone changed it here — it wins over appsettings, even after
a redeploy with a new appsettings value, until **Reset**.

---

## 4. Alerts

**What is an alert?**
A message from PeopleHub that something needs a person's attention. Each alert is stored on the **Alerts** page and sent to the
configured **email group** and/or **Teams channel** (`Alerts:Email`, `Alerts:Teams` in appsettings). The message names the
environment (`Alerts:EnvironmentName`), so a UAT alert is not mistaken for Prod.

**Which alerts are there?**

| Alert | Severity | Raised when | What to do |
|---|---|---|---|
| **No successful employee import** (`missed_run`) | Critical | No import has succeeded for `ExpectedIntervalHours` + `AlertGraceHours` (24 + 2 h). Checked every 15 minutes; closes by itself once an import succeeds again. | Internal runbook: missed run |
| **Import run N rejected by the safety brake** (`safety_brake`) | Critical | A run would deactivate too many people or was much smaller than the last | Internal runbook: rejected run |
| **Import run N had X invalid rows** (`import_errors`) | Warning | A run skipped more than `Alerts:ErrorRowThreshold` (10) rows | Open the run's errors; fix the data in greytHR or the field map |
| **Webhook deliveries to <app> are failing** (`webhook_dead_letter`) | Warning | A delivery failed every attempt (about an hour) | [Webhooks FAQ](webhooks-faq.md); fix the app, then replay dead letters |
| **Rehire candidates waiting for review** (`rehire_review`) | Warning | A rehire suggestion has been open more than `Alerts:RehireReviewDays` (7) | [Rehire review](admin-employees-and-rehire-faq.md#4-rehire-review) |

**Will I get the same alert again and again?**
No. While an alert is open, the same problem does not raise a new one:

- missed run — one alert per gap (per last successful run),
- safety brake and invalid rows — one per run,
- webhooks — one per subscription while it is open; after it is resolved, the next dead letter raises a new one,
- rehire review — one at a time.

**Acknowledge or resolve?**

| | Acknowledge | Resolve |
|---|---|---|
| Means | "I have seen it and I am working on it" | "It is over" |
| Alert stays | **Open** (still counted on the dashboard), marked with your name | **Closed**; it leaves the open list |
| Second time | Refused (`404`) — it is acknowledged once | Refused (`404`) — it is already resolved |

Resolving an alert that nobody acknowledged also records you as having acknowledged it. Neither action sends a message.

**Does an alert close by itself?**
Two of them do, at the next check (every 15 minutes):

- **No successful employee import** — once an import has succeeded again;
- **Rehire candidates waiting for review** — once no suggestion is older than the limit.

All others stay open until an admin **resolves** them — including a safety-brake alert after the run was approved or superseded.

### The alert endpoints

| Endpoint | What it does |
|---|---|
| `GET /api/v1/admin/alerts` | Open alerts (newest first, 100 by default, `limit` up to 1000). `open=false` includes resolved ones (open first). Each has type, severity, title, message, details (run id, subscription …), raised, acknowledged by/at, resolved at. |
| `POST /api/v1/admin/alerts/{id}/acknowledge` | Acknowledge (`204`) |
| `POST /api/v1/admin/alerts/{id}/resolve` | Resolve (`204`) |

Resolved alerts are kept for 365 days (`Retention:ResolvedAlertsDays`).

---

## 5. Everyday situations

**greytHR started sending a new status, e.g. "Sabbatical".**

1. **Status mappings → Seen but not mapped** shows it with the number of employees.
2. Choose the PeopleHub status → **Map**.
3. Replay the last run so those people get the right status now.

**ASP CORE renamed a field (`doj` → `joining_date`).**

1. **Field map** → add `joining_date → dateOfJoining` → **Preview**: check it now shows `dateOfJoining`, not `(extra)`.
2. **Save**, then replay the affected run.

**ASP CORE now wraps the list: `{ "data": { "employees": [ … ] } }`.**
**Field map → Array path** `$.data.employees` → Preview → Save → replay.

**A genuine large exit next month (a process moves out).**
Either approve the rejected run when it happens, or raise `Import:SafetyBrake:MaxDeactivatePercent` beforehand in **Settings** and
**Reset** it afterwards.

**ASP CORE will push twice a day from now on.**
**Settings → Import:ExpectedIntervalHours** = 12, so a missed push is noticed sooner.

**An alert came in. What do I do?**
**Acknowledge** it so colleagues know someone is on it, follow the "what to do" column above, then **Resolve** it.

**Nobody receives alert emails.**
Check `Alerts:Email` (recipients and SMTP server) or `Alerts:Teams` (webhook URL) in appsettings with IT. Alerts are still listed on
the **Alerts** page and counted on the dashboard even when no email goes out.

---

## 6. Good to know

**Is it recorded?**
Yes, in the **Audit log**:

| Action | Audit entry | Target |
|---|---|---|
| Save status mappings | `status_mappings.saved` (the lines) | `status_mappings` |
| Remove a status mapping | `status_mappings.deleted` | `status_mappings` |
| Save settings or the field map | `settings.saved` (the values) | `settings` |
| Reset a setting | `settings.reset` | the setting key, e.g. `Import:Strict` |
| Acknowledge / resolve an alert | `alert.acknowledged` / `alert.resolved` | `alert:N` |

**Can a mapping change break old data?**
No. A mapping only changes how **future** imports (and replays) read data. Nothing already stored is rewritten until a run is
replayed — and a replay goes through the safety brake like any import.

**Which run used which mapping?**
Each run keeps a copy of the settings it ran with, so a later change does not make old runs confusing.

**Preview first, always.**
Preview writes nothing and shows exactly what the next import would do with the stored payload.

---

## 7. Quick reference

| Endpoint | In simple words |
|---|---|
| `GET /admin/status-mappings` | Status lines + words seen but not mapped |
| `PUT /admin/status-mappings` | Add or change status lines |
| `DELETE /admin/status-mappings/{raw}` | Remove a status line |
| `GET /admin/field-map` | The field map in use |
| `POST /admin/field-map/preview` | Try a draft map on a stored payload (writes nothing) |
| `PUT /admin/field-map` | Save the field map |
| `GET /admin/settings` | All runtime settings, value in use, overridden or not |
| `PUT /admin/settings` | Change settings (all or nothing) |
| `DELETE /admin/settings/{key}` | Back to the appsettings value |
| `GET /admin/alerts` | Open (or all) alerts |
| `POST /admin/alerts/{id}/acknowledge` | "I'm on it" |
| `POST /admin/alerts/{id}/resolve` | "It's over" |

| Situation | Action |
|---|---|
| New status word | Status mappings → Seen but not mapped → Map → replay |
| Field renamed | Field map → add line → Preview → Save → replay |
| Change a threshold | Settings → value → Save (Reset to undo) |
| Alert received | Acknowledge → fix → Resolve |

See also: [client library FAQ](client-library-faq.md) · [import FAQ](import-faq.md) (the push and the Excel upload) · [import runs FAQ](import-runs-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [webhooks FAQ](webhooks-faq.md) ·
[admin employees and rehire FAQ](admin-employees-and-rehire-faq.md).
