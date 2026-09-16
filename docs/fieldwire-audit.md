# Fieldwire Feature Audit

Complete inventory of Fieldwire functionality mapped to what we replicate, cut, or extend for fire alarm work.

Source: Fieldwire Developer API (`https://developers.fieldwire.com`) — audited September 2025 using API key investigation.

---

## Authentication & API Structure

**Fieldwire implementation:**
- Two-token system: long-lived refresh token (API key) + short-lived JWT access token
- Refresh: `POST https://client-api.super.fieldwire.com/api_keys/jwt` with `{"api_token": "<refresh_token>"}`
- All requests: `Authorization: Bearer <access_token>` + `Fieldwire-Version: 2023-11-30`
- Region-specific base URLs: `client-api.us.fieldwire.com/api/v3`, `client-api.eu.fieldwire.com/api/v3`, `client-api.ca.fieldwire.com/api/v3`
- Cursor-based pagination: `Fieldwire-Per-Page` header (max 1000), `X-Has-More` + `X-Last-Synced-At` response headers, `last_synced_at` query param for next page
- Rate limits: 3,000 req/5 min global; per-endpoint per-minute/hour limits; HTTP 429 on exceed with `retry-after` header

**Our implementation:**
- Same two-token pattern, own JWT secret
- Single regional deployment (US, Fly.io)
- Same pagination strategy (timestamp cursor)
- Rate limiting via Fastify rate-limit plugin

---

## Entity Hierarchy

```
Account
└── Projects
    ├── Users (roles: admin, member, viewer)
    ├── Teams (PM groups / companies)
    ├── Floorplans
    │   └── Sheets
    │       ├── Attachments
    │       │   └── Hyperlinks / Multi-Hyperlinks
    │       ├── Markups (GeoJSON-based)
    │       └── Bubbles (comments/photos on sheet)
    ├── Tasks
    │   ├── Check Items (checklist rows)
    │   ├── Bubbles (comments/photos/files)
    │   ├── Task Relations (dependencies)
    │   └── Task Attributes (custom fields)
    ├── Forms
    │   ├── Form Template
    │   │   └── Form Template Sections
    │   │       └── Form Template Inputs
    │   └── Form Record
    │       └── Form Section Records
    │           └── Form Input Records (values)
    ├── Locations (hierarchical: building → floor → room/zone)
    ├── RFIs
    │   ├── Attachments
    │   └── Markups
    ├── Submittals
    │   └── Attachments
    ├── Specs
    │   └── Spec Sections
    ├── Budget
    │   ├── Line Items
    │   ├── Cost Codes (tier 1 + tier 2)
    │   └── Change Orders
    ├── Reports (templates + generated)
    ├── BIM Files
    └── Data Types (custom field definitions)
```

**We replicate:** Projects, Users/Teams, Floorplans/Sheets, Tasks/CheckItems/Bubbles, Forms, Locations, Reports, Attachments, Webhooks, Custom Fields

**We skip (not relevant to fire alarm):** Specs, Submittals, BIM Files, Budget/Cost tracking (use separate accounting software)

**We extend:** Device Database, NFPA 72 compliance tracking, Panel configuration, AHJ management (see `fire-alarm-specifics.md`)

---

## Projects

**Fieldwire fields:**
- `id`, `name`, `description`
- `account_id`
- `creator_user_id`, `last_editor_user_id`
- `code` (project code)
- `status` (active, archived)
- `address`, `city`, `state`, `zip`, `country`
- `latitude`, `longitude`
- `start_date`, `end_date`
- `created_at`, `updated_at`, `deleted_at`
- `is_sample` (demo project flag)
- Stats: task counts by status, form counts, user counts

**Fieldwire endpoints:**
- `GET /api/v3/projects` — list all
- `POST /api/v3/projects` — create
- `GET /api/v3/projects/:id` — get one
- `PATCH /api/v3/projects/:id` — update
- `DELETE /api/v3/projects/:id` — soft delete
- `GET /api/v3/projects/:id/activity` — audit log
- `POST /api/v3/projects/:id/transfer` — transfer to another account
- `GET /api/v3/projects/:id/statistics` — task/form counts

