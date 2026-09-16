# Data Model

Tigris is MongoDB-compatible. All collections stored in Tigris. Object storage (plans, photos, PDFs) also uses Tigris S3-compatible storage.

---

## Collections

### `accounts`
```typescript
{
  _id: ObjectId,
  name: string,                    // company name
  slug: string,                    // url-safe, unique
  plan: "internal" | "client" | "contractor",
  owner_user_id: ObjectId,
  logo_url: string,                // Tigris object storage URL
  timezone: string,
  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `users`
```typescript
{
  _id: ObjectId,
  account_id: ObjectId,
  email: string,                   // unique
  name: string,
  phone: string,
  role: "owner" | "project_manager" | "technician" | "inspector" | "client",
  license_number: string,          // fire alarm tech license
  license_state: string,
  avatar_url: string,
  is_active: boolean,
  refresh_token_hash: string,      // hashed API key
  last_login_at: Date,
  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `projects`
```typescript
{
  _id: ObjectId,
  account_id: ObjectId,
  name: string,
  description: string,
  code: string,                    // human-readable project code

  // Address
  address: string,
  city: string,
  state: string,
  zip: string,
  latitude: number,
  longitude: number,

  // People
  creator_user_id: ObjectId,
  last_editor_user_id: ObjectId,
  project_manager_id: ObjectId,
  client_account_id: ObjectId,     // if client has portal access

  // Status
  status: "active" | "archived" | "complete",
  contract_type: "new_install" | "inspection" | "service" | "monitoring",

  // Fire alarm specifics
  building_type: "commercial" | "industrial" | "residential" | "high_rise" | "healthcare" | "educational",
  system_type: "conventional" | "addressable" | "wireless" | "hybrid",
  occupancy_classification: string, // per NFPA 72 / IBC
  ahj_id: ObjectId,
  panel_id: ObjectId,

  // Compliance
  last_annual_inspection_date: Date | null,
  next_annual_due_date: Date | null,
  last_quarterly_inspection_date: Date | null,
  next_quarterly_due_date: Date | null,
  compliance_status: "compliant" | "due_soon" | "overdue" | "deficient",
  certificate_number: string,
  certificate_issued_date: Date | null,
  certificate_expiry_date: Date | null,
  ahj_notification_required: boolean,

  // Counts (denormalized for dashboard)
  open_task_count: number,
  critical_deficiency_count: number,
  device_count: number,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `project_users`
Junction: which users have access to which projects and at what role.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  user_id: ObjectId,
  role: "admin" | "member" | "viewer",
  created_at: Date,
}
```

### `locations`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  parent_location_id: ObjectId | null,
  name: string,
  level: number,                   // 0=building, 1=floor, 2=zone, 3=room
  path: string,                    // "Building A / Floor 2 / Zone 3" (materialized path)
  zone_number: string,             // fire alarm zone if level=2
  active_task_count: number,       // denormalized
  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `floorplans`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  name: string,
  description: string,
  collection_id: ObjectId | null,  // folder grouping
  creator_user_id: ObjectId,
  order: number,

  // Processing status
  status: "pending" | "processing" | "ready" | "error",

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `sheets`
One sheet per page of a floorplan PDF. Each floorplan has N sheets.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  floorplan_id: ObjectId,
  name: string,
  version: number,
  order: number,

  // File
  file_key: string,                // Tigris object storage key (original PDF page)
  thumbnail_key: string,           // Tigris key (PNG thumbnail)
  width_px: number,
  height_px: number,

  // Sheet type
  sheet_type: "fire_alarm" | "electrical" | "architectural" | "as_built" | "other",

  location_id: ObjectId | null,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `floorplan_collections`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  name: string,
  parent_collection_id: ObjectId | null,
  order: number,
  created_at: Date,
  updated_at: Date,
}
```

### `markups`
GeoJSON features drawn on a sheet.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  sheet_id: ObjectId,
  creator_user_id: ObjectId,

  markup_type: "device_pin" | "zone_boundary" | "annotation" | "shape" | "text" | "arrow" | "cloud",

  // GeoJSON geometry (coordinates are % of sheet width/height, 0-1 scale)
  geometry: {
    type: "Point" | "LineString" | "Polygon",
    coordinates: number[] | number[][] | number[][][]
  },

  // Style
  color: string,
  stroke_width: number,
  fill_opacity: number,
  label: string,

  // Links
  task_id: ObjectId | null,
  device_id: ObjectId | null,        // for device_pin type
  form_id: ObjectId | null,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `tasks`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  sequence_id: number,             // auto-increment per project (human-readable: T-0042)
  name: string,
  description: string,

  // Assignment
  creator_user_id: ObjectId,
  last_editor_user_id: ObjectId,
  owner_user_id: ObjectId | null,  // assignee
  watcher_ids: ObjectId[],

  // Status
  status: "open" | "in_progress" | "ready_for_review" | "complete" | "deleted",
  priority: "critical" | "high" | "normal" | "low",

  // Type
  task_type: "deficiency" | "service_call" | "installation_item" | "device_swap" | "panel_programming" | "inspection_item" | "general",

  // Schedule
  due_date: Date | null,
  start_date: Date | null,
  completed_at: Date | null,

  // Location
  location_id: ObjectId | null,
  sheet_id: ObjectId | null,
  position_x: number | null,       // 0-1 fraction of sheet width
  position_y: number | null,

  // Fire alarm specifics
  device_id: ObjectId | null,
  nfpa_code: string,               // e.g., "10.4.3.1"
  corrective_action: string,
  resolution_date: Date | null,
  requires_reinspection: boolean,
  ahj_notified: boolean,
  deficiency_severity: "critical" | "major" | "minor" | null,

  // Tags
  tag_ids: ObjectId[],

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `task_check_items`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  task_id: ObjectId,
  name: string,
  is_complete: boolean,
  sequence_number: number,
  assignee_id: ObjectId | null,
  due_date: Date | null,
  completed_at: Date | null,
  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `bubbles`
Comments, photos, files on tasks or sheets.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,

  // Parent (one of these is set)
  task_id: ObjectId | null,
  sheet_id: ObjectId | null,
  position_x: number | null,
  position_y: number | null,

  creator_user_id: ObjectId,
  content: string,                 // text content
  attachment_id: ObjectId | null,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `attachments`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  creator_user_id: ObjectId,

  file_key: string,                // Tigris object storage key
  file_name: string,
  file_size: number,               // bytes
  content_type: string,            // MIME type
  thumbnail_key: string | null,

  // Status
  upload_status: "pending" | "uploaded" | "failed",

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `devices`
The fire alarm device database. One record per physical device.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  location_id: ObjectId | null,
  sheet_id: ObjectId | null,
  position_x: number | null,
  position_y: number | null,

  // Identity
  device_type: string,             // see fire-alarm-specifics.md for full enum
  zone_number: string,
  loop_number: number | null,      // addressable systems
  address: number | null,          // addressable device address
  label: string,                   // panel label (e.g., "SD-301")

  // Hardware
  manufacturer: string,
  model_number: string,
  serial_number: string,
  install_date: Date | null,
  manufacture_date: Date | null,
  warranty_expiry: Date | null,

  // Operational status
  operational_status: "active" | "disabled" | "trouble" | "supervisory" | "removed",

  // Test tracking
  last_test_date: Date | null,
  last_test_result: "pass" | "fail" | "not_tested",
  last_test_technician_id: ObjectId | null,
  sensitivity: number | null,      // smoke detectors: %/ft
  sensitivity_last_checked: Date | null,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `panels`
Fire alarm control panel per project.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,

  manufacturer: string,
  model: string,
  firmware_version: string,
  serial_number: string,
  install_date: Date | null,
  last_programming_date: Date | null,
  last_programming_tech_id: ObjectId | null,

  zone_count: number,
  slc_loop_count: number,

  battery_manufacturer: string,
  battery_model: string,
  battery_ah: number,
  battery_install_date: Date | null,
  battery_test_date: Date | null,
  battery_test_result: "pass" | "fail" | "marginal" | null,

  monitoring_company: string,
  monitoring_account_number: string,
  monitoring_phone_primary: string,
  monitoring_phone_backup: string,
  central_station_receiver: string,

  zones: Array<{
    zone_id: string,
    zone_number: string,
    zone_label: string,
    zone_type: "alarm" | "supervisory" | "trouble" | "monitor",
    device_count: number,
    circuit_style: "A" | "B",
  }>,

  created_at: Date,
  updated_at: Date,
}
```

