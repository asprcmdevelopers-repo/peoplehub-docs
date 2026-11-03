# Field list and permissions

This page lists every field ASP CORE sends to PeopleHub, where each one is stored, and which permission lets an app read it.

The two things are separate. **The push** decides what arrives in PeopleHub, and permissions play no part in it. **Permissions** only control what each app can read afterwards.

---

## 1. What ASP CORE sends

**Main fields**

| ASP CORE field | Hub field | In IT's mail? | Column required? | Can one person's value be blank? |
|---|---|---|---|---|
| `employeeCode` | `employeeCode` | Yes | Yes | No |
| `hrmsId` | `hrmsId` | Yes | Yes | No |
| `name` | `name` | Yes | Yes | No |
| `email` | `email` | Yes | Yes | No |
| `department` | `department` | Yes | Yes | No |
| `designation` | `designation` | Yes | Yes | No |
| `dateOfJoining` | `dateOfJoining` | No, IT must add it | Yes | No |
| `status` | `status` and `statusRaw` | Yes | At least one of the two | No |
| `active` | `active` | Yes | At least one of the two | No |
| `shiftName` | `shift` | Yes | Yes | Yes |
| `managerEmployeeCode` | `managerEmployeeCode` | No, IT is asked to add it | At least one of the two | Yes |
| `managerEmail` | `managerEmail` | Yes | At least one of the two | Yes |
| `location` | `location` | No | No | Yes |
| `leavingDate` | `leavingDate` | Yes | No | Yes |
| `lastModified` | `lastModified` | No | No | Yes |

`shiftName` is the only field whose name changes. `status` is stored twice: `status` holds the standard word, such as `NoticePeriod`, and `statusRaw` holds greytHR's exact words, such as `Notice Period`.

**Extra data**

None of these are required, and they are stored under the same name ASP CORE sends:

| ASP CORE field | Hub location |
|---|---|
| `managerName` | `extra.managerName` |
| `grade` | `extra.grade` |
| `gender` | `extra.gender` |
| `dateOfBirth` | `extra.dateOfBirth` |
| `phone` | `extra.phone` |
| Any other field not listed here | `extra.<same name>` |

**Not stored on the employee**

If ASP CORE sends any of these, they are dropped from the employee record and a warning is logged: salary, CTC, gross pay, net pay, PAN, Aadhaar, SSN, passport, bank account number, IFSC and UAN.

The admin-only raw copy of the push still contains them for 30 days.

---

## 2. How the check works

The required column and blank value checks run when strict mode is on. Until then, only `employeeCode` and `name` are needed on each row.

- **Required column missing, or empty for everyone:** the whole push stops and nothing changes. The run shows **Failed**, and ASP CORE gets a `400` naming the columns.
- **One person's value is blank where the table says "No":** that person is skipped and reported, and everyone else is imported.
- **`status` and `active`:** blank means that person has neither value. Please send both.
- **Manager:** a person with neither value is still imported, because the top person in the company has no manager.

An Excel upload needs the same columns, except `hrmsId`.

---

## 3. Permission groups

| Permission | What it is for |
|---|---|
| `employees:read:basic` | Who the person is. Every app that reads employees needs this one. |
| `employees:read:org` | Where the person sits in the company. |
| `employees:read:dates` | When the person joined and left. |
| `employees:read:extra` | All the extra data. |

Other permissions, such as `webhooks:subscribe`, `import:write` and `admin`, are not about reading employee fields, so they are in the [client apps and API keys guide](clients-and-api-keys-faq.md).

---

## 4. Fields in each permission group

**`employees:read:basic`**

| Field | Comes from |
|---|---|
| `employeeCode` | ASP CORE |
| `hrmsId` | ASP CORE |
| `name` | ASP CORE |
| `email` | ASP CORE |
| `active` | ASP CORE |
| `status` | ASP CORE, as the standard word |
| `statusRaw` | ASP CORE, as greytHR's exact words |
| `shift` | ASP CORE's `shiftName` |
| `lastModified` | ASP CORE |
| `source` | PeopleHub: where the record came from |
| `aliases` | PeopleHub: the person's old employee codes |

**`employees:read:org`**

| Field | Comes from |
|---|---|
| `department` | ASP CORE |
| `designation` | ASP CORE |
| `managerEmployeeCode` | ASP CORE |
| `managerEmail` | ASP CORE |
| `location` | ASP CORE |
| `managerResolved` | PeopleHub: whether the manager was found |

**`employees:read:dates`**

| Field | Comes from |
|---|---|
| `dateOfJoining` | ASP CORE |
| `leavingDate` | ASP CORE |

**`employees:read:extra`**

| Field | Comes from |
|---|---|
| `extra` | ASP CORE: `managerName`, `grade`, `gender`, `dateOfBirth`, `phone` and any other field not listed in section 1 |

**How the permissions work together**

- **Basic comes first.** An app needs `employees:read:basic` to read any employee, so a key with only `employees:read:extra` gets `403`.
- **Permissions add up.** An app with basic and org sees both groups of fields.
- **A field the app may not read is left out.** It doesn't arrive as empty, so an app can tell "not allowed" from "no value".
- **Extra data is all or nothing.** An app with `employees:read:extra` sees every extra field, and an app without it sees none.

---

See also: [what ASP CORE must send](../it-contract.md) · [import FAQ](import-faq.md) (the push and the Excel upload) · [client apps and API keys](clients-and-api-keys-faq.md) · [mappings, settings and alerts FAQ](mappings-settings-and-alerts-faq.md)
