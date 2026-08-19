---
name: Capture a lead and log its first activity in LeadSquared
description: >-
  Create a lead in LeadSquared and post the activity that explains where it came
  from, using the v2 REST API with accessKey/secretKey authentication against the
  correct regional host.
api: openapi/leadsquared-leads-api-openapi.yml, openapi/leadsquared-activities-api-openapi.yml
operations:
  - createLead
  - postActivity
generated: '2026-08-13'
method: generated
source: >-
  openapi/leadsquared-leads-api-openapi.yml,
  openapi/leadsquared-activities-api-openapi.yml,
  conventions/leadsquared-conventions.yml,
  errors/leadsquared-problem-types.yml
---

# Capture a lead and log its first activity

## Before you call anything

1. **Resolve the host.** The base URL is tenant-specific: `https://<host>/v2`. Do not
   assume `api.leadsquared.com` — that is the Singapore host. Read the tenant's host
   from My Account > Settings > API and Webhooks, or from the User Authentication API.
   Calling the wrong host returns **401**, and the correct host is named in the error
   message. Known hosts: `api.leadsquared.com` (SGP), `api-us11`, `api-in21` (Mumbai),
   `api-in22` (Hyderabad), `api-me61`, `api-ir31`, `api-ca12`.
2. **Authenticate with headers, not the query string.** Send `x-LSQ-AccessKey` and
   `x-LSQ-SecretKey`. The query-string form (`?accessKey=…&secretKey=…`) works but puts
   credentials in URLs and logs. HTTPS is mandatory; plain HTTP fails.
3. **Know whose keys these are.** Keys belong to an individual admin user. If that user
   is deactivated the integration stops working. Do not bind an agent to a personal key.

## Step 1 — create the lead (`createLead`)

`POST /LeadManagement.svc/Lead.Create`

The body is an **array of `{Attribute, Value}` pairs**, not a typed object. At least one
unique field — `EmailAddress` or `Phone` — must be present.

```json
[
  {"Attribute": "EmailAddress", "Value": "joe.pesci@example.com"},
  {"Attribute": "FirstName",    "Value": "Joe"},
  {"Attribute": "LastName",     "Value": "Pesci"},
  {"Attribute": "Phone",        "Value": "8888888888"}
]
```

A 200 returns `{"Status": "Success", "Message": {"Id": "<ProspectID GUID>"}}`. Keep that
`Id` — step 2 needs it.

**Custom fields.** Attribute names are tenant-defined. If you are not certain a field
exists, resolve the schema first at
`https://apidocs.leadsquared.com/get-lead-field-properties/` rather than guessing; an
unknown attribute is a 400.

**This call is not idempotent.** There is no idempotency key anywhere in the LeadSquared
API. If you retry after a timeout you may create a duplicate lead. Two options:

- Prefer `Lead.CreateOrUpdate` (`https://apidocs.leadsquared.com/create-or-update/`) or
  `Lead.Capture` with a `SearchBy` field when the flow tolerates upsert semantics — that
  is LeadSquared's only duplicate-control mechanism.
- Otherwise, before retrying, check with `quickSearchLeads` (below) whether the lead
  already landed.

## Step 2 — post the source activity (`postActivity`)

`POST /ProspectActivity.svc/Create`

```json
{
  "RelatedProspectId": "<ProspectID from step 1>",
  "ActivityEvent": <activity type code>,
  "ActivityNote": "Captured from the pricing page form",
  "ActivityDateTime": "2026-08-13 14:05:00"
}
```

`ActivityEvent` is a **tenant-defined numeric activity type**. Resolve it before calling —
`https://apidocs.leadsquared.com/get-list-of-activity-types/` — and cache it. Never
hard-code an activity code you have not looked up in the target tenant.

`ActivityDateTime` is UTC, `YYYY-MM-DD HH:MM:SS`.

**Posting an activity is not idempotent and has no upsert form.** A retry creates a second
activity. Only retry when you have confirmed the first call did not succeed.

## Step 3 — verify (`quickSearchLeads`)

`GET /LeadManagement.svc/Leads.GetByQuickSearch?key=<email or phone>`

Use this to confirm the lead exists before retrying step 1, and to reconcile after a
timeout. For anything larger than a spot check, use the date-range read with the `Paging`
object instead — `Paging {PageIndex, PageSize}` with `PageSize` up to 5000, `Sorting
{ColumnName, Direction}` and `Columns {Include_CSV}`, and read `RecordCount` from the
response. Those are POST body objects, not query parameters.

You can also fetch one lead directly with `getLeadById`
(`GET /LeadManagement.svc/Leads.GetById?id=<ProspectID>`).

## Handling failure

Every failure returns the same envelope — this is not RFC 9457:

```json
{"Status": "Error", "ExceptionType": "MXMandatoryAttributeMissingException",
 "ExceptionMessage": "'leadId' is mandatory for updating lead"}
```

Branch on the HTTP status first, then read `ExceptionMessage`. LeadSquared publishes no
enumerated list of `ExceptionType` values, so do not build exhaustive type-based
branching — log the type and fall through.

| Status | Meaning | What to do |
|---|---|---|
| 400 | Body does not match the API spec | Check `Content-Type: application/json`, JSON validity, attribute names |
| 401 | Invalid credentials **or wrong regional host** | Re-check keys; read the host named in the message |
| 404 | Operation path wrong | Check the service path and operation name |
| 500 | Server-side failure | Read `ExceptionMessage`, back off, retry with duplicate care |

## Rate limits — plan for them, you cannot observe them

LeadSquared returns **no** `RateLimit-*`, `X-RateLimit-*` or `Retry-After` headers. You
cannot read remaining quota; you can only fail and back off. Published limits:

- **Pro:** 10,000 calls/day (+1,000/user/day, cap 250,000); 10 calls per 5s standard,
  5 per 5s bulk.
- **Super:** 100,000 calls/day (+1,000/user/day, cap 1,000,000); 20 calls per 5s standard,
  10 per 5s bulk.

Keep a client-side token bucket at the 5-second throttle. For user-facing capture paths,
use the **Async API** (`https://asyncapi.leadsquared.com/`) instead: it queues, retries up
to 10 times, and returns a `RequestId` you poll on the matching status endpoint. It needs
an `x-api-key` header in addition to the key pair.

## Do not

- Do not write test data through this flow. LeadSquared has no sandbox tenant and no test
  keys — every call above writes to production CRM data.
- Do not invent attribute names, activity codes or opportunity type ids. Resolve them from
  the tenant's metadata endpoints.
