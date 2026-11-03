# PeopleHub import contract for the ASP CORE team

**What we ask:** on a schedule, ASP CORE sends PeopleHub the **complete list of employees** from greytHR in one HTTPS call.
PeopleHub is the only consumer of this push; every other internal app reads from PeopleHub, so greytHR's rate limit is not touched.

## 1. The call

| | |
|---|---|
| **URL** | UAT: `https://<uat-peoplehub-host>/api/v1/import/employees` · Prod: `https://<prod-peoplehub-host>/api/v1/import/employees`. While UAT has no certificate its address starts with `http://`: use the exact address we give you. |
| **Method** | `POST` |
| **Headers** | `Content-Type: application/json` · `X-Api-Key: <key we give you>` (one key per environment; keep it secret) |
| **Source IP** | Calls must come from the ASP CORE server(s); tell us their IPs so we can allow-list them. |
| **Schedule** | Nightly (we suggest 01:00 IST). If you can push more often (every few hours) or on demand, tell us — no change needed on your side. |
| **Size** | All employees in one call (≈ 600 rows, well under 1 MB; the limit is 20 MB). |

## 2. The body

A JSON array, one object per employee — **every employee greytHR has, including those who left** (with `active: false`).
An envelope such as `{ "employees": [ … ] }` is also fine. **Anyone missing from the list is treated as having left**, so never send a partial list.

```json
[
  {
    "hrmsId": "1042871",
    "employeeCode": "ASP1042",
    "name": "Priya Raman",
    "email": "priya.raman@example.com",
    "department": "AR Follow-up",
    "designation": "Senior Associate",
    "managerEmployeeCode": "ASP0311",
    "managerEmail": "k.iyer@example.com",
    "location": "Chennai",
    "shift": "US Night",
    "dateOfJoining": "2023-04-17",
    "active": true,
    "status": "Notice Period",
    "leavingDate": "2026-11-30",
    "lastModified": "2026-10-30T18:22:05+05:30"
  }
]
```

| Field | Required | Type / format | Notes |
|---|---|---|---|
| `employeeCode` | **yes** | text | The company's own employee code. Never reuse a code for a different person. |
| `hrmsId` | yes* | text | greytHR's own unique key for the person. Must be different for every person; a rehire gets a new one. |
| `name` | **yes** | text | Full name. |
| `email` | yes* | text | Official email. Used to detect rehires and to match managers. |
| `department` | yes* | text | |
| `designation` | yes* | text | |
| `managerEmployeeCode` | preferred | text | Reporting manager's employee code. |
| `managerEmail` | if no code | text | Only needed when the manager's code is not available. |
| `location` | no | text | |
| `shift` or `shiftName` | when known | text | The employee's shift exactly as greytHR shows it (e.g. `General`, `US Night`). |
| `dateOfJoining` | yes* | date `yyyy-MM-dd` | Date only, no time. `dd/MM/yyyy` also accepted. |
| `active` | yes* | `true` / `false` | `false` for resigned, absconded, terminated. (`1/0`, `Y/N`, `Yes/No` also accepted.) |
| `status` | yes* | text | greytHR status exactly as greytHR shows it (Active, Notice Period, Resigned, Absconded, Terminated, …). |
| `leavingDate` | when known | date `yyyy-MM-dd` | |
| `lastModified` | no | timestamp | ISO 8601 with offset; without an offset we read it as IST. |

\* Rows missing these are accepted today; once we switch on strict validation (after the first test pushes) they will be rejected.

**Field names:** if ASP CORE already uses other names (`emp_id`, `doj`, …) or adds more fields, send them as they are — we map names on our side.
Extra fields are kept, not lost.

**Never send:** salary/CTC, PAN, Aadhaar, passport, UAN, bank details. If they arrive we drop them and log it, but please leave them out.
Other personal fields, such as date of birth, gender or phone number, are kept as extra fields if you send them.

## 3. The answer

| HTTP | Meaning | What ASP CORE should do |
|---|---|---|
| `200` | Imported. Body: `{ "runId", "status": "Succeeded", "received", "created", "updated", "deactivated", "reactivated", "skipped", "errors": [ … ], "warnings": [ … ] }` | Log `runId` and the counts. Rows listed in `errors` were skipped (the rest were imported); we fix those with HR. |
| `422` | **Rejected by the safety brake**: the list would deactivate more than 5 % of active employees, or is less than 80 % of the previous list. Nothing was applied; we are alerted. | **Do not retry.** Usually a partial or failed extract — check the export. If the change is genuine (mass exit), we approve it on our side. |
| `400` | Body is not readable JSON or has no employee array. Stored for diagnosis. | Fix the export; retry. |
| `401` / `403` | Missing or wrong key, or call from an IP not on the allow-list. | Check the key and source IP. |
| `409` | Another import is still running. | Retry after a few minutes. |
| `5xx` / timeout | PeopleHub unavailable. | Retry up to 3 times, 5 minutes apart. Resending the same list is always safe (it changes nothing twice). |

## 4. Getting started

1. We send you the **UAT** URL and key.
2. You send **one sample push** to UAT. We inspect exactly what arrived, set the field mapping, and confirm status values with you.
3. You schedule the nightly push to UAT for a few days; we compare with greytHR.
4. We switch on strict validation, then send the **Prod** URL and key.

If no successful push arrives within 26 hours we are alerted automatically. Contact: PeopleHub team, `<email / Teams channel>`.

## 5. When you receive your key

We send you the address and the key privately. They are not in this guide.

1. **Tell us your server's IP address**, so we can allow it. Until we do, your calls may get `403`.
2. **Store the key safely:** in your server's settings or secret store, never in code, and not in email or chat. It only allows sending the employee list.
3. **Try one call by hand.** Use made-up test employees while the UAT address starts with `http://`, because the key and the data travel unencrypted until UAT has a certificate. Your file must be a JSON list like the one in section 2:

   ```bash
   curl -X POST <address we give you> -H "X-Api-Key: <your key>" -H "Content-Type: application/json" -d @employees.json
   ```

   A `200` answer means it worked. Other answers are explained in section 3.

4. **Tell us when you have sent it.** We check what arrived, set up the field names on our side, and tell you what to change, if anything.
5. **Build the real push and schedule it** nightly, once we confirm the sample. After a full list has been sent, a much shorter list is rejected by the safety brake, and a slightly shorter one marks the missing people as having left, so always send everyone.

If the key is lost or may have leaked, tell us straight away. We issue a new one and stop the old one.
