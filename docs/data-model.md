# Data Model

**Database: Fly Postgres** (structured data). **Tigris S3** (files only — plans, photos, PDFs, QR assets).

All tables use UUID primary keys and `TIMESTAMPTZ` for timestamps. `gen_random_uuid()` requires the `pgcrypto` extension.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

---

## Tables

### `accounts`
```sql
CREATE TABLE accounts (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name         TEXT NOT NULL,
  slug         TEXT UNIQUE NOT NULL,
  plan         TEXT NOT NULL CHECK (plan IN ('internal', 'client', 'contractor')),
  owner_user_id UUID,                       -- FK set after first user created
  logo_url     TEXT,                        -- Tigris object storage URL
  timezone     TEXT NOT NULL DEFAULT 'America/New_York',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at   TIMESTAMPTZ
);
```

### `users`
```sql
CREATE TABLE users (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id          UUID NOT NULL REFERENCES accounts(id),
  email               TEXT UNIQUE NOT NULL,
  name                TEXT NOT NULL,
  phone               TEXT,
  role                TEXT NOT NULL CHECK (role IN ('owner', 'project_manager', 'technician', 'inspector', 'client')),
  license_number      TEXT,                 -- fire alarm tech license
  license_state       TEXT,
  avatar_url          TEXT,
  is_active           BOOLEAN NOT NULL DEFAULT TRUE,
  refresh_token_hash  TEXT,                 -- hashed refresh token
  last_login_at       TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at          TIMESTAMPTZ
);
CREATE INDEX ON users (account_id);
```

### `ahjs`
Authority Having Jurisdiction records (referenced by projects, created before projects).
```sql
CREATE TABLE ahjs (
  id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id                  UUID NOT NULL REFERENCES accounts(id),
  name                        TEXT NOT NULL,
  jurisdiction_type           TEXT CHECK (jurisdiction_type IN ('city', 'county', 'state', 'federal')),
  state                       TEXT,
  county                      TEXT,
  city                        TEXT,
  contact_name                TEXT,
  contact_title               TEXT,
  contact_phone               TEXT,
  contact_email               TEXT,
  address                     TEXT,
  requires_annual_report      BOOLEAN NOT NULL DEFAULT FALSE,
  report_submission_method    TEXT CHECK (report_submission_method IN ('mail', 'email', 'portal', 'in_person')),
  report_portal_url           TEXT,
  report_due_date_rule        TEXT,
  requires_permit_renewal     BOOLEAN NOT NULL DEFAULT FALSE,
  permit_renewal_interval_months INTEGER,
  requires_witness_testing    BOOLEAN NOT NULL DEFAULT FALSE,
  special_requirements        TEXT,
  created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON ahjs (account_id);
```

### `projects`
```sql
CREATE TABLE projects (
  id                             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id                     UUID NOT NULL REFERENCES accounts(id),
  name                           TEXT NOT NULL,
  description                    TEXT,
  code                           TEXT,                   -- human-readable project code

  -- Address
  address                        TEXT,
  city                           TEXT,
  state                          TEXT,
  zip                            TEXT,
  latitude                       DOUBLE PRECISION,
  longitude                      DOUBLE PRECISION,

  -- People
  creator_user_id                UUID REFERENCES users(id),
  last_editor_user_id            UUID REFERENCES users(id),
  project_manager_id             UUID REFERENCES users(id),
  client_account_id              UUID REFERENCES accounts(id),

  -- Status
  status                         TEXT NOT NULL DEFAULT 'active'
                                   CHECK (status IN ('active', 'archived', 'complete')),
  contract_type                  TEXT CHECK (contract_type IN ('new_install', 'inspection', 'service', 'monitoring')),

  -- Fire alarm specifics
  building_type                  TEXT CHECK (building_type IN ('commercial', 'industrial', 'residential', 'high_rise', 'healthcare', 'educational')),
  system_type                    TEXT CHECK (system_type IN ('conventional', 'addressable', 'wireless', 'hybrid')),
  occupancy_classification       TEXT,
  ahj_id                         UUID REFERENCES ahjs(id),

  -- Compliance
  last_annual_inspection_date    DATE,
  next_annual_due_date           DATE,
  last_quarterly_inspection_date DATE,
  next_quarterly_due_date        DATE,
  compliance_status              TEXT NOT NULL DEFAULT 'compliant'
                                   CHECK (compliance_status IN ('compliant', 'due_soon', 'overdue', 'deficient')),
  certificate_number             TEXT,
  certificate_issued_date        DATE,
  certificate_expiry_date        DATE,
  ahj_notification_required      BOOLEAN NOT NULL DEFAULT FALSE,

  -- Denormalized counts (updated by triggers or background job)
  open_task_count                INTEGER NOT NULL DEFAULT 0,
  critical_deficiency_count      INTEGER NOT NULL DEFAULT 0,
  device_count                   INTEGER NOT NULL DEFAULT 0,

  created_at                     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at                     TIMESTAMPTZ
);
CREATE INDEX ON projects (account_id, status);
CREATE INDEX ON projects (ahj_id);
CREATE INDEX ON projects (compliance_status);
```

