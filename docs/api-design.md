# API Design

Base URL: `https://api.1stfpalarm.com/api/v1`

Headers on every request:
```
Authorization: Bearer <access_token>
X-Api-Version: 2024-01-01
Content-Type: application/json
```

Pagination headers:
```
# Request
Fieldwire-Per-Page: 100   (max 1000)

# Response
X-Count: 47
X-Has-More: true
X-Last-Synced-At: 2024-01-15T12:00:00Z
```

Next page: `GET /endpoint?last_synced_at=<X-Last-Synced-At value>`

---

## Auth

```
POST   /auth/login                    { email, password } → { access_token, refresh_token }
POST   /auth/refresh                  { refresh_token } → { access_token }
POST   /auth/logout
GET    /auth/me                       → current user
POST   /auth/api-keys                 → create API key
DELETE /auth/api-keys/:id
```

---

## Account

```
GET    /account                       → account details
PATCH  /account                       → update name, logo, timezone
GET    /account/users                 → list all users in account
POST   /account/users/invite          { email, role } → invite user
PATCH  /account/users/:id             → update role
DELETE /account/users/:id             → remove user
```

---

## Projects

```
GET    /projects                      → list (filter: status, compliance_status)
POST   /projects                      → create
GET    /projects/:id                  → get
PATCH  /projects/:id                  → update
DELETE /projects/:id                  → soft delete
GET    /projects/:id/activity         → audit log
GET    /projects/:id/statistics       → task/form/device counts
GET    /projects/:id/users            → project users
POST   /projects/:id/users            → add user to project
PATCH  /projects/:id/users/:user_id   → update role on project
DELETE /projects/:id/users/:user_id   → remove from project
```

---

## AHJs

```
GET    /ahjs                          → list (account-scoped)
POST   /ahjs                          → create
GET    /ahjs/:id
PATCH  /ahjs/:id
DELETE /ahjs/:id
```

---

## Locations

```
GET    /projects/:id/locations        → flat list with path
POST   /projects/:id/locations
GET    /projects/:id/locations/:lid
PATCH  /projects/:id/locations/:lid
DELETE /projects/:id/locations/:lid
```

---

## Floorplans & Sheets

```
GET    /projects/:id/floorplans
POST   /projects/:id/floorplans
GET    /projects/:id/floorplans/:fid
PATCH  /projects/:id/floorplans/:fid
DELETE /projects/:id/floorplans/:fid

GET    /projects/:id/floorplans/:fid/sheets
GET    /projects/:id/sheets           → all sheets for project
POST   /projects/:id/sheets/upload-url  → get presigned S3 URL
POST   /projects/:id/sheets/:sid/process  → trigger processing
GET    /projects/:id/sheets/:sid
PATCH  /projects/:id/sheets/:sid
DELETE /projects/:id/sheets/:sid

GET    /projects/:id/floorplan-collections
POST   /projects/:id/floorplan-collections
PATCH  /projects/:id/floorplan-collections/:cid
DELETE /projects/:id/floorplan-collections/:cid
```

---

## Markups

```
GET    /projects/:id/sheets/:sid/markups
POST   /projects/:id/sheets/:sid/markups
GET    /projects/:id/sheets/:sid/markups/:mid
PATCH  /projects/:id/sheets/:sid/markups/:mid
DELETE /projects/:id/sheets/:sid/markups/:mid
```

---

## Tasks

```
GET    /projects/:id/tasks            → list (filter: status, priority, task_type, owner_user_id, due_before)
POST   /projects/:id/tasks
GET    /projects/:id/tasks/:tid
PATCH  /projects/:id/tasks/:tid
DELETE /projects/:id/tasks/:tid
POST   /projects/:id/tasks/:tid/restore

GET    /projects/:id/tasks/count      → counts by status

GET    /projects/:id/tasks/:tid/check-items
POST   /projects/:id/tasks/:tid/check-items
PATCH  /projects/:id/tasks/:tid/check-items/:cid
DELETE /projects/:id/tasks/:tid/check-items/:cid

GET    /projects/:id/tasks/:tid/relations
POST   /projects/:id/tasks/:tid/relations
DELETE /projects/:id/tasks/:tid/relations/:rid

GET    /projects/:id/tasks/:tid/task-attributes
POST   /projects/:id/tasks/:tid/task-attributes
PATCH  /projects/:id/tasks/:tid/task-attributes/:aid

GET    /projects/:id/task-type-attributes
POST   /projects/:id/task-type-attributes
PATCH  /projects/:id/task-type-attributes/:id
DELETE /projects/:id/task-type-attributes/:id
```

---

## Bubbles (Notes + Media)

```
GET    /projects/:id/tasks/:tid/bubbles
POST   /projects/:id/tasks/:tid/bubbles
PATCH  /projects/:id/tasks/:tid/bubbles/:bid
DELETE /projects/:id/tasks/:tid/bubbles/:bid

GET    /projects/:id/sheets/:sid/bubbles
POST   /projects/:id/sheets/:sid/bubbles
```

---

## Attachments

```
GET    /projects/:id/attachments
POST   /projects/:id/attachments/upload-url
PATCH  /projects/:id/attachments/:aid      → mark as uploaded
DELETE /projects/:id/attachments/:aid
```

---

## Forms