### `ahjs`
Authority Having Jurisdiction records.
```typescript
{
  _id: ObjectId,
  account_id: ObjectId,
  name: string,
  jurisdiction_type: "city" | "county" | "state" | "federal",
  state: string,
  county: string,
  city: string,

  contact_name: string,
  contact_title: string,
  contact_phone: string,
  contact_email: string,
  address: string,

  requires_annual_report: boolean,
  report_submission_method: "mail" | "email" | "portal" | "in_person",
  report_portal_url: string,
  report_due_date_rule: string,
  requires_permit_renewal: boolean,
  permit_renewal_interval_months: number,
  requires_witness_testing: boolean,
  special_requirements: string,

  created_at: Date,
  updated_at: Date,
}
```

### `form_templates`
```typescript
{
  _id: ObjectId,
  account_id: ObjectId,
  name: string,
  description: string,
  version: number,
  checksum: string,                // hash of template structure (version control)
  is_system_template: boolean,     // true = 1st FP built-in, false = account custom
  is_active: boolean,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `form_template_sections`
```typescript
{
  _id: ObjectId,
  form_template_id: ObjectId,
  name: string,
  description: string,
  order: number,
  nfpa_reference: string,          // e.g., "NFPA 72 14.4.3"
  is_repeating: boolean,           // true = repeating table (device-by-device rows)
  created_at: Date,
  updated_at: Date,
}
```

### `form_template_inputs`
```typescript
{
  _id: ObjectId,
  form_template_id: ObjectId,
  form_template_section_id: ObjectId,
  name: string,
  label: string,
  order: number,
  input_type: "short_text" | "long_text" | "number" | "date" | "dropdown" | "multiselect" | "checkbox" | "signature" | "photo" | "file" | "yes_no_na" | "device_reference" | "user_reference" | "location_reference",
  is_required: boolean,
  options: string[],               // for dropdown / multiselect
  default_value: string,
  help_text: string,
  created_at: Date,
  updated_at: Date,
}
```

### `forms`
Instance of a template on a project.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  form_template_id: ObjectId,
  form_template_checksum: string,
  name: string,
  description: string,

  creator_user_id: ObjectId,
  last_editor_user_id: ObjectId,
  owner_user_id: ObjectId,         // assigned inspector/technician

  status: "draft" | "submitted" | "approved" | "rejected",

  submitted_at: Date | null,
  approved_at: Date | null,
  approved_by_user_id: ObjectId | null,

  // Generated outputs
  pdf_key: string | null,          // Tigris key for generated PDF

  // Links
  inspection_date: Date | null,
  location_id: ObjectId | null,

  created_at: Date,
  updated_at: Date,
  deleted_at: Date | null,
}
```