### `project_users`
Junction: which users have access to which projects.
```sql
CREATE TABLE project_users (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role       TEXT NOT NULL CHECK (role IN ('admin', 'member', 'viewer')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (project_id, user_id)
);
CREATE INDEX ON project_users (project_id);
CREATE INDEX ON project_users (user_id);
```

### `locations`
Hierarchical building structure (building → floor → zone → room).
```sql
CREATE TABLE locations (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id         UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  parent_location_id UUID REFERENCES locations(id),
  name               TEXT NOT NULL,
  level              INTEGER NOT NULL,          -- 0=building, 1=floor, 2=zone, 3=room
  path               TEXT,                      -- "Building A / Floor 2 / Zone 3" (materialized path)
  zone_number        TEXT,                      -- fire alarm zone if level=2
  active_task_count  INTEGER NOT NULL DEFAULT 0,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at         TIMESTAMPTZ
);
CREATE INDEX ON locations (project_id);
```

### `floorplan_collections`
Folder grouping for floorplans.
```sql
CREATE TABLE floorplan_collections (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id            UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name                  TEXT NOT NULL,
  parent_collection_id  UUID REFERENCES floorplan_collections(id),
  "order"               INTEGER NOT NULL DEFAULT 0,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON floorplan_collections (project_id);
```

### `floorplans`
```sql
CREATE TABLE floorplans (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  description     TEXT,
  collection_id   UUID REFERENCES floorplan_collections(id),
  creator_user_id UUID REFERENCES users(id),
  "order"         INTEGER NOT NULL DEFAULT 0,
  status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'ready', 'error')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX ON floorplans (project_id);
```

### `sheets`
One sheet per page of a floorplan PDF.
```sql
CREATE TABLE sheets (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  floorplan_id    UUID NOT NULL REFERENCES floorplans(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  version         INTEGER NOT NULL DEFAULT 1,
  "order"         INTEGER NOT NULL DEFAULT 0,
  file_key        TEXT,                -- Tigris key (original PDF page)
  thumbnail_key   TEXT,                -- Tigris key (PNG thumbnail)
  width_px        INTEGER,
  height_px       INTEGER,
  sheet_type      TEXT CHECK (sheet_type IN ('fire_alarm', 'electrical', 'architectural', 'as_built', 'other')),
  location_id     UUID REFERENCES locations(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX ON sheets (project_id);
CREATE INDEX ON sheets (floorplan_id);
```