**Our additions:**
- `building_type` (commercial, industrial, residential)
- `system_type` (conventional, addressable, wireless)
- `ahj_id` (FK to AHJ record)
- `panel_brand`, `panel_model`, `panel_serial`
- `nfpa_zone` (occupancy classification for NFPA 72)
- `contract_type` (new_install, inspection, service, monitoring)
- `inspection_due_date`, `last_inspection_date`
- `certificate_number`

---

## Floorplans & Sheets

**What Fieldwire does:**
- Upload PDF floor plans (multi-page PDFs split into individual sheets)
- Rasterize PDFs to tiles for pan/zoom viewing
- Sheet versioning (can upload new version of same plan)
- Rotate, merge, export sheets
- OCR for text extraction
- Organize into Floorplan Collections (folder groups)
- GeoJSON-based markups drawn on top of sheets

**Upload workflow:**
1. Create a floorplan record
2. Request S3 upload token
3. Upload PDF directly to S3
4. Trigger processing (PDF → tiles)
5. Poll for processing completion
6. Sheets available

**Sheet markup types:**
- Bubble markups (pinned location with task/form link)
- Sheet markups (GeoJSON shapes: polygons, lines, text)
- Attachment hyperlinks
- Multi-hyperlinks (one pin → multiple files)

**Our implementation:**
- PDF upload → Tigris object storage
- PDF.js for client-side rendering (no tiling needed for MVP)
- Server-side tile generation for performance (Phase 2)
- GeoJSON stored in Tigris for markup overlays
- Fire alarm device pins as a special markup type (linked to Device DB records)
- Plan versioning for as-built vs. design drawings

**Fire alarm additions:**
- Device symbol library (NFPA 72 standard symbols: smoke detector, heat detector, pull station, horn/strobe, control panel, duct detector, etc.)
- Device pin = markup point linked to a `Device` record
- Zone boundary overlays (GeoJSON polygons per fire alarm zone)
- Color-coded overlays: green = tested/pass, red = fail/deficiency, yellow = pending

---

## Tasks

**Fieldwire fields:**
- `id`, `project_id`, `name`, `description`
- `creator_user_id`, `last_editor_user_id`, `owner_user_id` (assignee)
- `sequence_id` (human-readable number, auto-increment per project)
- `status`: open, in_progress, ready_for_review, complete
- `priority`: critical, high, normal, low
- `due_date`, `start_date`
- `location_id`, `floorplan_id`
- `position_x`, `position_y` (pin on sheet)
- `task_type` (custom type defined per project)
- `watchers` (user IDs subscribed to updates)
- `tags` (entity tags)
- `custom_fields` (task attributes: text, number, list)

**Task relationships:**
- `check_items`: ordered checklist with `is_complete` boolean
- `bubbles`: comments, photos, files attached to task
- `task_relations`: depends_on, blocked_by relationships
- `attachments`: files directly on task

**Status flow:** open → in_progress → ready_for_review → complete (any → deleted)

**Our implementation - identical status flow, additional task types:**

Fire alarm task types:
- `deficiency` — issue found during inspection (maps to NFPA 72 deficiency log)
- `service_call` — reactive repair
- `installation_item` — punch list item on new install
- `device_swap` — replace a specific device
- `panel_programming` — programming task
- `inspection_item` — routine inspection action

**Fire alarm additions:**
- `device_id` (FK to Device record — task tied to specific device)
- `nfpa_code` (e.g., "10.4.3.1" — NFPA 72 section the deficiency violates)
- `corrective_action` (what was done to resolve)
- `resolution_date`
- `requires_reinspection` boolean
- `ahj_notified` boolean (for life-safety deficiencies)

---

## Task Check Items

