# JHA Prototype — Technical Specification

**Product:** J. J. Keller Safety Management Suite — Job Hazard Analysis
**Prototype file:** `index.html` (repo root)
**Last updated:** 2026-06-30
**Size:** ~10,900 lines · single HTML file · 5 features completed

---

## Table of Contents

1. [Overview](#1-overview)
2. [UI Views](#2-ui-views)
   - [Wizard Steps](#wizard-steps)
   - [Start Job Flow](#start-job-flow)
   - [JHA Status Model](#jha-status-model)
3. [Reference Data](#3-reference-data)
   - [Trade Templates](#trade-templates)
   - [Control Taxonomy](#control-taxonomy)
   - [Risk Scoring](#risk-scoring)
4. [Database Schema](#4-database-schema)
   - [CompanyData namespace](#companydata-namespace)
   - [Jha.JhaForm tree](#jhajhaform-tree)
   - [Jha.JhaJob tree](#jhajhajob-tree)
   - [Training](#training)
   - [Incident Recordkeeping](#incident-recordkeeping)
   - [JJK Lookup Tables](#jjk-lookup-tables)
5. [Feature Log](#5-feature-log)
6. [Dev Timeline](#6-dev-timeline)

---

## 1. Overview

A self-contained HTML prototype for the JHA module inside the JJ Keller SMS platform. It demonstrates the full end-to-end workflow — from creating a JHA through a 6-step wizard to running a job-site "Start Job" safety walk-through — using an in-browser file system database (File System Access API + IndexedDB for persistence).

### Tech Stack

| Layer | Implementation |
|---|---|
| Runtime | Single HTML file, vanilla JS — no build step, no framework |
| Persistence | File System Access API (JSON files) + IndexedDB (handle persistence) |
| Icons | Google Material Icons (CDN) |
| Fonts | Inter & Roboto (CDN) |
| QR codes | api.qrserver.com (external, demo only) |
| PDF / Print | Browser native `window.print()` with `@media print` CSS |
| IDs | `crypto.randomUUID()` for DB rows; sequential human-readable IDs for JHAs (`JHA2026NNNNN`) and Jobs (`JOB2026NNNNN`) |

### Local DB Storage

The prototype writes JSON files to a user-selected local directory via the File System Access API. The directory handle is persisted in IndexedDB (`JJKA_SMS` store, key `dbRoot`). Table data is stored as individual `.json` files under a `Database/` subfolder, one file per table.

---

## 2. UI Views

All views render into `#mainContent` via `setView(v)`. The sidebar navigation mirrors the production SMS sidebar pattern.

| View | `setView()` key | Description |
|---|---|---|
| Dashboard | `dashboard` | KPI tiles (total JHAs, completed jobs, hazards, controls), recent activity, trend charts. Lazy-rendered via double-rAF to avoid blocking the spinner paint. |
| JHA Forms | `list` | Filterable grid of all JHA records. Columns: ID, Title, PDF, Location, Work Area, Equipment, Job Function, Supervisor, Review Due By, Status, Steps, Hazards, Controls, Uncontrolled Risk, Controlled Risk. |
| Completed Jobs | `completed` | Archive of completed job run records linked back to the originating JHA form. Row menu: View, Edit, Print Report, Add Incident, View JHA Form. |
| Start Job | `start-job` | Pre-job safety walk-through. Reviewer confirms every control per hazard per task. Progress bar tracks confirmations. Crew signatures collected before completion. |
| Employee Access | `employee-access` | Public-facing JHA list available via a shareable URL and downloadable QR code. Employees can view and sign without logging in. |
| Reporting Center | `reporting-center` | Standard report templates: hazard frequency by category, risk score distribution, completed-job summary, crew sign-off tracking. |
| Training | `training` | Training record management linked to job functions. Writes to `Jha.Training`. |
| Incident Recordkeeping | `incidents` | OSHA-aligned incident record entry. Types: Recordable, Near Miss, First Aid, Property Damage, Equipment Failure, Safety Observation, Other. |
| Company Data | `company-data` | Reference data management: Locations, Work Areas, Job Functions, Equipment, Employees, Data Groups. DB connection status displayed. |

### Wizard Steps

New JHA creation follows a 6-step wizard (`WIZARD_STEPS = ['Job Info','Steps','Hazard ID','Controls','Crew Review','Complete']`):

| # | Step | Key Fields / Actions |
|---|---|---|
| 1 | Job Info | Title, Type (multi-select), Supervisor, Location/Facility, Work Area, Data Group, Equipment, Job Function, Date, LOTO Required, Training Required, Review Frequency, Review Due By — **+ Template picker** with industry filter tabs (All / Manufacturing / Construction / Transportation / Health Care) |
| 2 | Steps | Task list seeded from selected trade template. Each task: name, description, photo upload. Tasks can be reordered, added, or removed. Lazy-seeded from `wiz.trade.tasks` on first render. |
| 3 | Hazard ID | Per-task hazard list seeded from trade template. Each hazard: category badge, severity, frequency sliders. State: `accepted`, `dismissed`, or `rejected`. AI-flagged hazards marked. Custom hazards can be added. |
| 4 | Controls | For each accepted hazard: select control type (hierarchy: Elimination → Substitution → Engineering → Administrative → PPE), then a specific control from a sub-list. Owner and due-date fields per control. Validation prevents advancing without a specific control on every hazard. |
| 5 | Crew Review | Add crew members from employee list. Each member types a signature to sign off. Status: all signed → Complete; any unsigned → Pending Approval. |
| 6 | Complete | Confirmation screen. "Make Available to Employees" button. Closes wizard, opens JHA detail view. `finishWizard()` writes to DB. |

> **Save & Close (any step):** `saveWizardDraft()` saves as *In Progress* from any wizard step. `_ensureWizTasks()` is called first to seed tasks from the trade template if the user never reached Step 2.

### Start Job Flow

1. **Select JHA** — choose a completed JHA from the list (or enter via row menu → Start Job).
2. **Reviewer Info** — reviewer name, job title, work area, equipment in use.
3. **Checklist** — task-by-task accordion. Each hazard shows its controls; reviewer checks each one (`rev.checks[c{ti}_{hi}_{ci}]`). Progress bar updates in real time. Hazards can be removed via × button (`removeRevHazard(ti, hi)`).
4. **Crew Sign-off** — each crew member types their name as a signature. Completion writes to `Jha.JhaJob`, `Jha.JhaJobControlReview`, and `Jha.JhaJobSignatures`.

### JHA Status Model

Status is computed dynamically by `computeJhaStatus(j)` — never stored directly:

| Status | Condition |
|---|---|
| In Progress | Any accepted hazard has no controls assigned |
| Pending Approval | All hazards controlled; crew members exist but at least one hasn't signed |
| Complete | All hazards controlled AND all crew signed (or no assignees required) |

---

## 3. Reference Data

### Trade Templates

18 trade templates across 5 industry groups. Each provides pre-seeded tasks and hazards.

| ID | Name | Industry | Tasks | Hazards |
|---|---|---|---|---|
| `cse` | Confined Space Entry | Construction | 3 | 6 |
| `rfw` | Rooftop HVAC Service | Electrical | 4 | 6 |
| `exc` | Excavation & Trenching | Construction | 4 | 5 |
| `ew` | Electrical Panel Work | Electrical | 4 | 5 |
| `wah` | Working at Heights – Scaffolding | Construction | 4 | 6 |
| `hm` | Hazardous Material Handling | Manufacturing | 3 | 5 |
| `gr` | Grinding & Cutting Operations | Manufacturing | 3 | 5 |
| `lm` | Line Maintenance – Utilities | Utilities | 4 | 6 |
| `fkl` | Forklift Operations | Warehouse | 4 | 6 |
| `pht` | Patient Handling & Transfer | Health Care | 4 | 6 |
| `sbwh` | Sharps & Biohazard Waste Handling | Health Care | 4 | 6 |
| `cmp` | Chemical Manufacturing Process | Manufacturing | 5 | 8 |
| `mda` | Medication Administration | Health Care | 4 | 6 |
| `cmv-gen` | CMV Driver – General | Transportation | 5 | 13 |
| `cdl-prop` | CDL Driver – Property | Transportation | 5 | 13 |
| `cdl-flat` | CDL Driver – Flatbeds | Transportation | 2 | 7 |
| `cdl-tank` | CDL Driver – Tank Vehicles | Transportation | 2 | 7 |
| `cdl-pass` | CDL Driver – Passengers | Transportation | 2 | 9 |
| `noncdl-prop` | Non-CDL Drivers – Property | Transportation | 2 | 8 |
| `drv-pass` | Driver – Passengers | Transportation | 1 | 3 |
| `cdl-hazmat` | CDL Driver – Hazmat | Transportation | 3 | 10 |

### Control Taxonomy

Based on the OSHA hierarchy of controls. Five types, each with a curated sub-list of specific controls:

| Type | Risk Multiplier | # Specific Controls |
|---|---|---|
| Elimination | 0.0 (eliminates risk) | ~40 |
| Substitution | 0.2 | ~45 |
| Engineering Control | 0.4 | ~50 |
| Administrative Control | 0.6 | ~35 |
| PPE | 0.8 | ~65 |

**Hazard categories** (`HAZ_CATS`):
`Fall`, `Physical`, `Atmospheric`, `Electrical`, `Chemical`, `Ergonomic`, `Struck-By`, `Health`, `Psychosocial`, `Environmental`, `Falls`, `Caught-in/Between`, `Radiological`, `Custom`

### Risk Scoring

Risk score formula: `score = (severity + frequency) × controlMultiplier`

Scores are averaged across all hazards on a JHA to produce the overall uncontrolled and controlled risk ratings shown in the list view.

| Score | Label |
|---|---|
| 0 | ELIMINATED |
| 1–3 | LOW |
| 4–6 | MEDIUM |
| 7–8 | HIGH |
| ≥ 9 | CRITICAL |

**Category default risk values** (used when no custom sev/freq set):

```
Fall:        {sev:4, freq:3}    Physical:    {sev:3, freq:3}
Atmospheric: {sev:5, freq:2}    Electrical:  {sev:5, freq:2}
Chemical:    {sev:4, freq:2}    Ergonomic:   {sev:2, freq:3}
Struck-By:   {sev:4, freq:3}    Health:      {sev:3, freq:3}
Custom:      {sev:3, freq:3}
```

---

## 4. Database Schema

Tables are organized into four namespaces: `CompanyData`, `Auth`, `Jha`, and `JJK` (reference/lookup).

> **Prototype-only fields:** Fields prefixed with `_` (e.g. `_title`, `_supervisor`) are prototype stopgaps. In production these will be replaced by proper FK relationships to normalized tables.

**Shared audit columns** on all transactional tables:
```
isDeleted BOOLEAN
createdDate DATETIME
createdByUserId UUID
modifiedDate DATETIME
modifiedByUserId UUID
impersonationCreatedByUserId UUID?
impersonationModifiedByUserId UUID?
```

---

### CompanyData namespace

#### CompanyData.Employee

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| companyId | UUID FK | Tenant scoping |
| firstName | VARCHAR | |
| lastName | VARCHAR | |
| middleInitial | CHAR(1)? | |
| employeeCode | VARCHAR | e.g. `EMP001` |
| jobTitle | VARCHAR? | |
| workEmail | VARCHAR? | Used for control owner assignment |
| isActive | BOOLEAN | |
| supervisorId | UUID FK? | Self-referential |
| _+ audit columns_ | | |

#### CompanyData.Location / WorkArea / Equipment / JobFunction

All share the same shape:

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| companyId | UUID FK | |
| name | VARCHAR | |
| _+ audit columns_ | | |

#### Auth.Groups

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| companyId | UUID FK | |
| name | VARCHAR | Data group name |
| _+ audit columns_ | | |

---

### Jha.JhaForm tree

#### Jha.JhaForm

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR PK | Human-readable: `JHA2026NNNNN` |
| employeeId | UUID FK | → CompanyData.Employee (creator) |
| groupId | UUID FK | → Auth.Groups |
| jhaTypeId | UUID FK? | → JJK.JhaType (null in prototype) |
| reviewByDate | DATE? | |
| jhaReviewFrequencyId | UUID FK? | → JJK.JhaReviewFrequency |
| isLotoRequired | BOOLEAN | |
| isTrainingRequired | BOOLEAN | |
| jhaTemplateId | UUID FK? | → Jha.JhaTemplate (null in prototype) |
| jhaOverallUncontrolledRiskId | UUID FK? | → JJK.JhaFormHazardSeverity |
| jhaOverallControlledRiskId | UUID FK? | → JJK.JhaFormHazardSeverity |
| isAvailableToEmployees | BOOLEAN | Controls Employee Access view |
| `_title` *(proto)* | VARCHAR | → production: normalized field |
| `_type` *(proto)* | VARCHAR | Trade template name |
| `_location` *(proto)* | VARCHAR | Denormalized location string |
| `_supervisor` *(proto)* | VARCHAR | → production: FK to Employee |
| `_site` *(proto)* | VARCHAR | |
| `_dataGroup` *(proto)* | VARCHAR | |
| `_date` *(proto)* | DATE | JHA creation date |
| `_reviewedOn` *(proto)* | DATE? | |
| `_reviewFrequency` *(proto)* | VARCHAR? | |
| `_jhaTypes` *(proto)* | JSON array | → production: junction table |
| `_status` *(proto)* | VARCHAR | Computed via `computeJhaStatus()` |
| `_crew` *(proto)* | INT | Denormalized crew count |
| _+ audit columns_ | | |

#### Jha.JhaFormLocationAssociation

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaFormId | VARCHAR FK | → Jha.JhaForm |
| locationId | UUID FK? | → CompanyData.Location (null if new) |
| `_locationName` *(proto)* | VARCHAR | |
| _+ audit columns_ | | |

*Same pattern for:* `Jha.JhaFormWorkAreaAssociation` (workAreaId), `Jha.JhaFormJobFunctionAssociation` (jobFunctionId), `Jha.JhaFormEquipmentAssociation` (equipmentId).

#### Jha.JhaFormCrewMember

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaFormId | VARCHAR FK | → Jha.JhaForm |
| sortOrder | INT | |
| `_name` *(proto)* | VARCHAR | → production: FK to Employee |
| `_role` *(proto)* | VARCHAR | Defaults to "Crew Member" |
| `_isSigned` *(proto)* | BOOLEAN | |
| `_signedDate` *(proto)* | DATE? | |
| _+ audit columns_ | | |

#### Jha.JhaFormJobTask

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaFormId | VARCHAR FK | → Jha.JhaForm |
| name | VARCHAR | Task/step name |
| sortOrder | INT | |
| `_description` *(proto)* | TEXT | |
| `_mediaFiles` *(proto)* | JSON array | `[{name, fileType}]` → production: blob storage refs |
| _+ audit columns_ | | |

#### Jha.JhaFormJobTaskHazard

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaFormJobTaskId | UUID FK | → Jha.JhaFormJobTask |
| jhaHazardId | UUID FK? | → JJK.JhaHazard (null for custom hazards) |
| uncontrolledRiskId | UUID FK? | → JJK.JhaFormHazardSeverity |
| controlledRiskId | UUID FK? | → JJK.JhaFormHazardSeverity |
| sortOrder | INT | |
| `_name` *(proto)* | VARCHAR | Hazard display name |
| `_cat` *(proto)* | VARCHAR | Category key (e.g. `Fall`, `Health`) |
| `_sev` *(proto)* | INT 1–5 | Severity |
| `_freq` *(proto)* | INT 1–5 | Frequency |
| `_state` *(proto)* | ENUM | `accepted` \| `dismissed` \| `rejected` |
| `_ai` *(proto)* | BOOLEAN | AI-suggested flag |
| _+ audit columns_ | | |

#### Jha.JhaFormJobTaskHazardControl

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaFormJobTaskHazardId | UUID FK | → Jha.JhaFormJobTaskHazard |
| jhaControlTypeId | UUID FK? | → JJK.JhaControlType |
| sortOrder | INT | |
| `_type` *(proto)* | VARCHAR | Hierarchy type (e.g. `Engineering Control`) |
| `_text` *(proto)* | VARCHAR | Specific control text |
| `_owner` *(proto)* | VARCHAR? | Responsible person name |
| `_due` *(proto)* | DATE? | Due date for implementation |
| _+ audit columns_ | | |

---

### Jha.JhaJob tree

#### Jha.JhaJob

| Column | Type | Notes |
|---|---|---|
| id | VARCHAR PK | Human-readable: `JOB2026NNNNN` |
| jhaFormId | VARCHAR FK? | → Jha.JhaForm (null for blank jobs) |
| employeeId | UUID FK | → CompanyData.Employee |
| groupId | UUID FK | → Auth.Groups |
| completedByUserId | UUID FK | |
| completedByJobTitle | VARCHAR? | |
| completedDate | DATETIME | |
| reviewedByUserId | UUID FK? | |
| reviewedDate | DATETIME? | |
| `_type` *(proto)* | VARCHAR | |
| `_completedBy` *(proto)* | VARCHAR | |
| `_supervisor` *(proto)* | VARCHAR? | |
| `_location` *(proto)* | VARCHAR | |
| `_workArea` *(proto)* | VARCHAR | |
| `_equipment` *(proto)* | VARCHAR | |
| `_hazards` *(proto)* | INT | Denormalized count |
| `_controls` *(proto)* | INT | Denormalized count |
| `_crew` *(proto)* | INT | Denormalized count |
| _+ audit columns_ | | |

#### Jha.JhaJobControlReview

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaJobId | VARCHAR FK | → Jha.JhaJob |
| reviewerName | VARCHAR | |
| reviewedDate | DATETIME | |
| workArea | VARCHAR | |
| equipment | VARCHAR | |
| checksJson | JSON | Map of `c{ti}_{hi}_{ci}` → boolean |
| _+ audit columns_ | | |

#### Jha.JhaJobSignatures

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| jhaJobId | VARCHAR FK | → Jha.JhaJob |
| employeeName | VARCHAR | |
| jobTitle | VARCHAR | |
| typedSignature | VARCHAR | Typed name — production may use canvas/SVG |
| signedAt | DATETIME | |
| _+ audit columns_ | | |

*Association tables:* `Jha.JhaJobLocationAssociation`, `Jha.JhaJobWorkAreaAssociation`, `Jha.JhaJobEquipmentAssociation` — same FK + name pattern as JhaForm associations.

---

### Training

#### Jha.Training

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| companyId | UUID FK | |
| title | VARCHAR | |
| duration | VARCHAR? | e.g. "2 hours" |
| isActive | BOOLEAN | |
| _+ audit columns_ | | |

**Jha.TrainingJobFunctionAssociation:** `id`, `trainingId` → Jha.Training, `jobFunctionId` → CompanyData.JobFunction (nullable), `_jobFunctionName` *(proto)*.

---

### Incident Recordkeeping

#### Jha.IncidentRecord

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| companyId | UUID FK | |
| caseNumber | VARCHAR | Format: `{LOC}{YEAR}{NNNNN}` |
| status | ENUM | `To Do` \| `In Progress` \| `On Hold` \| `Reopened` \| `Complete` |
| type | ENUM | `Recordable` \| `Near Miss` \| `First Aid` \| `Property Damage` \| `Equipment Failure` \| `Safety Observation` \| `Other` |
| employeeLastName | VARCHAR? | |
| employeeFirstName | VARCHAR? | |
| nonEmployeeName | VARCHAR? | For non-employee incidents |
| jobTitle | VARCHAR? | |
| location | VARCHAR? | |
| dateOfIncident | DATE? | |
| timeOfIncident | TIME? | |
| timeUnknown | BOOLEAN | |
| whereOccurred | VARCHAR? | |
| workArea | VARCHAR? | |
| equipment | VARCHAR? | |
| description | TEXT? | |
| _+ audit columns_ | | |

- **Jha.IncidentJobFunctionAssociation** — links incidents to job functions.
- **Jha.IncidentLinkedJob** — `id`, `incidentId`, `jhaJobId` — links incidents to completed job runs.

---

### JJK Lookup Tables

| Table | Purpose | Status |
|---|---|---|
| `JJK.JhaType` | JHA type values (Maintenance/Service, Startup, Operation, Shutdown, Pre-Trip, Post-Trip) | Seeded at DB init |
| `JJK.JhaHazardType` | Hazard category lookup (Fall, Health, Chemical, …) | Seeded |
| `JJK.JhaHazard` | Named hazard master list | Seeded from TRADES |
| `JJK.JhaControlType` | Hierarchy types (Elimination → PPE) | Seeded |
| `JJK.JhaSpecificControl` | All specific control sub-options per type | Seeded from CONTROL_SUBCATEGORIES |
| `JJK.JhaFormHazardSeverity` | Severity scale labels (1–5) | Seeded |
| `JJK.JhaFormHazardFrequency` | Frequency scale labels (1–5) | Seeded |
| `JJK.JhaReviewFrequency` | Review cadence options (Daily … Every 5 Years) | Seeded |

---

## 5. Feature Log

### Feature 1 — Remove Hazard from Start Job

**Code locations:** `removeRevHazard` ~line 8797 · × button in `buildRevChecklist` ~line 8547 and `buildRevTaskBody` ~line 8931

Users can remove a hazard from the checklist during the Start Job safety walk-through. An × button appears on each hazard row. On removal, `removeRevHazard(ti, hi)`:

- Splices the hazard from `rev.detail.tasks[ti].hazards`
- Re-keys all `rev.checks` entries (format: `c{ti}_{hi}_{ci}`) to keep indices consistent after removal
- Re-renders the affected task body via `document.getElementById('rtask-body-${ti}')`
- Calls `updateRevProgress()` to refresh the progress bar

---

### Feature 2 — Multi-Page Print Fix

**Root cause:** `#printOverlay` had `overflow: hidden` and a fixed `height` at print time, preventing the browser's print engine from paginating the full document.

**Fix** — updated `@media print` block:

```css
@media print {
  body > *:not(#printOverlay) { display: none !important; }
  #printOverlay {
    display: block !important;
    position: static !important;
    height: auto !important;
    overflow: visible !important;
  }
  #printControls { display: none !important; }
  .print-doc { max-width: none !important; padding: 0 !important; }
  .print-header { break-after: avoid; }
  .print-section { break-inside: avoid; }
  .print-task-header { break-after: avoid; break-inside: avoid; }
  .print-crew-table { break-inside: avoid; }
  .print-footer { break-before: auto; }
  @page { size: letter portrait; margin: 0.6in 0.5in; }
}
```

Each task block after the first received `break-before: page` in `buildPrintDoc()`. The Crew sign-off section also gets `break-before: page`.

---

### Feature 3 — Health Hazard Category

"Heat Stress in Enclosed Space" (Confined Space Entry trade) and "Heat Exhaustion / Stroke" (Rooftop HVAC Service trade) reclassified from `Ergonomic` → `Health`.

**Changes made:**

- Updated `TRADES` array entries: `cat:'Health'` on both hazards
- Added `Health` to all category color maps:
  - `CAT_BG: Health: '#e0f2f1'` (teal background)
  - `CAT_TC: Health: '#00695c'` (teal text)
- Added `Health` to all risk scoring default maps (`_RC`, `CAT_DEF`, `CAT_DEFAULTS`, `_CAT_DEF`): `Health: {sev:3, freq:3}`

---

### Feature 4 — Dev & Launch Plan (.docx)

Microsoft Word Dev & Launch Plan built with Node.js `docx` npm package, mirroring the structure of the SMS ENV template:

- TOC, Executive Summary, Product Overview, Key Stakeholders, Timeline & Milestones, Development Phases, Launch Strategy, Success Metrics, Risk Register, Appendix
- TOC uses `headingStyleRange: '1-3'` without `hyperlink: true` to avoid the "fields that may refer to other files" Word dialog on open
- Word will still prompt to update fields on open — standard TOC behavior, cannot be suppressed without removing the TOC

---

### Feature 5 — Bug Fix: JHA Form View Shows Only Step 1

**Code location:** `_ensureWizTasks()` ~line 6149 · called in `saveWizardDraft()` and `finishWizard()`

**Root cause:** `buildStep2()` lazily seeds `wiz.tasks` from the selected trade template — but only when it renders. If a user clicked "Save & Close" while still on Step 1 (before ever reaching Step 2), `wiz.tasks` had not been seeded. The detail view then showed 0 or 1 steps instead of the template's full task list.

**Fix:** Added `_ensureWizTasks()` called at the top of both `saveWizardDraft()` and `finishWizard()`:

```javascript
function _ensureWizTasks() {
  if (!wiz.tasks.length && wiz.trade) {
    wiz.tasks = (wiz.trade.tasks || []).map((t, i) => ({
      id: 'task_' + i,
      name: typeof t === 'string' ? t : t.name,
      description: typeof t === 'string' ? '' : (t.description || ''),
      hazards: [],
      media: [],
    }));
  }
}
```

> **Note:** Existing JHAs already saved with only 1 task are unaffected. Those would need to be re-edited via the wizard to add the missing steps.

---

## 6. Dev Timeline

Full production development estimate for 2–3 developers, using the prototype as the design reference:

| Phase | Estimate | Notes |
|---|---|---|
| Backend & Data Layer | 6–8 weeks | Schema, migrations, repositories, services |
| Frontend (from prototype) | 4–6 weeks | Prototype serves as near-complete design reference |
| Platform Integrations | 2–4 weeks | Auth (Okta/SSO), SecurityContext scoping, existing SMS APIs |
| Testing | 3–4 weeks | Unit, integration, E2E, QA |
| Launch Prep | 2–3 weeks | Staging, UAT, data migration |
| **Total** | **~4–5 months** | |

**Compressing factors:** existing JJKellerPortal infrastructure (auth, API patterns, DB access, EF Core, code-gen scaffold); prototype serves as near-complete design reference reducing front-end decision time.

**Extending factors:** regulatory/compliance review (OSHA alignment), mobile requirements, integration complexity with existing platform data, ActiveReports PDF generation for JHA forms.