### `markups`
GeoJSON features drawn on a sheet (device pins, zones, annotations).
```sql
CREATE TABLE markups (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  sheet_id        UUID NOT NULL REFERENCES sheets(id) ON DELETE CASCADE,
  creator_user_id UUID REFERENCES users(id),
  markup_type     TEXT NOT NULL CHECK (markup_type IN ('device_pin', 'zone_boundary', 'annotation', 'shape', 'text', 'arrow', 'cloud')),
  geometry        JSONB NOT NULL,      -- GeoJSON geometry; coordinates are 0–1 fractions of sheet dimensions
  color           TEXT,
  stroke_width    REAL,
  fill_opacity    REAL,
  label           TEXT,
  task_id         UUID REFERENCES tasks(id),
  device_id       UUID REFERENCES devices(id),
  form_id         UUID REFERENCES forms(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX ON markups (sheet_id);
CREATE INDEX ON markups (device_id) WHERE device_id IS NOT NULL;
```

### `panels`
Fire alarm control panel (FACP) per project.
```sql
CREATE TABLE panels (
  id                         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id                 UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  manufacturer               TEXT,
  model                      TEXT,
  firmware_version           TEXT,
  serial_number              TEXT,
  install_date               DATE,
  last_programming_date      DATE,
  last_programming_tech_id   UUID REFERENCES users(id),
  zone_count                 INTEGER,
  slc_loop_count             INTEGER,
  battery_manufacturer       TEXT,
  battery_model              TEXT,
  battery_ah                 REAL,
  battery_install_date       DATE,
  battery_test_date          DATE,
  battery_test_result        TEXT CHECK (battery_test_result IN ('pass', 'fail', 'marginal')),
  monitoring_company         TEXT,
  monitoring_account_number  TEXT,
  monitoring_phone_primary   TEXT,
  monitoring_phone_backup    TEXT,
  central_station_receiver   TEXT,
  created_at                 TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                 TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `panel_zones`
```sql
CREATE TABLE panel_zones (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  panel_id       UUID NOT NULL REFERENCES panels(id) ON DELETE CASCADE,
  zone_number    TEXT NOT NULL,
  zone_label     TEXT,
  zone_type      TEXT CHECK (zone_type IN ('alarm', 'supervisory', 'trouble', 'monitor')),
  device_count   INTEGER NOT NULL DEFAULT 0,
  circuit_style  TEXT CHECK (circuit_style IN ('A', 'B'))
);
CREATE INDEX ON panel_zones (panel_id);
```

### `devices`
One record per physical fire alarm device.
```sql
CREATE TABLE devices (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id                UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  location_id               UUID REFERENCES locations(id),
  sheet_id                  UUID REFERENCES sheets(id),
  position_x                REAL,           -- 0–1 fraction of sheet width
  position_y                REAL,           -- 0–1 fraction of sheet height

  -- Identity
  device_type               TEXT NOT NULL,  -- see fire-alarm-specifics.md for full enum
  zone_number               TEXT,
  loop_number               INTEGER,        -- addressable systems
  address                   INTEGER,        -- addressable device address
  label                     TEXT,           -- panel label (e.g., "SD-301")

  -- Hardware
  manufacturer              TEXT,
  model_number              TEXT,
  serial_number             TEXT,
  install_date              DATE,
  manufacture_date          DATE,
  warranty_expiry           DATE,

  -- Status
  operational_status        TEXT NOT NULL DEFAULT 'active'
                              CHECK (operational_status IN ('active', 'disabled', 'trouble', 'supervisory', 'removed')),

  -- Test tracking
  last_test_date            DATE,
  last_test_result          TEXT CHECK (last_test_result IN ('pass', 'fail', 'not_tested')),
  last_test_technician_id   UUID REFERENCES users(id),
  sensitivity               REAL,           -- smoke detectors: %/ft
  sensitivity_last_checked  DATE,

  -- QR code (permanent physical label on device)
  qr_code_id                TEXT UNIQUE NOT NULL,   -- short URL-safe ID (e.g., "d_abc123")
  qr_code_printed           BOOLEAN NOT NULL DEFAULT FALSE,
  qr_code_applied_date      DATE,

  -- External integrations
  servicetrade_asset_id     TEXT,

  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at                TIMESTAMPTZ
);
CREATE INDEX ON devices (project_id, device_type);
CREATE INDEX ON devices (project_id, last_test_result);
CREATE INDEX ON devices (qr_code_id);
```

### `tasks`
```sql
CREATE TABLE tasks (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id            UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  sequence_id           INTEGER NOT NULL,     -- auto-increment per project (T-0042)
  name                  TEXT NOT NULL,
  description           TEXT,

  -- Assignment
  creator_user_id       UUID REFERENCES users(id),
  last_editor_user_id   UUID REFERENCES users(id),
  owner_user_id         UUID REFERENCES users(id),
  watcher_ids           UUID[] NOT NULL DEFAULT '{}',

  -- Status
  status                TEXT NOT NULL DEFAULT 'open'
                          CHECK (status IN ('open', 'in_progress', 'ready_for_review', 'complete', 'deleted')),
  priority              TEXT NOT NULL DEFAULT 'normal'
                          CHECK (priority IN ('critical', 'high', 'normal', 'low')),
  task_type             TEXT NOT NULL DEFAULT 'general'
                          CHECK (task_type IN ('deficiency', 'service_call', 'installation_item', 'device_swap', 'panel_programming', 'inspection_item', 'general')),

  -- Schedule
  due_date              DATE,
  start_date            DATE,
  completed_at          TIMESTAMPTZ,

  -- Location on sheet
  location_id           UUID REFERENCES locations(id),
  sheet_id              UUID REFERENCES sheets(id),
  position_x            REAL,
  position_y            REAL,

  -- Fire alarm specifics
  device_id             UUID REFERENCES devices(id),
  nfpa_code             TEXT,
  corrective_action     TEXT,
  resolution_date       DATE,
  requires_reinspection BOOLEAN NOT NULL DEFAULT FALSE,
  ahj_notified          BOOLEAN NOT NULL DEFAULT FALSE,
  deficiency_severity   TEXT CHECK (deficiency_severity IN ('critical', 'major', 'minor')),

  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at            TIMESTAMPTZ,

  UNIQUE (project_id, sequence_id)
);
CREATE INDEX ON tasks (project_id, status);
CREATE INDEX ON tasks (project_id, owner_user_id);
CREATE INDEX ON tasks (project_id, task_type);
CREATE INDEX ON tasks (device_id) WHERE device_id IS NOT NULL;
CREATE INDEX ON tasks (due_date) WHERE due_date IS NOT NULL;
```

### `task_check_items`
```sql
CREATE TABLE task_check_items (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id),
  task_id         UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  is_complete     BOOLEAN NOT NULL DEFAULT FALSE,
  sequence_number INTEGER NOT NULL DEFAULT 0,
  assignee_id     UUID REFERENCES users(id),
  due_date        DATE,
  completed_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX ON task_check_items (task_id);