**Fieldwire:**
- Ordered list of checklist items per task
- `name`, `is_complete`, `sequence_number`, `assignee_id`, `due_date`
- CRUD endpoints: create, list, update, delete
- Webhook events: created, updated, deleted

**Our use:** Directly replicated. Used for NFPA 72 test procedure steps (e.g., "Activate device", "Verify signal at panel", "Verify notification appliances", "Reset system").

---

## Bubbles (Comments + Media)

**Fieldwire:**
- Polymorphic: attached to task OR sheet position
- Types: comment (text), photo (image), file (PDF/doc)
- `content` (text), `attachment_id` (media)
- `creator_user_id`, `created_at`
- Supports @mentions

**Our implementation:** Identical. Called "Notes" in the UI. Support photo capture from mobile (PWA camera access).

---

## Forms

**Fieldwire form architecture:**
- `FormTemplate` — reusable template (e.g., "Annual Inspection Report")
  - `FormTemplateSection` — grouped set of inputs (e.g., "Smoke Detectors", "Pull Stations")
    - `FormTemplateInput` — individual field (see types below)
- `Form` — instance of a template on a project
  - Required: `form_template_id`, `checksum`, `name`, `creator_user_id`, `owner_user_id`
  - Two-step creation: create record, then call "generate" endpoint
  - `FormSectionRecord` — instance of a template section
    - `FormInputRecord` — captured value for one input

**Fieldwire input types (inferred from API + docs):**
- Short text
- Long text / paragraph
- Number
- Date
- Dropdown / select
- Multi-select
- Checkbox
- Signature
- Photo
- File attachment
- User reference
- Location reference
- Yes/No/N/A (pass/fail pattern)
- Table (repeating rows)

**Form statuses:** draft, submitted, approved, rejected (with transitions API)

**Form permissions:** per-template or per-form user access control

**Our implementation:**
- Exact same template → instance pattern
- Templates pre-loaded for fire alarm (see `fire-alarm-specifics.md`)
- Add: `nfpa_reference` field on each template section (e.g., "NFPA 72 Ch.14")
- Add: PDF auto-generation on form submission (Inspection Report)
- Add: AHJ submission packet generation (multi-form PDF bundle)
- Add: Certificate of Inspection auto-fill from form data

---

## Locations

**Fieldwire:**
- Hierarchical tree (any depth)
- `name`, `parent_location_id`, `project_id`
- Each location has `active_task_count`
- Full path string: "Building A / Floor 3 / Server Room"

**Our fire alarm mapping:**
- Level 1: Building / Structure
- Level 2: Floor / Level
- Level 3: Zone (fire alarm zone boundary)
- Level 4: Room / Area

Locations link to:
- Tasks (issue is in this location)
- Devices (device is installed in this location)
- Form sections (test this zone)
- Sheet markups (overlay on plan)

---

## Attachments & File Storage

**Fieldwire:**
- Files stored in S3 (presigned upload tokens)
- `POST /api/v3/projects/:id/attachments` — create record
- `GET /api/v3/projects/:id/s3_tokens` — get presigned S3 upload URL
- Then PUT file directly to S3 URL
- Then PATCH attachment to mark as uploaded
- Attachment types: image, pdf, video, file

**Our implementation:**
- Tigris object storage (S3-compatible)
- Same presigned URL flow
- Mobile: photo capture → base64 or direct multipart upload
- Auto-thumbnail generation for images
- PDF preview generation

---

## Markups

**Fieldwire:**
- Sheet markups stored as GeoJSON features
- Markup types: polygon, polyline, circle, rectangle, text, arrow, cloud shape
- Color, stroke width, fill properties
- `bubble_markup` — pin with linked task/form
- Markup categories and custom symbols (per account)
- Multi-hyperlink markups (one pin → N files)
- Markup flattening (bake markups into sheet image)

**Our additions:**
- Fire alarm device symbols (NFPA 72 standard icon set as SVG)
- Device pin markup linked to Device DB record
- Test status overlay (color by last test result)
- Zone boundary markup (polygon with zone number/name)