### `form_section_records`
```typescript
{
  _id: ObjectId,
  form_id: ObjectId,
  form_template_section_id: ObjectId,
  iteration: number,               // for repeating sections, which row (0-indexed)
  created_at: Date,
  updated_at: Date,
}
```

### `form_input_records`
```typescript
{
  _id: ObjectId,
  form_id: ObjectId,
  form_section_record_id: ObjectId,
  form_template_input_id: ObjectId,

  // Value (only one is set depending on input_type)
  text_value: string | null,
  number_value: number | null,
  date_value: Date | null,
  boolean_value: boolean | null,
  option_values: string[] | null,  // selected option(s)
  attachment_id: ObjectId | null,  // for photo/file inputs
  reference_id: ObjectId | null,   // device, user, or location reference

  creator_user_id: ObjectId,
  created_at: Date,
  updated_at: Date,
}
```

### `task_type_attributes`
Custom field definitions per project (task type + field name).
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  task_type: string,
  name: string,
  field_type: "short_text" | "number" | "list" | "date" | "yes_no_na",
  options: Array<{ id: string, label: string }>,  // for list type
  order: number,
  created_at: Date,
  updated_at: Date,
}
```

### `task_attributes`
Values of custom fields per task instance.
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  task_id: ObjectId,
  task_type_attribute_id: ObjectId,
  text_value: string | null,
  number_value: number | null,
  option_id: string | null,        // for list type
  creator_user_id: ObjectId,
  created_at: Date,
  updated_at: Date,
}
```

### `task_relations`
```typescript
{
  _id: ObjectId,
  project_id: ObjectId,
  task_id: ObjectId,
  related_task_id: ObjectId,
  relation_type: "depends_on" | "blocked_by" | "related_to",
  created_at: Date,
}
```

### `webhooks`
```typescript
{
  _id: ObjectId,
  account_id: ObjectId,
  url: string,
  secret: string,                  // HMAC signing secret
  events: string[],                // e.g., ["task.created", "form.submitted"]
  is_active: boolean,
  created_at: Date,
  updated_at: Date,
}
```

### `entity_tags`
```typescript
{
  _id: ObjectId,
  account_id: ObjectId,
  name: string,
  color: string,
  created_at: Date,
  updated_at: Date,
}
```

### `entity_taggings`
```typescript
{
  _id: ObjectId,
  tag_id: ObjectId,
  entity_type: "task" | "project" | "form",
  entity_id: ObjectId,
  created_at: Date,
}
```

### `notifications`
```typescript
{
  _id: ObjectId,
  user_id: ObjectId,
  type: string,                    // "task_assigned" | "form_submitted" | "inspection_due" | etc.
  title: string,
  body: string,
  entity_type: string,
  entity_id: ObjectId,
  is_read: boolean,
  created_at: Date,
}
```

---

## Tigris Object Storage Buckets

| Bucket | Contents |
|---|---|
| `plans` | Uploaded PDF floor plans + processed sheet images |
| `attachments` | Task/form photos and file attachments |
| `reports` | Generated PDF inspection reports + certificates |
| `logos` | Account logos |
| `exports` | Temporary export files (auto-expire 24h) |

### Object Key Patterns

```
plans/{account_id}/{project_id}/{sheet_id}/original.pdf
plans/{account_id}/{project_id}/{sheet_id}/thumbnail.png
attachments/{account_id}/{project_id}/{attachment_id}/{filename}
reports/{account_id}/{project_id}/{form_id}/inspection-report.pdf
```

---

## Indexes

Key indexes for query performance:

```
projects: { account_id, status }
projects: { ahj_id }
projects: { compliance_status }
tasks: { project_id, status }
tasks: { project_id, owner_user_id }
tasks: { project_id, task_type }
tasks: { device_id }
tasks: { due_date } (for compliance scheduler)
devices: { project_id, device_type }
devices: { project_id, last_test_result }
forms: { project_id, status }
forms: { project_id, form_template_id }
form_input_records: { form_id }
notifications: { user_id, is_read }
```