```
GET    /form-templates                → account templates
POST   /form-templates
GET    /form-templates/:id
PATCH  /form-templates/:id
DELETE /form-templates/:id

GET    /form-templates/:id/sections
POST   /form-templates/:id/sections
PATCH  /form-templates/:id/sections/:sid
DELETE /form-templates/:id/sections/:sid

GET    /form-templates/:id/sections/:sid/inputs
POST   /form-templates/:id/sections/:sid/inputs
PATCH  /form-templates/:id/sections/:sid/inputs/:iid
DELETE /form-templates/:id/sections/:sid/inputs/:iid

GET    /projects/:id/forms
POST   /projects/:id/forms            → create form from template
GET    /projects/:id/forms/:fid
PATCH  /projects/:id/forms/:fid
DELETE /projects/:id/forms/:fid
POST   /projects/:id/forms/:fid/submit   → trigger PDF gen, status→submitted
POST   /projects/:id/forms/:fid/approve
POST   /projects/:id/forms/:fid/reject

GET    /projects/:id/forms/:fid/section-records
POST   /projects/:id/forms/:fid/section-records   → create (for repeating sections)

GET    /projects/:id/forms/:fid/section-records/:srid/input-records
POST   /projects/:id/forms/:fid/section-records/:srid/input-records
PATCH  /projects/:id/forms/:fid/section-records/:srid/input-records/:irid
```

---

## Devices

```
GET    /projects/:id/devices          → list (filter: device_type, zone_number, last_test_result, location_id)
POST   /projects/:id/devices
GET    /projects/:id/devices/:did
PATCH  /projects/:id/devices/:did
DELETE /projects/:id/devices/:did
POST   /projects/:id/devices/bulk-import   → CSV import

# Test record
POST   /projects/:id/devices/:did/test     { result, date, tech_id, sensitivity? }
GET    /projects/:id/devices/:did/history  → service log
```

---

## Panels

```
GET    /projects/:id/panel
POST   /projects/:id/panel
PATCH  /projects/:id/panel
```

---

## Compliance

```
GET    /compliance                    → account-wide compliance dashboard data
GET    /compliance/due                → jobs due within :days (query param)
GET    /compliance/overdue            → all overdue jobs
GET    /projects/:id/compliance       → single project compliance detail
POST   /projects/:id/compliance/generate-tasks  → auto-create inspection tasks
```

---

## Reports

```
GET    /projects/:id/reports          → list generated PDFs
POST   /projects/:id/reports/inspection   { form_id } → generate inspection report PDF
POST   /projects/:id/reports/deficiency-log  → generate open deficiency PDF
POST   /projects/:id/reports/device-inventory
POST   /projects/:id/reports/ahj-packet     { form_ids[] } → bundled AHJ submittal
GET    /projects/:id/reports/:rid/download  → presigned download URL
POST   /projects/:id/reports/:rid/email     { recipients[] }
```

---

## Webhooks

```
GET    /webhooks
POST   /webhooks                      { url, secret, events[] }
GET    /webhooks/:id
PATCH  /webhooks/:id
DELETE /webhooks/:id
POST   /webhooks/:id/test             → sends test payload
```

---

## Notifications

```
GET    /notifications                 → user's notifications (unread first)
PATCH  /notifications/:id/read
POST   /notifications/read-all
```

---

## QR Code Scan

Public endpoint - no auth required for initial info, auth required for full record.
```
GET    /scan/:qr_code_id            → device summary (public: property name, device type, last inspection date)
GET    /scan/:qr_code_id/full       → full device record (requires auth)
POST   /scan/:qr_code_id/log        → log a scan event (anonymous OK, used for audit trail)

POST   /projects/:id/devices/qr-batch-print  { device_ids[] } → generate QR label PDF for batch printing
```

---

## Closeout Packages

```
GET    /projects/:id/closeout-packages
POST   /projects/:id/closeout-packages/generate   → compile current state into PDF bundle
GET    /projects/:id/closeout-packages/:cid/status → checklist of what's complete vs. missing
GET    /projects/:id/closeout-packages/:cid/download
POST   /projects/:id/closeout-packages/:cid/send  { recipients[] }
POST   /projects/:id/closeout-packages/:cid/trigger-billing  → push billing milestone to Sage
```

---

## Deficiency-to-Quote Pipeline

```
GET    /deficiencies                → account-wide aged/unquoted deficiency dashboard
GET    /projects/:id/deficiencies   → project-level (filter: unquoted, by_severity, aged_days)
POST   /projects/:id/deficiencies/:task_id/create-quote   → push to ServiceTrade as service opportunity
GET    /projects/:id/deficiencies/:task_id/quote          → get linked quote status
PATCH  /projects/:id/deficiencies/:task_id/quote          → update estimated_value, repair_scope
```

---

## ServiceTrade Integration

```
GET    /integrations/servicetrade/status
POST   /integrations/servicetrade/configure   { api_key, company_id }
POST   /integrations/servicetrade/sync        → manual sync trigger
GET    /integrations/servicetrade/sync-log    → recent sync history

POST   /integrations/servicetrade/push-deficiency/:task_id  → push one deficiency as ST quote
GET    /integrations/servicetrade/jobs        → list ST jobs (for linking to projects)
POST   /projects/:id/link-servicetrade-job    { st_job_id }
```

---

## Client Portal

Separate auth scope for client users. Same API, filtered to their `account_id` and read-only.

```
GET    /portal/properties             → their buildings
GET    /portal/properties/:id         → compliance status, last inspection, open deficiencies
GET    /portal/properties/:id/reports → downloadable PDFs
GET    /portal/properties/:id/devices → device inventory (read-only)
GET    /portal/properties/:id/deficiencies → open issues
```

---

## Error Responses

```json
{
  "error": {
    "code": "not_found",
    "message": "Project not found",
    "status": 404
  }
}
```

Common codes: `unauthorized`, `forbidden`, `not_found`, `validation_error`, `rate_limit_exceeded`, `internal_error`

---

## Versioning

API version in header: `X-Api-Version: 2024-01-01`

Breaking changes get a new date version. Non-breaking additions are backwards compatible. Deprecated endpoints serve until 90 days after announcement.
