# Fire Alarm Domain Features

Everything specific to fire protection/alarm work that doesn't exist in Fieldwire. These are the features that make this a fire alarm platform, not a generic field ops tool.

---

## NFPA 72 Compliance Framework

NFPA 72 (National Fire Alarm and Signaling Code) dictates testing intervals and procedures. Our app enforces these.

### Testing Intervals

| System Component | Required Frequency |
|---|---|
| Initiating devices (smoke detectors) | Annual |
| Heat detectors | Annual |
| Manual pull stations | Annual |
| Notification appliances (horns/strobes) | Annual |
| Control panel | Annual |
| Batteries (primary/secondary) | Annual (load test) + Semi-annual (visual) |
| Supervisory signals | Annual |
| Monitoring (central station) | Annual |
| Emergency voice/alarm comm | Annual |
| Sprinkler waterflow | Quarterly |
| Tamper switches | Quarterly |

### Compliance Tracking Fields (per job/project)

```
last_annual_inspection_date
next_annual_due_date             (= last + 365 days, configurable)
last_quarterly_inspection_date
next_quarterly_due_date
compliance_status: compliant | due_soon | overdue | deficient
deficiency_count_open
deficiency_count_critical        (life-safety)
certificate_issued_date
certificate_expiry_date
ahj_notification_required        (bool, set if critical deficiency open)
```

### Deficiency Severity Levels

| Level | Description | Response Time |
|---|---|---|
| `critical` | Impairs system operation, immediate life-safety risk | 24 hrs |
| `major` | Partial impairment, affects coverage area | 30 days |
| `minor` | Does not impair system, administrative | 90 days |

---

## Device Database

Every fire alarm device at every property tracked as a discrete record.

### Device Schema

```
id
project_id
location_id
sheet_id                  (which plan the device appears on)
position_x, position_y   (coordinates on sheet)

# Identity
device_type               (see types below)
zone_number               (fire alarm zone, e.g., "Zone 3")
loop_number               (for addressable systems, SLC loop)
address                   (addressable device address, 1-255 or more)
label                     (field label on panel, e.g., "SD-301")

# Hardware
manufacturer
model_number
serial_number
install_date
manufacture_date
warranty_expiry

# Status
operational_status: active | disabled | trouble | supervisory | removed
last_test_date
last_test_result: pass | fail | not_tested
last_test_technician_id
sensitivity                (for smoke detectors — percent/ft obscuration)
sensitivity_last_checked

# History
service_history[]         (array of service_log_ids)
```

### Device Types (NFPA 72 categories)

**Initiating Devices:**
- `smoke_detector_photo` — Photoelectric smoke detector
- `smoke_detector_ion` — Ionization smoke detector
- `smoke_detector_combo` — Combination
- `smoke_detector_duct` — Duct smoke detector
- `heat_detector_fixed` — Fixed-temperature heat detector
- `heat_detector_rate_of_rise` — Rate-of-rise heat detector
- `heat_detector_combo`
- `pull_station_single` — Single-action manual pull station
- `pull_station_double` — Double-action
- `beam_detector` — Projected beam smoke detector
- `co_detector` — Carbon monoxide detector
- `flame_detector` — Flame detector
- `waterflow_switch` — Sprinkler waterflow initiating device
- `tamper_switch` — Valve supervisory switch
- `pressure_switch`

**Notification Appliances:**
- `horn` — Audible horn
- `strobe` — Visual strobe only
- `horn_strobe` — Combined horn/strobe
- `speaker` — Voice evacuation speaker
- `speaker_strobe` — Combined speaker/strobe
- `chime`

**Control Equipment:**
- `facp` — Fire Alarm Control Panel (main panel)
- `facp_remote` — Remote annunciator
- `facp_networked` — Networked panel node
- `nac_extender` — NAC power extender
- `relay_module`
- `monitor_module`
- `control_module`

**Suppression-related:**
- `pre_action_valve`
- `deluge_valve`
- `suppression_releasing_panel`

---

## Fire Alarm Control Panel (FACP) Configuration

Each project/job has a panel record tracking the control panel details.

### Panel Schema

```
id
project_id
manufacturer              (Notifier, Simplex, Gamewell, EST, Bosch, Hochiki, etc.)
model
firmware_version
serial_number
install_date
last_programming_date
last_programming_tech_id

# Capacity
zone_count                (conventional systems)
slc_loop_count            (addressable systems)
devices_per_loop

# Battery
battery_manufacturer
battery_model
battery_ah                (amp-hour rating)
battery_install_date
battery_test_date
battery_test_result: pass | fail | marginal

# Supervision
monitoring_company
monitoring_account_number
monitoring_phone_primary
monitoring_phone_backup
central_station_receiver

# Panel zones[]
  zone_id
  zone_number
  zone_label               (e.g., "1st Floor East")
  zone_type: alarm | supervisory | trouble | monitor
  device_count
  circuit_style            (Class A / Class B)
```

---

## Inspection Forms (Pre-Built Templates)

Pre-loaded NFPA 72-compliant templates. Technicians select the template and fill it out in the field.

### Template: Annual Fire Alarm Inspection (NFPA 72)