---

## Teams & Users

**Fieldwire:**
- Account-level users with roles: owner, admin, member, viewer
- Project-level roles can differ from account role
- Teams (PM groups) — group users under a company name
- Invite by email (generates token-based signup link)
- Remove user from project or account
- `GET /api/v3/projects/:id/users` — list project users
- `POST /api/v3/accounts/:id/users/invite` — invite
- Batch operations for bulk role changes

**Our roles:**
- `owner` — account admin (company admin)
- `technician` — field tech, can create/edit tasks + forms
- `inspector` — can complete inspection forms, read-only plans
- `project_manager` — full project access, no billing
- `client` — external customer, read-only portal (their jobs only)

---

## Webhooks

**Fieldwire event catalog:**

| Entity | Actions |
|---|---|
| Attachments | created, updated, deleted, restored |
| Entity Tags | created, updated, deleted |
| Entity Taggings | created, deleted |
| Floorplans | created, updated, deleted, merged, rescanned, rotated |
| Forms | created, deleted, updated |
| Hyperlinks | created, updated, deleted, restored |
| Projects | created, updated, deleted, transferred |
| Sheets | created, updated, deleted |
| Sheet Uploads | created |
| Tasks | created, updated, deleted |
| Task Check Items | created, updated, deleted |
| Task Relations | created |

**Payload structure:**
```json
{
  "schema_version": "1",
  "event_category": "project",
  "event_id": "uuid",
  "event_timestamp": 1700000000,
  "account_id": "uuid",
  "project_id": "uuid",
  "user_id": "uuid",
  "event": {
    "entity_type": "task",
    "action": "updated",
    "attributes": {}
  },
  "entity_data": {}
}
```

**Our webhook use:**
- Push notifications to mobile (PWA) when task assigned
- Email digest on overdue inspection dates
- Slack integration (Phase 2)
- Client portal real-time updates

---

## Reports

**Fieldwire:**
- Report templates (per project)
- Generate PDF reports from template
- Email PDF to specified recipients
- Export to CSV
- Filter by date range, user, status, task type

**Our reports:**
- Inspection Report (auto-generated from completed inspection form)
- Deficiency Log (all open deficiencies on a job)
- AHJ Submittal Packet (inspection report + certificate + device list)
- Service History Report (all service calls on a property over time)
- Compliance Status Report (what's due, what's overdue)
- Device Inventory Report

---

## Custom Fields (Task Type Attributes)

**Fieldwire types:**
- `short_text` — `text_value: "string"`
- `number` — `number_value: 123`
- `list` — `uuid_value: "option-uuid"` (options are separate records)

**Endpoints:**
- `GET /api/v3/projects/:id/task_type_attributes` — attribute definitions
- `GET /api/v3/projects/:id/tasks/:task_id/task_attributes` — values for a task
- `POST /api/v3/projects/:id/tasks/:task_id/task_attributes` — set value
- `PATCH /api/v3/projects/:id/tasks/:task_id/task_attributes/:id` — update

**Our custom fields:**
- Same three types
- Extended with: `date`, `yes_no_na`, `device_reference`, `location_reference`
- Pre-built field sets per project type (inspection checklist, installation punch list)

---

## RFIs (Request for Information)

**Fieldwire:** RFI creation, attachments, markups, transition workflow, PDF export, watchers

**Our use:** Repurposed as "Deficiency Notice" — formal written notification to customer of a life-safety deficiency. Same workflow, fire-alarm-specific language.

---

## What Fieldwire Has That We Skip

| Feature | Reason |
|---|---|
| Submittals | Construction-specific, not applicable |
| Specifications | Not applicable |
| BIM Files | Overkill for fire alarm; not using 3D models |
| Budget / Cost Codes | Use separate accounting software (QuickBooks, etc.) |
| Change Orders | Handled in accounting |
| Weather Conditions | Not relevant |
| Template Checklists (project-level) | Replaced by our pre-built form templates |
