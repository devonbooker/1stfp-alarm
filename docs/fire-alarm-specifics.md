# Fire Alarm Domain Features

Everything specific to fire protection/alarm work that doesn't exist in Fieldwire. These are the features that make this a fire alarm platform, not a generic field ops tool.

---

## Business Context (1st Fire Protection Services)

- ~$40M revenue, targeting $100M+
- 25 field technicians, 12 project managers, 15 estimators, 12 inspectors
- Current stack: ServiceTrade (jobs/CRM), Sage Intacct (accounting), BambooHR (HR), Microsoft 365
- **1,171 aged unquoted deficiencies = ~$761K repair opportunity** sitting idle (as of deck date)
- Estimated 5,300 hours/year lost to duplicate data entry, searching, and rebuilding closeout packages
- APi Group benchmark: 54% of revenue from inspection/service/monitoring - long-term target >60%

### The core problem
Work gets captured multiple times (field → PM → accounting → closeout). The same data lives in different systems. Deficiencies found in the field never get converted to quotes. Closeout packages are rebuilt from scratch at the end of every job instead of assembled as work happens.

### What this app solves
**Capture the work once. Use it across the company.**

The app is the field layer - it captures what happens on site. It then feeds that into ServiceTrade, Sage, and BambooHR via integrations, so data flows automatically instead of being re-entered.

### Existing integrations this app must connect to
- **ServiceTrade** - source of truth for customers, jobs, and invoicing. New deficiency tasks created here should appear in ServiceTrade as service quotes.
- **Sage Intacct** - accounting. Closeout packages and billing milestones flow here.
- **BambooHR** - HR/employee data. Technician assignments pull from here.
- **Microsoft 365** - email/calendar. Inspection reports emailed from here.

---

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

## QR Code System

Every physical device gets a permanent QR code label applied in the field. The QR links to that device's record in the app.

**What the QR unlocks:**
- Technicians scan on arrival - instant access to device history, last test result, deficiencies, serial number, O&M manual
- Customers scan - see their system documentation (limited portal view)
- No app login required for initial scan (public landing page with property info, then auth prompt for details)

**QR code data:**
- Encodes: `device_id` + `project_id`
- URL pattern: `app.1stfpalarm.com/scan/{device_id}`
- QR label: weatherproof vinyl, printed in-house via label printer

**QR code persistence:** The physical label stays on the device forever. The record behind it gets richer every inspection. This is the core value-over-time proposition for customers.

---

## Deficiency-to-Quote Pipeline

This is the #1 revenue opportunity. 1,171 aged unquoted deficiencies = ~$761K sitting idle.

### How it works

1. Inspector finds a deficiency during inspection
2. Creates a deficiency task with: description, device_id, location, NFPA reference, severity, photo
3. App prompts: "Generate quote for this deficiency?" with pre-filled repair scope
4. Quote pushed to ServiceTrade as a service opportunity on the same customer/location
5. PM reviews and sends to customer from ServiceTrade
6. When quote is accepted and work is scheduled, work order links back to original deficiency task
7. When repair is complete, deficiency task is marked resolved

### Deficiency aging dashboard

Shows leadership: deficiencies by age (30/60/90/90+), unquoted count, estimated value, customer bucket.

### Auto-prompting

When a deficiency is open for >14 days with no quote created, PM and inspector get a notification: "Deficiency from [date] at [property] has not been quoted. Estimated value: $X."

---

## Closeout Package Automation

Currently rebuilt from scratch at the end of every job. This app assembles it as work happens.

### What a closeout package contains
- Completed inspection/test forms (PDF)
- As-built floor plans (PDF export from plan viewer)
- Device inventory list (auto-generated from Device DB)
- Open deficiency log (if any remaining)
- Certificate of inspection
- O&M manuals (linked from device records)
- Photos (pulled from task bubbles and form inputs)
- Change order documentation (future: link to Sage)

### How automation works
At any point, PM clicks "Generate Closeout Package" on a project:
- App checks what's complete vs. missing (shows a checklist: forms ✓, as-builts ✓, device inventory ✓, certs ✗)
- Compiles what's ready into a PDF bundle (via Puppeteer)
- Uploads to Tigris
- Sends to customer via email or portal
- PM can re-generate as many times as needed as items get completed

### Billing trigger
When closeout package is marked "sent to customer," a webhook fires to Sage Intacct to trigger the billing milestone for that project phase.

---

## Property Record (Permanent Building File)

The app's master record per property - built up job by job over time.

### What it contains
- All floor plans (current + historical versions)
- All devices (with full history)
- All past inspections (forms as PDFs)
- All deficiencies (open and resolved with resolution dates)
- Panel configuration
- Certificates and permits
- Monitoring information
- AHJ information
- O&M manuals and warranty docs

### Key behaviors
- Record persists even between jobs (not job-scoped, property-scoped)
- When a new job starts at a property, tech sees the full history before arriving
- Customer can access their portion via portal (read-only)
- "Technicians spend less time searching and show up already knowing the building."

---

## Build Roadmap

From the executive deck: phased capability plan.

### Phase 1 - Build Now
- Fieldwire replacement feature parity (plans, tasks, forms, markups)
- Permanent property and equipment records
- QR-linked device documentation
- Automated closeout packages
- ServiceTrade deficiency → quote push
- Basic compliance dashboard

### Phase 2 - Build Next
- Deficiency-to-quote automation with ServiceTrade integration
- Customer documentation portal (multi-location, $12K-$50K/yr)
- Automatic field-change tracking (changes on plans auto-flagged)
- Account cross-sell recommendations (sprinkler customer → alarm inspection opportunity)
- Fire alarm modernization opportunity reports
- Office and department scorecards (Sage + BambooHR data)

### Phase 3 - Build Later
- AI equipment detection on floor plans (auto-identify device symbols, pre-populate device DB)
- AI-assisted takeoff (count devices from plans)
- Automatic plan-revision comparison (highlight changes between versions)
- Predictive equipment replacement (based on age + manufacturer data)
- AI-generated capital plans for customers
- Company-wide operational forecasting

---

## Revenue Model (Validated Pricing Hypotheses from Deck)

### Internal (no charge)
All 1st FP employees. Full access.

### Digital Property Record (one-time project revenue)
| Service | Price |
|---|---|
| Initial record setup | $1.5K - $5K per property |
| Living as-built package | $2.5K - $10K |
| Equipment audit | $2.5K minimum |

### Compliance and Inspection Portal (recurring revenue)
| Service | Price |
|---|---|
| Multi-location portal access | $12K - $50K / year |
| Premium inspection documentation (uplift on existing inspection contract) | 5-10% uplift |
| O&M and warranty portal | Bundled |

### Capital Planning and Alarm Modernization (project revenue)
| Service | Price |
|---|---|
| Capital replacement forecast | $2.5K - $7.5K |
| Alarm modernization roadmap | $2.5K - $10K |
| Annual asset validation | $500 - $2.5K |

### Contractor SaaS (other fire protection companies)
- Full platform license
- Per-seat or per-project
- White-label option (custom domain, logo)
- ~$50-100/seat/month (to validate)
