# Client library — help and FAQ

Plain-English answers for **app teams** (Timesheet, the training portal …) connecting a .NET app to PeopleHub:

- **`PeopleHub.Client`** — the library: lookups, change events, webhooks.
- **`PeopleHub.Client.PostgreSql`** — a ready-made local copy of the employees in the app's own PostgreSQL database.
- **`peoplehub-backfill`** — a one-time job for apps that already have users.
- **The sample app** — a small working "Timesheet-lite" to copy from.

The step-by-step checklist is [onboarding](../onboarding.md); this guide explains *why* each piece works the way it does.

**Contents**

1. [The basics](#1-the-basics)
2. [Asking questions: `IPeopleHubClient`](#2-asking-questions-ipeoplehubclient)
3. [The handler: your app's rules](#3-the-handler-your-apps-rules)
4. [How a change reaches your app](#4-how-a-change-reaches-your-app)
5. [The local copy (PostgreSQL)](#5-the-local-copy-postgresql)
6. [Backfill: apps that already have users](#6-backfill-apps-that-already-have-users)
7. [The sample app](#7-the-sample-app)
8. [Everyday situations](#8-everyday-situations)
9. [Good to know](#9-good-to-know)
10. [Quick reference](#10-quick-reference)

---

## 1. The basics

**Why use the library at all?**
Without it, every team writes the same code: calling the API with the key, retries, reading `/changes`, remembering the cursor, checking
webhook signatures, skipping duplicates. With it, a team writes **only its own rules**.

**What does an app add?**

```csharp
builder.Services.AddPeopleHubClient();                                     // 1. address, key, webhook secret from configuration
builder.Services.AddPeopleHubHandler<MyEmployeeHandler>();                 // 2. your rules
builder.Services.AddPeopleHubLocalSync(o => o.ConnectionString = appDb);   // 3. local copy + background sync
app.MapPeopleHubWebhooks("/peoplehub/events");                             // 4. webhook receiver
```

```json
"PeopleHub": {
  "BaseUrl": "https://peoplehub-uat.example.com/",
  "ApiKey": "<from an environment variable or secret store>",
  "WebhookSecret": "<after registering the webhook>"
}
```

The app **refuses to start** if `BaseUrl` is not a full URL or `ApiKey` is empty, so a configuration mistake shows up at once.

**What belongs where?**

| | Lives in |
|---|---|
| HR facts (name, department, manager, shift, status) | **PeopleHub**; the app gets a copy |
| App decisions (roles, cost centres, a locked role) | **The app's own tables**, keyed by employee code |
| App rules ("Team Lead means approver") | **The app's handler** |
| App problems ("no role rule for Billing / Clerk") | **The app's own review list**, never back to PeopleHub |

**The settings**

| Setting | Default | Meaning |
|---|---|---|
| `PeopleHub:BaseUrl`, `ApiKey` | required | Which hub, and this app's key. One key per app, per environment. |
| `PeopleHub:WebhookSecret` | — | Secret of this app's webhook (`WebhookSecrets` when two are valid during a rotation) |
| `PeopleHub:CacheTtl` | 2 minutes | How long one employee's lookup is remembered; `00:00:00` switches it off |
| `PeopleHub:FallbackToLocalStore` | `true` | Answer lookups from the local copy when PeopleHub cannot be reached |
| `PeopleHub:LocalSync:Enabled` | `true` | Run the background sync |
| `PeopleHub:LocalSync:PollInterval` | 5 minutes | How often `/changes` is read |
| `PeopleHub:LocalSync:PageSize` | 500 | Rows per page |
| `PeopleHub:LocalSync:RaiseLeftOnInitialLoad` | `false` | Also call `OnLeftAsync` for people who had already left, during the first load |

---

## 2. Asking questions: `IPeopleHubClient`

Inject `IPeopleHubClient` wherever you need employee data.

| Method | Answers | Cached 2 min | Local copy if PeopleHub is unreachable |
|---|---|---|---|
| `GetEmployeeAsync(code)` | One person; `null` if unknown. Old codes give the current record. | Yes | **Yes** |
| `IsActiveAsync(code)` | Active? An unknown code is `false`. | Yes | **Yes** |
| `GetManagerAsync(code)` | The person's manager, or `null` | No | **Yes** |
| `GetDirectReportsAsync(code)` | Who reports to this person | No | No |
| `SearchEmployeesAsync(query)` | Everyone matching, going through the pages by itself | No | No |
| `GetModifiedSinceAsync(since)` | Changed since a time | No | No |
| `GetDepartmentsAsync()` | Departments with head count | No | No |
| `GetChangesSinceAsync(cursor)` | A page of change events (the sync does this for you) | No | No |
| `MatchAsync(request)` | Guesses for an existing user (backfill does this for you) | No | No |
| `SubscribeWebhookAsync(url)` | Registers the webhook; returns the secret once | No | No |

**What does a field I may not see look like?**
`null`. `Department` needs `employees:read:org`, `DateOfJoining` needs `employees:read:dates`,
`Extra` needs `employees:read:extra`. See the [clients and API keys FAQ](clients-and-api-keys-faq.md#2-scopes-what-an-app-may-do).

**What if PeopleHub has a hiccup?**
The library retries by itself: up to 20 seconds per attempt and 60 seconds in total. If PeopleHub keeps failing it stops calling for a
while (a *circuit breaker*), so your pages fail fast instead of hanging.

**What does my code see when it finally fails?**

| Situation | Result |
|---|---|
| PeopleHub unreachable, local copy available, one of the three lookups above | The answer from the local copy; a warning in your log |
| PeopleHub unreachable, any other call | `PeopleHubException`, `StatusCode` empty |
| PeopleHub answered with an error (wrong key `401`, missing scope `403`, too many calls `429`, `500`) | `PeopleHubException` with the `StatusCode`. **The local copy is not used**: this means something is wrong, not just down. |
| Unknown code on a lookup (`404`) | `null`, no exception |

**How fresh is a cached answer?**
At most 2 minutes old, and usually newer: when a change event for that person arrives, their cached entry is thrown away at once.

---

## 3. The handler: your app's rules

This is the one class you write. Derive from `EmployeeChangedHandler` and override only what you care about:

```csharp
public sealed class MyEmployeeHandler(MyDb db) : EmployeeChangedHandler
{
    public override Task OnLeftAsync(Employee employee, CancellationToken ct = default) =>
        db.BlockLoginAsync(employee.EmployeeCode, ct);
}
```

| Method | When | Example for Timesheet |
|---|---|---|
| `OnJoinedAsync` | New person, **and** everyone active during a first load or resync | Create or refresh the user, give a role |
| `OnUpdatedAsync(employee, changedFields)` | Fields changed; `changedFields` has old and new | Designation became "Team Lead" → approver |
| `OnLeftAsync` | Left or deactivated | Block login, hand over pending approvals |
| `OnReactivatedAsync` | Back | Unblock login |
| `OnManagerChangedAsync(employee, old, new)` | New manager | Move their pending approvals |
| `OnAliasLinkedAsync(employee, oldCode)` | Two codes are one person (rehire, or a contractor who joined greytHR) | Merge the records |
| `OnAliasUnlinkedAsync(employee, oldCode)` | That link was a mistake | Split, or flag it for a person to check |

**Why "idempotent"? What does that mean?**
It means running the method twice has the same result as running it once. Duplicates are filtered, but after a crash or retry your method
can still run again. "Block login" twice is fine; "send a welcome email" twice is not. If you send emails, record that you sent them.

**`OnJoinedAsync` runs for existing people too?**
Yes: during the first load and every full resync it runs for **every active employee**. So write it as "create **or refresh**": find the
user by employee code and update it if it exists. Do not send welcome emails from it without checking.

**Where do I find a shift change?**
In `OnUpdatedAsync`: `changedFields` contains `shift` (with `OldText` and `NewText`), and `employee.Shift` has the new value.

---

## 4. How a change reaches your app

A change can arrive three ways, and all three take the **same road**, so your handler is written once:

```
PeopleHub ──webhook (seconds)──┐
          ──/changes (5 min)───┼──► event processor ──► YOUR handler ──► your tables
          ──full load (once)───┘    skip duplicate/stale     └──► local HR copy
```

For each event the library:

1. **skips it if this `eventId` was already handled** (a webhook and a poll can bring the same event);
2. **skips it if it is older** than what was applied for this person, so a late event cannot overwrite newer data;
3. **calls your handler**;
4. **saves** the person's HR copy and marks the event as done.

**What if my handler throws an error?**
Nothing is marked as done, so the event comes again:

- **Webhook:** the library answers `500` and PeopleHub retries (see the [webhooks FAQ](webhooks-faq.md)).
- **Polling:** the cursor stays there and the next poll tries again. **Later changes wait behind it**, so a handler that always fails on one
  event blocks the sync until you fix it. Watch your log for `PeopleHub sync failed; will retry`.

Do not swallow errors you want retried. If one record can never succeed, put it on your own review list and return normally.

**What does the first sync do?**
With no cursor yet, it loads the full list page by page, calls `OnJoinedAsync` for each active person, then stores the cursor. Changes made
during the load are picked up right after it. If it fails halfway, the next run starts the load again.

**What does `resyncRequired` do?**
If an admin requested a resync, or the app was away longer than PeopleHub keeps events, PeopleHub answers `resyncRequired` and the library
does the full load again by itself. See the [Employees and Changes FAQ](employees-and-changes-faq.md#4-full-resync).

**What does the webhook endpoint answer?**

| Answer | Meaning |
|---|---|
| `200` | Handled, or already handled |
| `401` | Bad or expired signature (older than 5 minutes: check the server clocks) |
| `400` | The body could not be read |
| `500` | Your handler failed; PeopleHub will retry |
| `503` | `PeopleHub:WebhookSecret` is not set |

**Do I need webhooks?**
No. Polling alone gives changes within 5 minutes. Webhooks bring them within seconds; polling stays on as the safety net. See
[how often should an app poll](employees-and-changes-faq.md#7-how-often-should-an-app-poll).

---

## 5. The local copy (PostgreSQL)

**Why keep a copy?**

1. **The app keeps working when PeopleHub is down**: logins and "who is the manager?" are answered from the copy.
2. **It remembers what was handled**, so a change is never applied twice, even after a restart.

**Which tables does it create?**
Three, in their **own schema** (`peoplehub`), so they never clash with yours:

| Table | Holds |
|---|---|
| `peoplehub.employees` | One row per person: the HR fields, old codes, `extra`, plus `last_sequence`, `merged_into`, `synced_at` |
| `peoplehub.processed_events` | Event ids already handled, kept **30 days** (cleaned up daily) |
| `peoplehub.sync_state` | The cursor: how far in `/changes` the app has got |

**Can I add my own columns?**
You may: the library writes only its own columns. Your **own tables keyed by `employee_code`** are better, because HR data and app data
then never mix.

**What happens with old codes?**
When a code becomes an old code of someone, its row stays with `merged_into` set and is treated as inactive. Looking up the old code returns
the current record, and lists leave the old row out, so nobody appears twice. An unlink removes the marker.

**The database user cannot create tables.**
Set `CreateSchemaOnStartup = false` and have the DBA run the scripts from the package's `content/sql` folder, **in order** (v1, then v2 …).
Each is safe to run more than once.

**What happens when I upgrade the package?**
New columns are added by themselves on the next start (v2 added `shift`, v3 added `hrms_id`). If your DBA runs the scripts, they run the new one first.

**Store settings**

| Setting | Default | Meaning |
|---|---|---|
| `ConnectionString` | required | The app's own database |
| `Schema` | `peoplehub` | Where the tables go (letters, digits, underscores) |
| `CreateSchemaOnStartup` | `true` | Create and upgrade tables at start |
| `ProcessedEventRetention` | 30 days | How long handled event ids are remembered |

**Should I read the copy or the API?**
Normal lookups go through `IPeopleHubClient` (API, then cache, then the copy when PeopleHub is down). For heavy work such as a report on
every employee, read the copy directly: inject `IEmployeeStore` and use `ListAsync`, or query the table.

**My app is not on PostgreSQL.**
Implement `IEmployeeStore` and call `AddPeopleHubLocalSync<MyStore>()`. `PostgreSqlEmployeeStore` is the pattern to copy. There is also an
`InMemoryEmployeeStore` for development and tests; it forgets everything on restart and does the first load again.

---

## 6. Backfill: apps that already have users

**Why is it needed?**
The first sync calls `OnJoinedAsync` for everyone, and your handler looks for a user with that employee code. A user who cannot be found that
way gets **created a second time**, and their history stays on the old one. So every existing user must carry the right code **before the
first sync**. Until then keep `PeopleHub:LocalSync:Enabled` set to `false`.

A brand-new app with no users does not need this.

**Which case am I?**

| | Full match | Check |
|---|---|---|
| **When** | Users have no employee codes | Users already have codes (**Timesheet**) |
| **Export** | `appUserId`, email, name, department | The same **plus `employeeCode`** |
| **What you get** | A proposed code per user | Confirmation, and corrections where a code is wrong or old |

**The three steps**

```bash
peoplehub-backfill match --users users.csv --out review.csv --url https://peoplehub-uat.example.com/ --api-key <key>
```

```bash
peoplehub-backfill apply --review review.csv --template "UPDATE users SET employee_code = {employeeCode} WHERE id = {appUserId};" --out apply.sql
```

1. **match** asks PeopleHub about each user and writes `review.csv`.
2. **review:** someone who knows the users opens it in Excel and sets `accept` to yes or no, or types another code in `proposedCode`.
3. **apply** writes a SQL script with one statement for each accepted row **that changes a code**. It never touches your database: you read
   the script and run it.

**How is `accept` filled in?**

| Situation | `accept` | Note |
|---|---|---|
| One strong match (code or email) | **yes** | "Matched by Email." |
| Only a medium or low match (name) | no | "Only a low match by Name; check before accepting." |
| Several strong matches | no | "Several strong matches; pick one." |
| No match | no | "No match in PeopleHub. Possibly not in greytHR (add as a manual person) or a different email." |
| No code, email or name | no | "Nothing to match on" |
| Two users proposed for one employee | no, for **both** | "Employee … is proposed for more than one app user." |

**What should I look at when checking codes?**

- rows with `accept` = **no**;
- rows where `proposedCode` differs from `currentCode`; the note says *"Differs from the app's code …"*. Usually an old code of someone rehired, or a
  wrong code that the email corrected. These come pre-accepted, so read them.

`apply.sql` is then exactly your list of corrections. Accepted rows that already have the right code produce no statement.

**What about someone PeopleHub does not know?**
A contractor or intern not in greytHR: ask a PeopleHub admin to add them under **Manual people**, then run `match` again. See the
[manual people FAQ](manual-people-faq.md).

**Can I run it from code instead?**
Yes: implement `IBackfillTarget` ("list my users", "attach this code") and use `BackfillRunner`, as `TimesheetBackfillTarget` does in the sample.

**Is it safe?**
It only reads from PeopleHub, so it can run as often as you like, with the same key the app will use. Values starting with `=`, `+`, `-` or `@` are
written so Excel does not run them as formulas, and SQL values are quoted safely.

---

## 7. The sample app

`samples/PeopleHub.SampleApp` is a small fake Timesheet with no screens, only JSON endpoints. **Copy the patterns, not the app.** Its
README explains how to run it (port 5190).

| Look at | To see |
|---|---|
| `Program.cs` | The four lines, and approval routing at `/timesheets` |
| `TimesheetEmployeeHandler.cs` | A complete handler; every method safe to repeat |
| `RoleMapping.cs` + `appsettings.json` | Department and designation → app role, from settings; first matching rule wins |
| `Data/TimesheetDb.cs` | `role_locked`: a role set by an app admin survives every sync |
| `TimesheetBackfillTarget.cs` | Backfill in code, for users with and without codes |

**Ideas worth copying:**

- **Roles are the app's decision.** PeopleHub knows nothing about them; another app may map the same person differently.
- **App decisions survive sync.** HR changes update HR columns; `role_locked` keeps the admin's choice.
- **App problems stay in the app.** "No role rule" and "no active manager" go to the app's own `/review` list.

---

## 8. Everyday situations

**Connecting a new app.**
Follow [onboarding](../onboarding.md).

**PeopleHub is down.**
Logins and manager lookups keep working from the copy. Reports, searches and direct reports fail with `PeopleHubException`, so show a friendly message
or read the copy. The sync catches up when PeopleHub returns.

**My app was off for days.**
It continues from its cursor. If it was away longer than PeopleHub keeps events, a full reload happens by itself.

**My app's data looks wrong as a whole** (restored from an old backup, rows deleted).
Ask a PeopleHub admin to **Request full resync**, or call `PeopleHubSynchronizer.ResetAsync()`. The reload calls `OnJoinedAsync` for everyone.

**My app starts seeing nulls in a field.**
Its key probably lacks the scope. Ask an admin to add it on **Client apps**; the key stays the same.

**We just got a new scope, but the copy is empty.**
A new scope shows the field on **future** reads and events; it does not resend old data. If the data was already in PeopleHub when the scope was ticked,
**request a full resync once**.

**Changes stopped arriving.**
Look in your log for `PeopleHub sync failed; will retry`:

- a handler error → fix the handler (the sync is waiting on that event);
- `401` or `403` → the key or scope;
- not reachable → the network or PeopleHub.

**Webhook deliveries fail with `401`.**
The secret in `PeopleHub:WebhookSecret` does not match, or a server clock is more than 5 minutes off.

**Changing the webhook secret without downtime.**
Put both secrets in `PeopleHub:WebhookSecrets` during the change, as in the [webhooks FAQ](webhooks-faq.md).

---

## 9. Good to know

**Running several copies of the app (load balancer).**
Each copy polls, and each can receive webhooks. Handled events are recorded in the shared database, but two copies can occasionally handle the same event at
the same moment. One more reason to keep handlers idempotent.

**Without a local store?**
You can leave out `AddPeopleHubLocalSync`: lookups and webhooks still work. Duplicates are then remembered in memory for 2 days only, there is no fallback
when PeopleHub is down, and nothing polls, so a missed webhook is not caught up.

**New event types later.**
An event type your version of the library does not know is ignored and recorded, so upgrading PeopleHub does not break older apps. Fields may be added within
v1, never renamed or removed.

**Where is the key kept?**
In an environment variable or secret store, never in source control. If it leaks, see the
[clients and API keys FAQ](clients-and-api-keys-faq.md#5-everyday-situations).

---

## 10. Quick reference

| You want | Use |
|---|---|
| One person | `GetEmployeeAsync` |
| Is this person active? | `IsActiveAsync` |
| Who approves? | `GetManagerAsync` |
| Everyone | `SearchEmployeesAsync`, or the local copy |
| React to HR changes | Your handler |
| Sync now | `PeopleHubSynchronizer.RunOnceAsync()` |
| Reload everything | `PeopleHubSynchronizer.ResetAsync()`, or an admin's *Request full resync* |
| Existing users | Backfill, **before the first sync** |

| Rule | |
|---|---|
| Handlers | Safe to run twice |
| `OnJoinedAsync` | Create **or refresh** |
| The library | Writes HR columns only |
| App data | Your own tables, keyed by employee code |
| App problems | Your own review list |

See also: [onboarding](../onboarding.md) · [Employees and Changes FAQ](employees-and-changes-faq.md) · [webhooks FAQ](webhooks-faq.md) · [clients and API keys FAQ](clients-and-api-keys-faq.md) ·
[manual people FAQ](manual-people-faq.md) · [admin employees and rehire FAQ](admin-employees-and-rehire-faq.md).