```

### `task_relations`
```sql
CREATE TABLE task_relations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id),
  task_id         UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  related_task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  relation_type   TEXT NOT NULL CHECK (relation_type IN ('depends_on', 'blocked_by', 'related_to')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON task_relations (task_id);
```

### `task_type_attributes`
Custom field definitions per project.
```sql
CREATE TABLE task_type_attributes (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  task_type   TEXT NOT NULL,
  name        TEXT NOT NULL,
  field_type  TEXT NOT NULL CHECK (field_type IN ('short_text', 'number', 'list', 'date', 'yes_no_na')),
  options     JSONB,           -- [{ id, label }] for list type
  "order"     INTEGER NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON task_type_attributes (project_id, task_type);
```

### `task_attributes`
Values of custom fields per task instance.
```sql
CREATE TABLE task_attributes (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id               UUID NOT NULL REFERENCES projects(id),
  task_id                  UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  task_type_attribute_id   UUID NOT NULL REFERENCES task_type_attributes(id) ON DELETE CASCADE,
  text_value               TEXT,
  number_value             DOUBLE PRECISION,
  option_id                TEXT,       -- for list type
  creator_user_id          UUID REFERENCES users(id),
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (task_id, task_type_attribute_id)
);
CREATE INDEX ON task_attributes (task_id);
```

### `attachments`
```sql
CREATE TABLE attachments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id),
  creator_user_id UUID REFERENCES users(id),
  file_key        TEXT NOT NULL,           -- Tigris object storage key
  file_name       TEXT NOT NULL,
  file_size       BIGINT,                  -- bytes
  content_type    TEXT,
  thumbnail_key   TEXT,
  upload_status   TEXT NOT NULL DEFAULT 'pending'
                    CHECK (upload_status IN ('pending', 'uploaded', 'failed')),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX ON attachments (project_id);
```

### `bubbles`
Comments, photos, files on tasks or sheets.
```sql
CREATE TABLE bubbles (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL REFERENCES projects(id),
  task_id         UUID REFERENCES tasks(id) ON DELETE CASCADE,
  sheet_id        UUID REFERENCES sheets(id) ON DELETE CASCADE,
  position_x      REAL,
  position_y      REAL,
  creator_user_id UUID REFERENCES users(id),
  content         TEXT,
  attachment_id   UUID REFERENCES attachments(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ,
  CHECK (task_id IS NOT NULL OR sheet_id IS NOT NULL)  -- must belong to one parent
);
CREATE INDEX ON bubbles (task_id) WHERE task_id IS NOT NULL;
CREATE INDEX ON bubbles (sheet_id) WHERE sheet_id IS NOT NULL;
```

### `form_templates`
```sql
CREATE TABLE form_templates (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id         UUID NOT NULL REFERENCES accounts(id),
  name               TEXT NOT NULL,
  description        TEXT,
  version            INTEGER NOT NULL DEFAULT 1,
  checksum           TEXT,                -- hash of template structure
  is_system_template BOOLEAN NOT NULL DEFAULT FALSE,
  is_active          BOOLEAN NOT NULL DEFAULT TRUE,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at         TIMESTAMPTZ
);
CREATE INDEX ON form_templates (account_id);
```

### `form_template_sections`
```sql
CREATE TABLE form_template_sections (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  form_template_id     UUID NOT NULL REFERENCES form_templates(id) ON DELETE CASCADE,
  name                 TEXT NOT NULL,
  description          TEXT,
  "order"              INTEGER NOT NULL DEFAULT 0,
  nfpa_reference       TEXT,              -- e.g., "NFPA 72 14.4.3"
  is_repeating         BOOLEAN NOT NULL DEFAULT FALSE,
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON form_template_sections (form_template_id);
```

### `form_template_inputs`
```sql
CREATE TABLE form_template_inputs (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  form_template_id          UUID NOT NULL REFERENCES form_templates(id) ON DELETE CASCADE,
  form_template_section_id  UUID NOT NULL REFERENCES form_template_sections(id) ON DELETE CASCADE,
  name                      TEXT NOT NULL,
  label                     TEXT NOT NULL,
  "order"                   INTEGER NOT NULL DEFAULT 0,
  input_type                TEXT NOT NULL CHECK (input_type IN (
    'short_text', 'long_text', 'number', 'date', 'dropdown', 'multiselect',
    'checkbox', 'signature', 'photo', 'file', 'yes_no_na',
    'device_reference', 'user_reference', 'location_reference'
  )),
  is_required               BOOLEAN NOT NULL DEFAULT FALSE,
  options                   TEXT[],       -- for dropdown / multiselect
  default_value             TEXT,
  help_text                 TEXT,
  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON form_template_inputs (form_template_section_id);
```

### `forms`
Instance of a template on a project.
```sql
CREATE TABLE forms (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id              UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  form_template_id        UUID NOT NULL REFERENCES form_templates(id),
  form_template_checksum  TEXT,
  name                    TEXT NOT NULL,
  description             TEXT,
  creator_user_id         UUID REFERENCES users(id),
  last_editor_user_id     UUID REFERENCES users(id),
  owner_user_id           UUID REFERENCES users(id),
  status                  TEXT NOT NULL DEFAULT 'draft'
                            CHECK (status IN ('draft', 'submitted', 'approved', 'rejected')),
  submitted_at            TIMESTAMPTZ,
  approved_at             TIMESTAMPTZ,
  approved_by_user_id     UUID REFERENCES users(id),
  pdf_key                 TEXT,              -- Tigris key for generated PDF
  inspection_date         DATE,
  location_id             UUID REFERENCES locations(id),
  created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at              TIMESTAMPTZ
);
CREATE INDEX ON forms (project_id, status);
CREATE INDEX ON forms (project_id, form_template_id);
```

### `form_section_records`
```sql
CREATE TABLE form_section_records (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  form_id                   UUID NOT NULL REFERENCES forms(id) ON DELETE CASCADE,
  form_template_section_id  UUID NOT NULL REFERENCES form_template_sections(id),
  iteration                 INTEGER NOT NULL DEFAULT 0,  -- for repeating sections, which row
  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON form_section_records (form_id);
```

### `form_input_records`
```sql
CREATE TABLE form_input_records (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  form_id                  UUID NOT NULL REFERENCES forms(id) ON DELETE CASCADE,
  form_section_record_id   UUID NOT NULL REFERENCES form_section_records(id) ON DELETE CASCADE,
  form_template_input_id   UUID NOT NULL REFERENCES form_template_inputs(id),
  text_value               TEXT,
  number_value             DOUBLE PRECISION,
  date_value               DATE,
  boolean_value            BOOLEAN,
  option_values            TEXT[],          -- selected option(s) for multiselect/dropdown
  attachment_id            UUID REFERENCES attachments(id),
  reference_id             UUID,            -- device, user, or location ID
  creator_user_id          UUID REFERENCES users(id),
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON form_input_records (form_id);
CREATE INDEX ON form_input_records (form_section_record_id);
```

### `entity_tags`
```sql
CREATE TABLE entity_tags (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id  UUID NOT NULL REFERENCES accounts(id),
  name        TEXT NOT NULL,
  color       TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (account_id, name)
);
```

### `entity_taggings`
```sql
CREATE TABLE entity_taggings (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tag_id      UUID NOT NULL REFERENCES entity_tags(id) ON DELETE CASCADE,
  entity_type TEXT NOT NULL CHECK (entity_type IN ('task', 'project', 'form')),
  entity_id   UUID NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tag_id, entity_type, entity_id)
);
CREATE INDEX ON entity_taggings (entity_type, entity_id);
```

### `closeout_packages`
```sql
CREATE TABLE closeout_packages (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id                UUID NOT NULL REFERENCES projects(id),
  generated_by_user_id      UUID REFERENCES users(id),
  status                    TEXT NOT NULL DEFAULT 'draft'
                              CHECK (status IN ('draft', 'sent', 'archived')),
  included_form_ids         UUID[] NOT NULL DEFAULT '{}',
  included_sheet_ids        UUID[] NOT NULL DEFAULT '{}',
  include_device_inventory  BOOLEAN NOT NULL DEFAULT TRUE,
  include_deficiency_log    BOOLEAN NOT NULL DEFAULT TRUE,
  include_certificate       BOOLEAN NOT NULL DEFAULT TRUE,
  pdf_key                   TEXT,              -- Tigris key for bundled PDF
  generated_at              TIMESTAMPTZ,
  sent_at                   TIMESTAMPTZ,
  sent_to                   TEXT[] NOT NULL DEFAULT '{}',  -- email addresses
  billing_milestone_triggered BOOLEAN NOT NULL DEFAULT FALSE,
  sage_invoice_id           TEXT,
  created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON closeout_packages (project_id);
```

### `deficiency_quotes`
Tracks the deficiency → ServiceTrade quote pipeline.
```sql
CREATE TABLE deficiency_quotes (
  id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id                  UUID NOT NULL REFERENCES projects(id),
  task_id                     UUID NOT NULL REFERENCES tasks(id),
  quote_status                TEXT NOT NULL DEFAULT 'pending'
                                CHECK (quote_status IN ('pending', 'sent_to_st', 'customer_sent', 'accepted', 'declined', 'expired')),
  servicetrade_quote_id       TEXT,
  servicetrade_work_order_id  TEXT,
  estimated_value             NUMERIC(12, 2),
  repair_scope                TEXT,
  days_open_at_last_notify    INTEGER NOT NULL DEFAULT 0,
  last_notified_at            TIMESTAMPTZ,
  notified_user_ids           UUID[] NOT NULL DEFAULT '{}',
  created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (task_id)
);
CREATE INDEX ON deficiency_quotes (project_id, quote_status);
```

### `servicetrade_sync`
ServiceTrade integration config per account.
```sql
CREATE TABLE servicetrade_sync (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id               UUID UNIQUE NOT NULL REFERENCES accounts(id),
  servicetrade_company_id  TEXT NOT NULL,
  api_key_encrypted        TEXT NOT NULL,
  sync_deficiencies        BOOLEAN NOT NULL DEFAULT TRUE,
  sync_jobs                BOOLEAN NOT NULL DEFAULT TRUE,
  sync_customers           BOOLEAN NOT NULL DEFAULT TRUE,
  project_st_job_map       JSONB NOT NULL DEFAULT '{}',   -- { "project_id": "st_job_id" }
  customer_st_map          JSONB NOT NULL DEFAULT '{}',   -- { "client_account_id": "st_customer_id" }
  last_sync_at             TIMESTAMPTZ,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `webhooks`
```sql
CREATE TABLE webhooks (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id  UUID NOT NULL REFERENCES accounts(id),
  url         TEXT NOT NULL,
  secret      TEXT NOT NULL,              -- HMAC signing secret
  events      TEXT[] NOT NULL,            -- e.g., '{"task.created","form.submitted"}'
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON webhooks (account_id);
```

### `notifications`
```sql
CREATE TABLE notifications (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type         TEXT NOT NULL,         -- "task_assigned" | "form_submitted" | "inspection_due" | etc.
  title        TEXT NOT NULL,
  body         TEXT,
  entity_type  TEXT,
  entity_id    UUID,
  is_read      BOOLEAN NOT NULL DEFAULT FALSE,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON notifications (user_id, is_read);
```

---

## Tigris Object Storage Buckets

| Bucket | Contents |
|---|---|
| `1stfpalarm-plans` | Uploaded PDF floor plans + processed sheet PNG images |
| `1stfpalarm-attachments` | Task/form photos and file attachments |
| `1stfpalarm-reports` | Generated PDF inspection reports + certificates |
| `1stfpalarm-exports` | Temporary export files (auto-expire 24h) |

### Object Key Patterns

```
{account_id}/plans/{project_id}/{sheet_id}/original.pdf
{account_id}/plans/{project_id}/{sheet_id}/thumbnail.png
{account_id}/attachments/{project_id}/{attachment_id}/{filename}
{account_id}/reports/{project_id}/{form_id}/inspection-report.pdf
{account_id}/reports/{project_id}/closeout-{closeout_id}.pdf
{account_id}/qr/{device_id}/label.png
```

---

## Notes

- **`markups` FK order**: `markups` references `tasks`, `devices`, and `forms` which are defined later in the schema — use `CREATE TABLE` then `ALTER TABLE ADD FOREIGN KEY` in migrations, or use a migration tool (Prisma, Flyway) that resolves dependency order automatically.
- **`panel_id` on `projects`**: Projects reference a panel, but panels reference projects. Handle this in migrations: create `projects` without the panel FK, create `panels`, then `ALTER TABLE projects ADD COLUMN panel_id UUID REFERENCES panels(id)`.
- **Denormalized counts** (`open_task_count`, `critical_deficiency_count`, `device_count` on `projects`): Updated by a background job or Postgres triggers, not application code — keeps dashboard queries fast.
- **UUID arrays** (`watcher_ids`, `included_form_ids`, etc.): Convenient for the pilot. If any of these need querying individually at scale, convert to junction tables.
