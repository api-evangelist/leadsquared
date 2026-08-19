---
name: Look up a LeadSquared lead by identifier or search term
description: >-
  Resolve a person to a LeadSquared lead record — by ProspectID when you have one,
  by email/phone/name when you do not — without writing anything to the CRM.
api: openapi/leadsquared-leads-api-openapi.yml
operations:
  - getLeadById
  - quickSearchLeads
generated: '2026-08-13'
method: generated
source: >-
  openapi/leadsquared-leads-api-openapi.yml,
  conventions/leadsquared-conventions.yml,
  errors/leadsquared-problem-types.yml
---

# Look up a lead (read-only)

Both operations in this skill are reads. Nothing here mutates CRM data, which makes it the
safe entry point for an agent that has just been handed LeadSquared credentials.

## Setup

- Base URL `https://<regional-host>/v2` — resolve the tenant's host first (see
  `conventions/leadsquared-conventions.yml`); the wrong host returns 401 and names the
  right one in the error message.
- Send `x-LSQ-AccessKey` and `x-LSQ-SecretKey` headers. HTTPS only.

## When you have a ProspectID — `getLeadById`

`GET /LeadManagement.svc/Leads.GetById?id=<ProspectID>`

`ProspectID` is a GUID, e.g. `cc3fef18-960b-4a60-98c0-af2ab61cee80`. LeadSquared ids carry
no type prefix, so a GUID on its own does not tell you whether it is a lead, an opportunity
or a task — track which entity an id came from.

The response is a list of leads, each carrying a `LeadPropertyList` of
`{Attribute, Value, Fields}` entries rather than a flat typed object. Read fields by
attribute name; do not assume ordering.

This operation documents `401` and `429` responses in addition to `200`.

## When you only have an email, phone or name — `quickSearchLeads`

`GET /LeadManagement.svc/Leads.GetByQuickSearch?key=<term>`

`key` matches across the searchable lead fields (email, phone, name, company). Quick search
returns a bounded set — treat it as a lookup, not as an export.

Use it to:

- resolve a person to a `ProspectID` before any write flow;
- check whether a lead already exists before creating one, since `createLead` has no
  idempotency key and a blind retry can duplicate a record.

## When you need many leads, do not loop quick search

Use the date-range read (`https://apidocs.leadsquared.com/get-leads-by-date-range/`), which
takes POST body objects rather than query parameters:

```json
{
  "Parameter": {"FromDate": "2026-08-01 00:00:00", "ToDate": "2026-08-13 00:00:00"},
  "Columns":   {"Include_CSV": "ProspectID,FirstName,LastName,EmailAddress,LeadType"},
  "Paging":    {"PageIndex": 1, "PageSize": 1000},
  "Sorting":   {"ColumnName": "ProspectAutoId", "Direction": "1"}
}
```

- `PageSize` maximum is 5000.
- `Direction` is `"1"` for newest first, `"0"` for oldest first.
- The response carries `RecordCount` — use it to decide how many pages to walk, and stop.
- Dates are UTC, `YYYY-MM-DD HH:MM:SS`.

## Rate limiting

Reads count against the same plan quota as writes and there is **no** runtime signal — no
`RateLimit-*` and no `Retry-After`. Pace at the published 5-second throttle (10 calls per 5s
on Pro, 20 on Super; bulk endpoints are half that) and back off on any error rather than
retrying immediately.

## Errors

Failures return `{"Status": "Error", "ExceptionType": "...", "ExceptionMessage": "..."}`
with status 400, 401, 404 or 500. A 401 on a read is far more often the wrong regional host
than a bad key — read the message before rotating credentials.