**Section 1 — System Information**
- Customer name (auto-fill from project)
- Property address (auto-fill)
- Panel manufacturer, model, serial (auto-fill from panel record)
- Date of inspection
- Inspector name/license number
- Weather conditions
- Power status at start of inspection
- Monitoring company (auto-fill)

**Section 2 — Visual Inspection**
- Control panel condition (pass/fail/NA + notes)
- Remote annunciator (pass/fail/NA)
- AC power indicator illuminated (pass/fail)
- Trouble indicators (any present at start — document)
- Battery condition (visual — corrosion, swelling, connections)
- Notification appliances (visible obstruction check)
- Initiating devices (visible obstruction check)
- Accessible pull stations (within 5 ft of exit — NFPA)

**Section 3 — Functional Testing: Initiating Devices**
Repeating table per device (populated from Device DB):
- Device label / address
- Device type
- Test method
- Result (pass / fail / N/A)
- Sensitivity reading (if applicable)
- Notes
- Technician initials

**Section 4 — Functional Testing: Notification Appliances**
Same repeating table pattern.

**Section 5 — Functional Testing: Control Panel**
- All zones functional (pass/fail per zone)
- Trouble indication test (pass/fail)
- Supervisory signal test (pass/fail)
- Ground fault indication (pass/fail)
- AC loss indication (pass/fail)
- Battery charger test (pass/fail)
- Remote annunciator test (pass/fail)
- Printer/annunciator output (pass/fail + notes)

**Section 6 — Monitoring Verification**
- Alarm signal verified at monitoring company (pass/fail)
- Trouble signal verified (pass/fail)
- Supervisory signal verified (pass/fail)
- Monitoring company contact name

**Section 7 — Battery Testing**
- Battery load test performed (yes/no)
- Battery voltage under load
- Duration of load test
- Result (pass/fail)

**Section 8 — Deficiencies**
Auto-populated from tasks tagged as deficiencies on this project. Each row:
- Location
- Description
- NFPA reference
- Severity (critical/major/minor)
- Recommended action

**Section 9 — Sign-Off**
- All devices tested successfully (yes/no/partial)
- System restored to normal operation (yes/no)
- Customer representative name + signature
- Inspector signature + license number
- Date

### Template: Quarterly Inspection (Sprinkler Supervision)

Shorter template focused on waterflow and tamper switch testing per NFPA 25 intervals.

### Template: New Installation Acceptance Test (NFPA 72 Ch. 14)

Full 100% device test for new system commissioning. Each device gets an individual test record.

### Template: Deficiency Notice

Formal written notice to property owner of open deficiencies. Generated from inspection data. Auto-populates deficiency table from open tasks.

### Template: Certificate of Inspection

One-page certificate. Auto-fills from completed inspection form data.

---

## AHJ (Authority Having Jurisdiction) Management

Each job is under a specific AHJ (local fire marshal / fire department).

### AHJ Schema

```
id
name                      (e.g., "City of Atlanta Fire Marshal")
jurisdiction_type         (city | county | state | federal)
state
county
city

contact_name
contact_title
contact_phone
contact_email
address

# Requirements (customize per AHJ)
requires_annual_report    (bool)
report_submission_method  (mail | email | portal | in_person)
report_portal_url
report_due_date_rule      (e.g., "within 30 days of inspection")
requires_permit_renewal   (bool)
permit_renewal_interval   (months)
requires_witness_testing  (bool — AHJ witness required for acceptance tests)
special_requirements      (text — any AHJ-specific notes)
```

---

## Compliance Dashboard

Dashboard showing compliance status across all active jobs.

**Metrics per job:**
- System status (compliant, due_soon, overdue, deficient)
- Days until next inspection due
- Open deficiency count + critical count
- Last inspection date
- Certificate status

**Account-wide metrics:**
- % of portfolio compliant
- Jobs due in next 30/60/90 days
- Jobs overdue
- Open critical deficiencies

**Scheduling integration:**
- Auto-create tasks for upcoming inspections
- Calendar view of due dates
- Assign inspections to technicians from dashboard

---

## Client Portal

What paying clients (property owners/managers) can see when logged in.

**Access level:** Read-only on their own properties

**Views:**
- Property list (all their locations in our system)
- Per-property dashboard:
  - Compliance status + certificate
  - Last inspection date + next due date
  - Open deficiencies (with severity)
  - Inspection history (download PDF reports)
  - Device inventory
- Deficiency tracking (see status of open deficiencies)
- Document vault (inspection reports, certificates, permits)

**Value prop for upsell:** Property managers with multiple buildings pay per-building monthly subscription to maintain portal access and compliance tracking.

---

## Revenue Model (SaaS)

### Internal Tier (no charge)
- All 1st FP Alarm employees
- Full platform access

### Client Portal Tier
- Property owner / facility manager
- Read-only access to their buildings
- Per-building / per-month pricing
- ~$15-30/building/month

### Contractor Tier (other fire protection companies)
- Full platform license (same as internal tier)
- Per-seat or per-project pricing
- White-label option (custom domain, logo)
- ~$50-100/seat/month
