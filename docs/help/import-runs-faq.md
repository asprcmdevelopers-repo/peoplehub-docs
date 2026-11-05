# Import runs — help and FAQ

Plain-English answers about **Admin: import runs**: what a run is, what the page and the API show, what to do when something
looks wrong. Written from real questions asked while testing PeopleHub; the examples use the local demo data
(runs #1–#6, employees `ASP0001…`). How a run starts — ASP CORE's push and the Excel upload — is in the [import FAQ](import-faq.md).

**Contents**

1. [The basics](#1-the-basics)
2. [Reading the import run page](#2-reading-the-import-run-page)
3. [The five API endpoints](#3-the-five-api-endpoints)
4. [Safety brake, approve and replay](#4-safety-brake-approve-and-replay)
5. [Skipped rows, field names and extra data](#5-skipped-rows-field-names-and-extra-data)
6. [Excel uploads](#6-excel-uploads)
7. [Raw payload: what is kept and for how long](#7-raw-payload-what-is-kept-and-for-how-long)
8. [Finding out exactly what a run changed](#8-finding-out-exactly-what-a-run-changed)
9. [When the data turns out to be wrong](#9-when-the-data-turns-out-to-be-wrong)
10. [Rehire links and undoing a wrong link](#10-rehire-links-and-undoing-a-wrong-link)
11. [Related: PeopleHub's fixed fields](#11-related-peoplehubs-fixed-fields)
12. [Quick reference](#12-quick-reference)

---

## 1. The basics

**What is an import run?**
Every time employee data arrives — ASP CORE's push, an Excel upload, or an admin's replay — PeopleHub records it as one numbered
*run*. The run keeps the result, the counts, every problem found, and (for 30 days) the data exactly as it arrived.

**Is every run the whole company?**
Yes. Every push and every Excel sheet is treated as the **complete** list of employees. Anyone missing from it is treated as
having left. That is how leavers are detected, and it is why a half-empty list is dangerous (see the safety brake).

**What do the statuses mean?**

| Status | Meaning |
|---|---|
| Succeeded | Applied. |
| Rejected | Stopped by the safety brake. **Nothing was applied.** An admin can look at it and approve it if the change is genuine. |
| Failed | The data could not be read at all (not JSON, no employee list found), or, in strict mode, a required column was missing. Nothing was applied. A failed run cannot be approved: fix the export and send again, or fix the field map and replay it. |
| Superseded | A rejected run that a newer successful run has replaced. It can no longer be approved. |
| Processing | Running right now. |

**What do the triggers mean?**
`Push` = sent by ASP CORE · `Upload` = Excel sheet uploaded in the Admin UI · `Replay` = an admin re-processed a stored run
("Replay of #1" shows which).

**What do the counts mean?**

| Count | Meaning |
|---|---|
| Received | Rows in the payload. |
| Created | New employees. |
| Updated | Existing employees with at least one changed field (and still active). |
| Deactivated | Active employees who became inactive (marked as left, or missing from the list). |
| Reactivated | Inactive employees who became active again. |
| Skipped | Rows that had an error and were not imported. |

On a **Rejected** run the counts show what *would* have happened; nothing was actually applied.

**Who can see and use import runs?**
Admins: in the Admin UI (**Imports → Import runs**), or through the admin API with an `admin` API key (also from Swagger:
**Authorize**, paste the key, **Try it out**).

---

## 2. Reading the import run page

**The list (Import runs) only shows one row per run. Where are the details?**
Click the run number (e.g. **#6**). The details, buttons and lists are on the run's own page.

**What is on a run's page, top to bottom?**

1. **Counts** (cards).
2. **Facts**: source and trigger, start and finish time, caller (API key name or admin) and IP, failure reason, safety-brake numbers,
   who approved it.

3. **Buttons**: *Approve and apply* (only for rejected runs) · *Replay with current mapping* · *Try a mapping on this payload*
   (opens the field-map preview) · *Download uploaded workbook* (only for Excel runs, while the file is kept).

4. **Errors** — rows that were skipped, and why.
5. **Warnings** — imported, but worth knowing (unknown status, dropped sensitive fields, bad optional values).
6. **Changes in this run** — every employee the run touched (see section 8). Folded when longer than 15 rows; click the title.
7. **Raw payload** — exactly what arrived. Folded; click to open.

Long lists scroll inside their own box with the headings fixed, so the page stays short.

**In Errors and Warnings, what does "row 20" or "run" mean?**
The row number is the position in the incoming list (1 = first employee). **"run"** (row 0 in the API) means a note about the whole
payload, for example *"Sensitive fields were dropped at the boundary: PAN No."*

---

## 3. The five API endpoints

All need an `admin` key (or the Admin UI sign-in). The Import runs pages use exactly these.

| Endpoint | In simple words |
|---|---|
| `GET /api/v1/admin/import-runs` | List runs, newest first. Optional `status`, `source` (Core/Excel), `page`, `pageSize` (default 50, max 500). |
| `GET /api/v1/admin/import-runs/{id}` | One run with its problems list (`issues`). |
| `GET /api/v1/admin/import-runs/{id}/raw` | The payload exactly as received (404 after 30 days). |
| `GET /api/v1/admin/import-runs/{id}/changes` | Which employees the run created, updated, deactivated or reactivated, with old → new values. Optional `kind=Created\|Updated\|Deactivated\|Reactivated`. |
| `POST /api/v1/admin/import-runs/{id}/approve` | Apply a run the safety brake rejected. |
| `POST /api/v1/admin/import-runs/{id}/replay` | Process a stored payload again with today's mapping (creates a new run). |

**What do the answers look like?** `approve` and `replay` answer like an import: `200` with the counts, `422` if the safety brake
stops it, `400` if the payload cannot be read, `404` unknown run, `409` not allowed (see section 4).

---

## 4. Safety brake, approve and replay

**What is the safety brake?**
A check before a run is applied. It stops runs that look like a broken list rather than real leavers:

1. the run would **deactivate more than 5 %** of active employees, or
2. the list is **smaller than 80 %** of the previous successful run.

Both limits can be changed without a redeploy in **Settings** (`Import:SafetyBrake:MaxDeactivatePercent`,
`…:MinPayloadPercentOfPrevious`). Manually added people (contractors, interns) are not counted.

**What happened with run #4 ("a broken half-size list")?**
Run #3 had 60 employees; run #4 arrived with only 30. Trusting it would have marked 30 people as left in one go, and every connected
app would have disabled their accounts although they still work here. Half the company does not leave overnight; a list that shrinks
like that almost always means the export broke (crashed half-way, wrong filter, timeout). The brake saw *30 of 59 active would be
deactivated (51 %) — limit 5 %* and *30 is 50 % of 60 — minimum 80 %*, so PeopleHub applied **nothing**, saved the run as
**Rejected** and raised a **Critical alert**. The next night run #5 arrived complete, and run #4 became **Superseded**.

**When should I approve a rejected run?**
Only when the big change is genuine (e.g. a whole process really moved out on one day). Open the run, check the numbers and the raw
payload, then **Approve and apply**. The brake is skipped for that one run; the run becomes *Succeeded* and records who approved it.
If the list was simply broken, do nothing and tell the ASP CORE team.

**Why can't I approve run #4 any more?**
Approval is refused (409) when:

- the run is not *Rejected* (only rejected runs can be approved),
- **a newer run has already succeeded** — approving the old list would undo the newer data (that is run #4's case: *Superseded*),
- the raw payload is older than 30 days and gone.

**What does Replay do?**
It takes the stored payload of any run and processes it **again with today's field map and status mappings**, as a **new** run
(trigger *Replay*, pointing back to the original). The original run stays as it was. Use it after fixing a mapping — that is how
run #1 (all rows skipped) became run #2 (60 employees created). The safety brake still applies. Replaying the same data twice is
harmless: unchanged rows stay unchanged.

**Is it recorded who approved, replayed or uploaded?**
Yes, in two places: the run's page (caller, *approved by*, "Replay of #N") and the **Audit log** (`run.approved`, `run.replayed`,
`run.uploaded` with the outcome; filter `target=run:`). See the [dashboard and audit FAQ](dashboard-and-audit-faq.md#5-audit-log-what-is-recorded).

**Approve vs replay — which one?**
*Approve* = "this rejected run is right, apply it as it is". *Replay* = "read this payload again, now that I fixed how it is read".

---

## 5. Skipped rows, field names and extra data

**Why were all 60 rows of run #1 skipped?**
ASP CORE sent the code as `emp_id` and the name as `full_name`. Before the field map was set, PeopleHub did not know those meant
`employeeCode` and `name`, so every row looked like it had **no employee code** — and a row without a code cannot be stored
(PeopleHub would not know who it is, could not update the right person next time, could not tell apps who changed). Each row was
skipped with *"employeeCode is required"*. Nothing was lost: the raw payload was kept, the field map was fixed, and run #1 was
replayed as run #2.

**But unknown fields are kept in `extra` — why not here?**
`extra` keeps the *additional* fields of a **valid** row. It cannot rescue a row that has no ID, because there is no person to attach
the data to. A row needs at least `employeeCode` and `name`.

**So if only `employeeCode` and `name` had been mapped, would it have worked?**
The rows would have been **imported**, but not completely right:

- `email`, `department`, `designation`, `location`, `status` already match PeopleHub's names → proper fields;
- `doj` and `manager_id` (unknown names) → kept in `extra`, so *date of joining* stays empty and *nobody has a manager*;
- `gender` → `extra` (intended); `salary` → dropped (sensitive).

So map every field apps need. Two safety nets: **Field map → Preview** lists every incoming field and what it becomes
(`doj → (extra)` shows a forgotten mapping), and **Strict mode** (`Import:Strict`, switch on once the mapping is agreed) stops the
whole import when a required column is missing, and skips a person whose required value is blank instead of importing them incomplete.

**Which incoming names match without a field-map row?**
Any name that equals a PeopleHub field once case, spaces and separators are ignored: `employeeCode`, `Employee Code`,
`employee_code`, `Date of Joining`, `Manager Employee Code` … all match automatically.

**Can I add and remove field-map rows?**
Yes, on **Field map**: fill an empty row (incoming name + PeopleHub field) and *Save*; clear a row's incoming name and *Save* to
remove it; pick *(drop this field)* to throw a field away instead of keeping it in `extra`. *Preview* tries a draft against the last
payload without saving. Changes apply from the next import; use **Replay** to apply them to a stored payload. **Settings →
Import:FieldMap → Reset** returns to the appsettings version.

**If a row is skipped, is that person marked as left?**
No. A skipped row's employee code is remembered for that run, so the person keeps their current data and stays active. Example from
run #6: ASP0033 had a broken email (`employee33@`), the row was skipped with an error, and ASP0033 is still active.

**What happens with a status PeopleHub does not know (e.g. "Sabbatical")?**
It is imported, stored raw, shown as status **Unknown**, and listed as a warning. **Status mappings** shows it under *Seen but not
mapped*; map it and replay the last run.

**How do apps read the `extra` data?**
Only apps whose key has the `employees:read:extra` scope receive it: `"extra": { "Gender": "Female" }` in the API and events,
`employee.Extra` in PeopleHub.Client, and the `extra` jsonb column of the app's local copy. Keys are exactly as the source sent them
— ASP CORE sent `gender`, the Excel sheet `Gender` — so read them case-insensitively. `extra` is not part of the fixed contract: if the
source renames a field, the key changes. If several apps depend on a field, promote it to a fixed PeopleHub field.

**Is a change inside `extra` recorded in history?**
Yes, including a change of letter case only (`gender` → `Gender`). (Before the fix of 1 Nov 2026 a case-only change was stored but not
written to history; runs from before that, such as #6, do not show it.)

---

## 6. Excel uploads

**How does an Excel upload work?**
**Imports → Excel upload**, choose an `.xlsx`, leave *Sheet* empty for the first sheet, *Import*. The sheet goes through the same
mapping, checks, safety brake, history and events as ASP CORE's push, and is a **full list** too: anyone missing is treated as having
left. Column headers that differ from PeopleHub's names go in **Field map → Excel-only column headers**.

**Why did my upload show "Updated 59" when only one person was promoted?**
Everyone already in PeopleHub came from ASP CORE (`source = Core`). The upload switched each record to `source = Excel`, and that
counts as a change (apps receive an *updated* event for it). Upload the same sheet again and every count is 0.

**Run #6 (`hr-employees-sample.xlsx`) — what was in it and where do I see each item?**

| Row in the sheet | Result | Where to see it |
|---|---|---|
| ASP0061 Meena Krishnan (new joiner) | Created | Run #6 → Changes in this run → *Created*; employee page → History "Record created" |
| ASP0007 promoted to Team Lead | Updated | Changes in this run; employee page → History `designation: Associate → Team Lead` |
| ASP0045 resigned, Active = No | Deactivated | Changes → *Deactivated*; employee page shows *Inactive · Resigned*, reason `feed_inactive` |
| ASP0033 broken email | Skipped, person stays active | Run #6 → **Errors** (`invalid_email`); employee page has no run #6 entries |
| ASP0020 status "Sabbatical" | Imported as Unknown | Run #6 → **Warnings** (`unknown_status`); **Status mappings** → *Seen but not mapped* |
| *Mobile No* column | Kept in `extra` | Employee page → *Extra fields* |
| *Gender* column | Kept in `extra` | Employee page → *Extra fields* `{"Gender":"Female"}` |

The test sheets `hr-employees-sample.xlsx` and `hr-employees-broken-half.xlsx` (only 30 people: the safety brake should reject it)
are in `samples/excel` on the development machine; they match the local demo data only and are not in Git.

**Where is the "Download uploaded workbook" button?**
On the **run's own page**, not the list: **Import runs → click the Excel run's number** → buttons above *Errors*. It appears only for
Excel runs whose file is still kept (30 days).

---

## 7. Raw payload: what is kept and for how long

**What is the raw payload?**
The data exactly as it arrived, before any mapping or cleaning. For a push it is ASP CORE's JSON; for an Excel upload it is the sheet
converted to rows (the original `.xlsx` is kept alongside and can be downloaded). It is kept so admins can see exactly what was sent
and **replay** it after fixing a mapping.

**How long is it kept? How many runs?**
**30 days per run**, counted from when the run arrived; limited by age, not by number of runs. Every run keeps its own copy (each push,
each upload and each replay). A nightly clean-up at 02:00 IST deletes older copies. Change the period in **Settings →
Import:RawRetentionDays**.

**What remains after the 30 days?**
The run stays listed with its counts, errors, warnings and *Changes in this run*, and every employee's history is kept for 2 years.
What goes is the original payload (and the Excel file), so **Replay**, **Approve** and **Download uploaded workbook** stop working for
that run.

**"Only the employee records are cleaned" — what does that mean?**
PeopleHub keeps two different things:

| | Raw copy | Employee records |
|---|---|---|
| What | The payload as it arrived, untouched | The cleaned data PeopleHub stores |
| Who sees it | Admins only | Every connected app (API, events) |
| How long | 30 days | Permanently (history 2 years) |
| Sensitive fields (salary, PAN, Aadhaar, bank details…) | **still there** | **removed** |

So a sensitive field never reaches any app, but the admin-only raw copy still contains it for up to 30 days. (Possible change if
preferred: strip deny-listed fields from the raw copy as well before saving it; replay would still work.)

---

## 8. Finding out exactly what a run changed

**Which employees did a run create, update or deactivate?**
Run page → **Changes in this run**. One row per employee: the kind of change and every field that changed (old → new). Filter
buttons: *All*, *Created*, *Updated*, *Deactivated*, *Reactivated*. Each code links to the employee's page. The same list for scripts:
`GET /api/v1/admin/import-runs/{id}/changes?kind=…`. A run stopped by the safety brake lists nothing ("Nothing was applied").

**Where is one person's full story?**
**Employees → search the code → open it → History**: every change, newest first, tagged with the run (`run #6`) or the admin who made
it.

---

## 9. When the data turns out to be wrong

**A run succeeded but some data is wrong. Is there an undo?**
There is no "undo run" button on purpose. Corrections go **forward**: fix the source, send the data again; PeopleHub records the
correction in history and tells the apps like any other change. Pick by where the mistake came from:

| The mistake is… | Do this |
|---|---|
| **Wrong in greytHR** (department, manager, someone marked as left by mistake) | Fix it in greytHR. The next push corrects it, or ask the ASP CORE team for an extra push now (resending is always safe). |
| **PeopleHub read it wrongly** (field mapped to the wrong place, status mapped wrongly) | Fix **Field map** / **Status mappings**, then the run → **Replay with current mapping**. |
| **Urgent, greytHR cannot be fixed today** | Correct HR's Excel sheet and upload it — the **complete** list (anyone missing is treated as left). |
| **A whole run was bad** (e.g. a rejected run approved by mistake) | Open the **last good run before it → Replay**. Its list is re-applied as a new run (brake still checks it). Genuine changes since then are lost until the next correct push; emergency use only, within 30 days. |

**Why can't I just edit the employee in PeopleHub?**
greytHR is the source of truth. Employees from ASP CORE or Excel cannot be edited in PeopleHub — the next push would overwrite the
edit anyway. (Manually added people, `M-…`, are the exception: they are edited in PeopleHub.)

**What happens in the connected apps?**
They already received the wrong change and now receive the correction as a new event (*updated*, or *reactivated* for someone wrongly
marked as left); their handlers apply it like any other change. History keeps both steps, so the audit trail stays honest.

---

## 10. Rehire links and undoing a wrong link

**What does "Same person" on Rehire review do?**
When greytHR gives a returning employee a new code (e.g. ASP0100 left, rejoins as ASP0890), PeopleHub suggests the link. *Same
person* makes the old code an **alias** of the new one: the old record is retired, lookups of the old code return the new record,
managers listed under the old code point to the new one, and every app receives `employee.alias_linked` and merges its records.

**How do wrong links happen, and what prevents them?**
Matches by **same email** are strong. Matches by **same name and department only** are weak — two "Priya Sharma"s in one team happen.
For weak matches the admin must tick **"I checked both records: same person"** before *Same person* works.

**I linked two different people by mistake. How do I undo it?**
**Employees → open the current person (new code) → Old codes (aliases) → Unlink**, with a reason. The old code becomes its own record
again **exactly as before the link** (a leaver stays a leaver; a converted contractor is active again), older codes carried by the
link go back, the review suggestion becomes *Dismissed*, and apps receive `employee.alias_unlinked` (`OnAliasUnlinkedAsync` in
PeopleHub.Client). Each app decides how to split what it merged — tell the app owners. Who linked, who unlinked and why stay in the
audit log. API: `POST /api/v1/admin/employees/{code}/aliases/{aliasCode}/unlink` with `{ "reason": "…" }`.

---

## 11. Related: PeopleHub's fixed fields

**Are PeopleHub's fields fixed?**
Yes. The fields an incoming name can be mapped to are fixed: `employeeCode, name, email, department, designation, managerEmployeeCode,
managerEmail, location, shift, grade, level, hrmsId, dateOfJoining, active, status, leavingDate`. Anything else goes into `extra` (sensitive fields are
dropped).

**Why fixed?**
Apps are built against those names. The source side (greytHR / ASP CORE / HR's sheet) can rename or add columns at any time; the field
map absorbs that in **one place**, so no app has to change or redeploy. Fixed names also mean "department" means the same thing in
every app.

**Can fields be added or removed?**
*Adding* is safe (old apps ignore fields they don't know) but is a small code change, not an Admin UI action: a new field needs storage
with history, validation rules, a scope (privacy: which apps may see it) and a client-library property. Until then it can live in
`extra`. *Removing* a field would make apps silently get empty values, so it only happens in a new API version (`/api/v2`) while v1
keeps sending it.

---

## 12. Quick reference

| I want to… | Go to |
|---|---|
| See whether last night's push arrived | **Dashboard** (last run per source, next push expected) or **Import runs** |
| Understand why a run was rejected | Run page → facts (*Safety brake* numbers) → raw payload |
| Apply a rejected run that is genuine | Run page → **Approve and apply** |
| Re-read a payload after fixing the mapping | Run page → **Replay with current mapping** |
| Try a mapping without saving | **Field map → Preview**, or run page → *Try a mapping on this payload* |
| See who a run changed | Run page → **Changes in this run** |
| See one person's changes | **Employees** → person → **History** |
| Get the uploaded Excel file back | Run page (Excel run) → **Download uploaded workbook** |
| Map an unknown status | **Status mappings** → *Seen but not mapped* |
| Change the brake limits or raw retention | **Settings** |
| Undo a wrong "same person" link | **Employees** → current person → **Old codes → Unlink** |

See also: [client library FAQ](client-library-faq.md) · [import FAQ](import-faq.md) (the push and the Excel upload) · [Employees and Changes FAQ](employees-and-changes-faq.md) (how apps get the data) ·
[clients and API keys FAQ](clients-and-api-keys-faq.md) · [webhooks FAQ](webhooks-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md) · [manual people FAQ](manual-people-faq.md) · [dashboard and audit FAQ](dashboard-and-audit-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md) · [IT contract](../it-contract.md) (what ASP CORE sends) ·
[onboarding](../onboarding.md) (connecting an app).
