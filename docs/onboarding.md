# Onboarding an app to PeopleHub in 5 steps

The integration takes about an hour. samples/PeopleHub.SampleApp is a complete working example.
The read endpoints, the change feed, resync and poll intervals are explained in plain English in
[help/employees-and-changes-faq.md](help/employees-and-changes-faq.md); webhooks in [help/webhooks-faq.md](help/webhooks-faq.md); the library, handler,
local copy and backfill in [help/client-library-faq.md](help/client-library-faq.md).

## 1. Get a key (PeopleHub admin)

Admin UI → **Client apps & keys** → *Add a client app*. Tick only the scopes the app needs:

| Scope | Gives |
|---|---|
| `employees:read:basic` | employeeCode, name, email, active, status, source, aliases |
| `employees:read:org` | + department, designation, manager, location; `/departments`, `/manager`, `/reports` |
| `employees:read:dates` | + date of joining, leaving date |
| `employees:read:shift` | + shift (from greytHR) |
| `employees:read:extra` | + extra greytHR fields |
| `webhooks:subscribe` | the app may receive change events by webhook |

Copy the key now (it is shown once). Use a separate app and key per environment (Dev/UAT/Prod hubs).

## 2. Add the packages and configuration

Add the GitHub Packages feed (`https://nuget.pkg.github.com/<org>/index.json`) to the app's `nuget.config`, then add
`PeopleHub.Client` (and `PeopleHub.Client.PostgreSql` if the app uses PostgreSQL).

```json
"PeopleHub": {
  "BaseUrl": "https://peoplehub-uat.example.com/",
  "ApiKey": "<from environment variable or secret store>",
  "WebhookSecret": "<step 5>"
}
```

```csharp
builder.Services.AddPeopleHubClient();
builder.Services.AddPeopleHubHandler<MyEmployeeHandler>();
builder.Services.AddPeopleHubLocalSync(o => o.ConnectionString = builder.Configuration.GetConnectionString("App"));
app.MapPeopleHubWebhooks("/peoplehub/events");
```

Apps on SQL Server or another database implement `IEmployeeStore` and call `AddPeopleHubLocalSync<MyStore>()`.

If the app already has users, set `PeopleHub:LocalSync:Enabled` to `false` until step 4 is done: otherwise the first sync starts as
soon as the app runs, before the backfill, and can create duplicate users.

## 3. Write the handler — the app's own rules live here

Implement `IEmployeeChangedHandler` (or derive from `EmployeeChangedHandler`): `OnJoinedAsync`, `OnUpdatedAsync`, `OnLeftAsync`,
`OnReactivatedAsync`, `OnManagerChangedAsync`, `OnAliasLinkedAsync` (two codes are one person: merge) and `OnAliasUnlinkedAsync`
(that link was a mistake: split, or flag for a person to check).

- Map department/designation to *your* roles, create or deactivate accounts, reassign approvals, compute derived fields.
- Keep app-owned data (roles, cost centres) in your own tables keyed by `EmployeeCode`; the library only writes HR fields.
  The shift is HR data from greytHR: read it from `employee.Shift` (scope `employees:read:shift`) instead of keeping your own.

- App rules ("every active employee needs a shift") go to the app's own review list — never back to PeopleHub.
- Make every method idempotent: delivery is at-least-once (duplicates are filtered, but a retry after a crash can repeat a call).
- `employee.Source` is `Core`, `Excel` or `Manual` (contractors, interns added in PeopleHub), if you treat them differently.

## 4. Backfill existing users (only if the app already has users)

The first sync calls `OnJoinedAsync` for every employee; your handler finds the existing user by `EmployeeCode`. Do this step
**before the first sync**, or users that cannot be found that way are created a second time.

- **Users already carry employee codes (Timesheet):** run the backfill once as a **check**. Export the users *with* their
  `employeeCode` column; each code is looked up (old codes included), then the email and name. In `review.csv`, look at:
  - rows with `accept` = no: nothing matched (a typo, or a person not in greytHR: add them as a manual person), several
    matches, or two users proposed for the same employee;

  - rows whose `proposedCode` differs from `currentCode` (the note says "Differs from the app's code …"): an **old code** of
    someone rehired (the current code is proposed), or a wrong code that the email corrected.

  `apply` writes statements only for rows that change a code, so `apply.sql` is the list of corrections.

- **Users have no employee codes:** run the full match below; it proposes a code per user from email or name.

```bash
dotnet tool install --global PeopleHub.Backfill --add-source https://nuget.pkg.github.com/<org>/index.json
```

```bash
peoplehub-backfill match --users users.csv --out review.csv --url https://peoplehub-uat.example.com/ --api-key <key>
```

`users.csv` columns: `appUserId,email,employeeCode,name,department`. Check every row of `review.csv` (`accept` = yes/no; change
`proposedCode` if you pick another person), then:

```bash
peoplehub-backfill apply --review review.csv --template "UPDATE users SET employee_code = {employeeCode} WHERE id = {appUserId};" --out apply.sql
```

Review `apply.sql` and run it against the app's database. (Or use `BackfillRunner` + `IBackfillTarget` in code, as the sample does.)

## 5. Start syncing and turn on webhooks

1. Start the app: the first sync loads every employee and raises `OnJoinedAsync` per active employee, then polls `/api/v1/changes`
   every 5 minutes (`PeopleHub:LocalSync:PollInterval`).

2. For near-real-time changes, register the webhook once (needs `webhooks:subscribe`):
   `await peopleHubClient.SubscribeWebhookAsync(new Uri("https://myapp.example.com/peoplehub/events"))`, or ask a PeopleHub admin
   (Admin UI → Webhooks). Store the returned secret as `PeopleHub:WebhookSecret` and restart.

3. Verify in the Admin UI: the app's page shows its recent reads; **Webhooks → Delivery log** shows deliveries succeeding.

Polling stays on as a safety net: anything a webhook missed arrives with the next poll, and nothing is applied twice.
