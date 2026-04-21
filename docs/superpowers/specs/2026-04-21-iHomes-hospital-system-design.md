# iHomes Hospital Information System — Design Specification

- **Status:** Draft for review
- **Date:** 2026-04-21
- **Author:** Design session
- **Target audience:** Implementation team (3 devs) + hospital stakeholders

---

## 1. Overview

iHomes is a production Hospital Information System for a PhilHealth-accredited Level 3 hospital in the Philippines. The system handles the full patient lifecycle from ER triage through in-patient admission to discharge, with deep integration into the PhilHealth Electronic Claims System (eCS), centered on the 2025-launched **Outpatient Emergency Care Benefit (OECB)** package.

The system is inspired by DOH iHOMIS but is a greenfield implementation built with a modern PHP stack.

## 2. Goals & Non-Goals

### Goals

- End-to-end ER workflow supporting OECB submission within 24 hours of discharge.
- Full Out-Patient (OPD) and In-Patient (IPD) workflows.
- Ancillary services: Laboratory (LIS), Radiology (RIS), Pharmacy.
- Financial workflows: Billing, Cashier, HMO claims.
- PhilHealth eClaims for OECB, CF1, CF2, CF3, Z-Benefit, MCP, NCP packages.
- Medical Records with ICD-10 coding and discharge summaries.
- HR with staff management and basic payroll.
- Reports and dashboards for admin and department heads.
- Queue management for OPD/ER/Pharmacy.
- RBAC with audit logging for DPA compliance.
- On-premise deployment resilient to internet outages.

### Non-Goals (MVP)

- Cloud mirror / multi-site replication (deferred; may be added later).
- HL7 lab machine auto-integration (manual result entry in v1).
- DICOM/PACS viewer (link-out only in v1).
- Patient portal / self-service.
- Mobile apps (iOS/Android).
- Telemedicine features.
- Biometric time in/out.
- AI-assisted coding or triage.
- Full ERP accounting (general ledger, fixed assets).

## 3. Users & Personas

| Persona | Primary modules |
|---|---|
| Admitting clerk | Patient registration, admissions |
| Triage nurse | ER triage, vitals |
| ER nurse | ER nursing notes, medication administration |
| ER doctor | ER consultation, orders, discharge |
| Attending physician | OPD, IPD rounds, discharge summary |
| Resident physician | Orders, notes (under attending) |
| Med tech | LIS result entry |
| Rad tech / Radiologist | RIS result entry, reporting |
| Pharmacist | Dispensing, inventory |
| Cashier / Billing clerk | Charges, payments, SOA |
| MRS officer / ICD coder | Medical records, ICD-10 assignment |
| PhilHealth liaison officer | Claim submission, remittance reconciliation |
| HR officer | Staff, schedules, payroll |
| Department head | Department oversight, reports |
| Hospital admin / medical director | System-wide reports, user management |
| IT super admin | System configuration, backups |
| Auditor | Read-only audit log review |

## 4. Decisions Summary

| Area | Decision |
|---|---|
| Architecture | Modular Monolith |
| Language / Framework | PHP 8.3 / CodeIgniter 4 |
| Database | MariaDB 11 |
| Cache / Queue | Redis 7 |
| Web server | Nginx + PHP-FPM |
| OS | Ubuntu Server 24.04 LTS |
| Dev environment | Docker Compose |
| CSS framework | Bootstrap 5.3 |
| Interactivity | HTMX + Alpine.js |
| PDF generation | Dompdf or mPDF |
| Background workers | Supervisor |
| Deployment | On-premise only in MVP |
| Backup | Local NAS + off-site Backblaze B2 |
| Monitoring | Netdata + Graylog |
| Target size | Level 3 hospital, 100–300 beds, 30–100 concurrent users |
| Team | 3 developers, 6-month timeline |

## 5. Architecture

### 5.1 Deployment Topology (On-Premise)

A single production server sa hospital hosts the entire application stack. LAN workstations access via intranet HTTPS. Internet is used only for PhilHealth submissions and off-site backup.

```
HOSPITAL (on-premise)
─────────────────────

  Production Server (Ubuntu 24.04)
  ├── Nginx (TLS termination, reverse proxy)
  ├── PHP-FPM 8.3 → iHomes CI4 application
  ├── MariaDB 11 (primary DB)
  ├── Redis 7 (sessions, cache, queues)
  └── Supervisor
      ├── PhilHealth submit worker
      ├── Backup worker
      └── Scheduled jobs (cron)

  Backup targets
  ├── Local NAS (daily rsync)
  └── Backblaze B2 (nightly encrypted)

  Network
  ├── LAN workstations (ER, OPD, Lab, Pharmacy, etc.)
  └── Internet (PhilHealth eCS API + off-site backup only)
```

### 5.2 Modular Monolith Pattern

A single CodeIgniter 4 application, deployable as one unit, but internally organized into strongly-bounded modules. Cross-module communication goes exclusively through **service classes** — never direct database access across module boundaries.

**Rules:**

1. Each module has its own Controllers, Models, Services, Views, Routes, and Migrations.
2. Controllers may only invoke Services from their own module or from `Shared`.
3. To invoke another module's logic, use that module's public Service class (registered via CI4 `service()` helper).
4. Models are private to their module — no external callers.
5. Schemas are in a shared MariaDB database, but tables are prefixed per module.

### 5.3 Module Catalog

**Shared (core):**
- Auth / RBAC
- Patient Master Index (MPI)
- Encounter (central clinical record)
- Audit Log
- Masterfiles (ICD-10, CPT, item catalog, facilities, departments, beds, doctors)
- UI Components (reusable partials)

**Clinical:** ER, OPD, IPD, LIS, RIS, Pharmacy, MRS, Queueing.

**Financial:** Billing, PhilHealth, HMO.

**Admin:** HR, Reports.

### 5.4 Folder Structure

```
iHomes/
├── app/
│   ├── Config/
│   ├── Modules/
│   │   ├── Shared/
│   │   │   ├── Entities/
│   │   │   ├── Services/
│   │   │   ├── Models/
│   │   │   └── Auth/
│   │   ├── ER/
│   │   ├── OPD/
│   │   ├── IPD/
│   │   ├── LIS/
│   │   ├── RIS/
│   │   ├── Pharmacy/
│   │   ├── Billing/
│   │   ├── MRS/
│   │   ├── HR/
│   │   ├── PhilHealth/
│   │   ├── HMO/
│   │   ├── Reports/
│   │   └── Queueing/
│   ├── Commands/
│   ├── Filters/
│   └── Libraries/
├── public/
├── writable/
├── tests/
│   ├── Modules/
│   └── _support/
├── docs/
├── docker/
├── docker-compose.yml
├── composer.json
├── .env
└── spark
```

### 5.5 Core Domain Model

```
Patient (1) ─── (*) Encounter (1) ─── (*) Orders
                     │                 ├── LabOrder
                     │                 ├── ImagingOrder
                     │                 ├── Prescription
                     │                 └── Procedure
                     │
                     ├── (*) Charges (posted to Billing)
                     ├── (*) ClinicalNotes (ER consult, nursing, doctor orders)
                     ├── (0..1) Admission (IPD only)
                     ├── (0..1) DischargeSummary
                     └── (0..1) PhilHealthClaim
```

**Encounter** is the anchor: every clinical, financial, and administrative artifact attaches to an Encounter. A single patient visit = one Encounter. Admitted patients have an `Admission` child record; ER/OPD encounters do not.

## 6. Database Design

### 6.1 Conventions

- Naming: `snake_case` with module prefix (`er_`, `lis_`, `rx_`, `bill_`, `ph_`, `ris_`, `ipd_`, `mrs_`, `hmo_`, `hr_`, `queue_`).
- Primary key: `id BIGINT UNSIGNED AUTO_INCREMENT`.
- Secondary key: `uuid CHAR(36)` on every table (for future cloud sync).
- Soft deletes: nullable `deleted_at`. No hard deletes for clinical data.
- Audit columns: `created_at`, `updated_at`, `created_by`, `updated_by` on every table.
- JSON columns for flexible/evolving data (vitals, allergies, address).

### 6.2 Shared Tables (no prefix)

- `patients` — master patient index
- `encounters` — all visits (ER, OPD, IPD)
- `users`, `roles`, `permissions`, `role_permissions`, `user_roles`, `user_departments`, `sessions`, `password_history`
- `audit_log`
- Masterfiles: `facilities`, `departments`, `wards`, `beds`, `doctors`, `items`, `icd10`, `cpt_codes`, `lab_tests`, `imaging_studies`

### 6.3 Module Tables (selected)

**ER:** `er_triages`, `er_consultations`.

**IPD:** `ipd_admissions`, `ipd_doctor_orders`, `ipd_nursing_notes`, `ipd_medication_administration`.

**LIS:** `lis_orders`, `lis_results`, `lis_specimens`.

**RIS:** `ris_orders`, `ris_results`.

**Pharmacy:** `rx_inventory_lots`, `rx_prescriptions`, `rx_dispensing`, `rx_returns`, `rx_ward_stock`.

**Billing:** `bill_charges`, `bill_soa`, `bill_payments`, `bill_adjustments`, `bill_deposits`, `bill_refunds`.

**MRS:** `mrs_clinical_abstracts`, `mrs_discharge_summaries`, `mrs_icd_assignments`.

**PhilHealth:** `ph_eligibility_checks`, `ph_claims`, `ph_claim_errors`, `ph_remittances`, `ph_submission_queue`.

**HMO:** `hmo_providers`, `hmo_approvals`, `hmo_claims`.

**HR:** `hr_employees`, `hr_schedules`, `hr_time_logs`, `hr_payroll_runs`, `hr_payslips`.

**Queueing:** `queue_tickets`.

### 6.4 Expected Volume (Level 3 Hospital)

| Table | Per day | Per year |
|---|---|---|
| encounters | ~300 | ~110K |
| lis_orders | ~500 | ~180K |
| ris_orders | ~100 | ~36K |
| rx_prescriptions | ~400 | ~145K |
| bill_charges | ~3,000 | ~1.1M |
| audit_log | ~10,000 | ~3.6M |

Year-1 DB size estimate: 15–25 GB.

## 7. PhilHealth Integration

### 7.1 Overview

Integrates with PhilHealth Electronic Claims System (eCS) via SOAP Web Services. Submits XML-formatted claims per supported package and reconciles PhilHealth responses and remittances.

### 7.2 Required Credentials

- Healthcare Institution (HCI) code
- Sender ID + password for eCS Web Service
- PKI digital signing certificate (x509)
- Whitelisted submission IP addresses

### 7.3 Supported Packages

OECB (primary), CF1, CF2, CF3, Z-Benefit, MCP, NCP, TB-DOTS, HD, Animal Bite, ABTC, Konsulta.

### 7.4 Submission Flow

1. MRS officer marks encounter "ready for claim" after ICD-10 coding, finalization of diagnosis, and charge posting.
2. `ClaimBuilder` detects package type and assembles the claim data structure.
3. `ClaimValidator` runs schema (XSD) validation and business rules (e.g., OECB ≤24-hour discharge, no DAMA, filing within 60 days).
4. On validation success, `SignatureService` digitally signs the XML.
5. Claim enters `ph_submission_queue` with status `pending`.
6. Supervisor-managed worker polls queue every minute, submits via SOAP, and records response.
7. Successful submissions transition to `submitted`; rejections surface errors to MRS for correction.
8. `RemittanceParser` reconciles payment advice from PhilHealth, updating claim status to `paid` and posting corresponding payments to Billing.

### 7.5 Module Structure

```
app/Modules/PhilHealth/
├── Controllers/
│   ├── ClaimController.php
│   ├── EligibilityController.php
│   └── RemittanceController.php
├── Services/
│   ├── EligibilityService.php
│   ├── ClaimBuilder.php
│   ├── ClaimValidator.php
│   ├── EClaimsClient.php
│   ├── SignatureService.php
│   ├── OECBService.php
│   ├── CF1Service.php
│   ├── ZPackageService.php
│   └── RemittanceParser.php
├── Builders/
│   ├── OECBClaimBuilder.php
│   ├── CF1ClaimBuilder.php
│   └── ...
├── Schemas/
└── Config/
    └── PhilHealth.php
```

### 7.6 Resilience

Queue-based submission isolates hospital operations from internet outages. Claims pile up as `pending` during outages and drain on reconnect. Retries use exponential backoff up to 24 hours.

### 7.7 Environment-Based Endpoints

| Environment | Endpoint | Mode |
|---|---|---|
| local (dev) | mock SOAP container | `mock` |
| staging | PhilHealth UAT | `uat` |
| production | PhilHealth eCS | `production` |

A local mock SOAP server runs in the Docker dev stack to avoid depending on PhilHealth UAT availability during day-to-day development.

## 8. Authentication & Access Control

### 8.1 Authentication

- Username + password (primary).
- 6-digit PIN for shared-workstation quick re-auth.
- TOTP 2FA for admin, MRS, cashier supervisor.
- Password policy: min 12 chars, complexity, no reuse of last 5, top-10K-leaked blocklist, optional 90-day rotation.
- Account lockout after 5 failed attempts (15-minute auto-unlock).

### 8.2 Session Management

- 30-minute idle timeout for clinical modules.
- Workstation profiles (`ER-PC-01`, etc.) avoid re-entering department each login.
- Concurrent sessions allowed (doctor on multiple workstations).
- Forced logout on password change.

### 8.3 RBAC

Permission codes follow `module.resource.action` pattern (e.g., `er.triage.create`, `billing.payment.void`, `philhealth.claim.submit`). Roles aggregate permissions; users can hold multiple roles.

### 8.4 Special Rules

- **Break-glass access:** emergency override with reason + supervisor PIN; audited heavily.
- **Time-bound edit windows:** doctor orders editable within 1 hour of creation; lab results mutable only pre-release; billing charges editable only pre-finalization.
- **Own-patient restriction:** doctors see only assigned patients; access to others requires referral/consultation.
- **Department scoping:** staff see data scoped to their department by default.

## 9. Frontend & UI

### 9.1 Rendering Approach

Server-rendered views (CI4 layouts + partials). Bootstrap 5.3 for layout and components. HTMX for partial page updates. Alpine.js for lightweight local reactivity. No build step, no Node.js required at runtime.

### 9.2 Layout

Fixed sidebar (module nav) + topbar (quick search, notifications, user menu). Desktop-first; tablet-usable; mobile-minimal.

### 9.3 Key UI Patterns

- **Patient header strip** consistent across modules.
- **Encounter dashboard** with tabs (triage, consult, orders, results, billing).
- **Catalog-based order entry** (no free-text test/med names).
- **Ctrl+K command palette** for quick patient search and navigation.
- **Print-optimized views** with CSS `@page` for receipts, SOA, prescriptions, lab requests, discharge summaries, PhilHealth forms.

### 9.4 Accessibility & Speed

- High-contrast mode for older staff and bright ER environments.
- Autocomplete for ICD-10, medications, tests.
- 5-second undo for destructive actions.
- Inline validation and loading states.
- No raw stack traces in production.

### 9.5 Internal REST API

Exposed under `/api/v1/` using CI4 Resource Controllers for future integrations (mobile, iPad rounds, external systems). JWT-based auth. Consistent JSON response envelope: `{success, data, meta, errors}`.

## 10. Testing Strategy

### 10.1 Pyramid

- Unit tests (~70%): pure business logic — ClaimBuilder, ClaimValidator, ChargeCalculator, InventoryService, BedManagementService, ResultFlagService, EncounterService.
- Integration tests (~25%): CI4 FeatureTestTrait — controllers + DB.
- E2E tests (~5%): Playwright for critical journeys.

### 10.2 Critical E2E Journeys

1. ER/OECB: register → triage → consult → orders → discharge → claim build.
2. Admission: admit → ward assign → rounds → discharge summary → billing → claim.
3. Pharmacy: prescription → verify → dispense → inventory deduct.
4. Cashier: post charge → receive payment → print OR.
5. RBAC: nurse blocked from billing.

### 10.3 Test Data

- Factories for Patient, Encounter, User, etc.
- Non-test seeds: ICD-10 master, CPT codes, drug formulary, PhilHealth packages, roles, permissions.

### 10.4 Coverage Targets

| Module | Unit | Integration | E2E |
|---|---|---|---|
| PhilHealth | 95%+ | 80% | 100% critical |
| Billing | 90%+ | 80% | 100% |
| Pharmacy | 85%+ | 70% | 80% |
| ER / Consultation | 70%+ | 60% | 80% |
| Reports | 60%+ | 50% | 40% |

### 10.5 Environments

Local (Docker, per dev) → CI (GitHub Actions or equivalent) → Staging (pre-release, anonymized prod sample) → Production (real hospital).

## 11. Error Handling & Observability

### 11.1 Error Layers

1. **Validation (expected):** inline, field-level, user-friendly.
2. **Domain (recoverable):** queue for retry or surface for manual resolution (e.g., PhilHealth rejection).
3. **System (unexpected):** log, alert IT, fail gracefully — never expose stack traces.

### 11.2 Logging

PSR-3 levels (`debug` through `emergency`). CI4 file logs → Filebeat → Graylog for centralized search. Critical/alert levels also fan out to SMS/email.

### 11.3 Monitoring Alerts

| Condition | Level | Action |
|---|---|---|
| DB down | emergency | SMS + call IT admin |
| Disk >90% | alert | Email + auto-cleanup logs |
| PhilHealth queue >100 | alert | Email + in-app banner |
| Failed login >20/min | warning | Possible brute-force alert |
| ER triage >3s response | warning | Investigate |
| Any 500 error | error | Daily digest to dev team |

### 11.4 Graceful Degradation

Module-level failures do not cascade: PhilHealth down → claims queue; Reports down → clinical ops unaffected; Pharmacy sync off → stock banner warning; Email/SMS down → log-only fallback.

### 11.5 Audit Log

Every destructive action logs actor, target, before/after JSON, reason (if provided), IP, workstation, timestamp. Append-only. Retained 10+ years for legal compliance.

## 12. Infrastructure

### 12.1 Production Hardware (minimum for Level 3)

- CPU: 8-core (Intel Xeon / AMD EPYC)
- RAM: 32 GB (upgradable to 64 GB)
- Storage: 2×1 TB NVMe SSD RAID 1 + 2×4 TB HDD RAID 1
- Network: Gigabit LAN, dual NIC
- Power: UPS 1500VA+

### 12.2 Software Stack (Production)

- OS: Ubuntu Server 24.04 LTS
- Web: Nginx + PHP-FPM 8.3
- DB: MariaDB 11
- Cache/Queue: Redis 7
- Process manager: Supervisor
- Container runtime: Docker + Docker Compose
- Backup: mariabackup + rsync + rclone
- Monitoring: Netdata
- Log aggregation: Graylog
- Security: Fail2ban, Wazuh agent

### 12.3 Dev Environment

Docker Compose with services: Nginx, PHP-FPM (8.3 with extensions), MariaDB 11, Redis 7, Mailhog, PhilHealth mock SOAP.

### 12.4 Deployment Pipeline

Git push → CI (tests + Docker image build) → SSH deploy to production. Symlink-based releases (`/releases/{timestamp}` → `/current`). Migrations compatible across old+new code during brief overlap.

### 12.5 Estimated Monthly Cost

- Off-site backup (500 GB Backblaze B2): ~₱200/month
- Domain + Let's Encrypt SSL: free
- Monitoring (Netdata self-hosted): free
- **Total cloud: ~₱200/month**
- One-time on-premise hardware: ~₱150K–250K

## 13. Security & Compliance

### 13.1 Data Privacy Act 2012 (RA 10173)

- Consent tracking for data processing.
- Patient data encryption at rest (MariaDB `innodb_encrypt_tables=ON`).
- TLS everywhere (LAN + internet).
- Access logs retained 10+ years.
- Data minimization in logs (no PHI in application logs).
- Breach notification procedure documented.

### 13.2 Hardening

- Fail2ban on SSH and Nginx.
- Wazuh agent for host IDS.
- Daily encrypted backups, off-site rotation.
- Audit log immutability (append-only, hashed checksums).

### 13.3 PhilHealth Compliance

- Digital signature certificate kept in HSM or encrypted file store.
- Submission retry window ≤60 days per claim (PhilHealth filing deadline).
- Audit-ready trail of every claim version.

## 14. Roadmap

### Phase 0 — Foundation Infrastructure (Weeks 1–2)

Git repo, Docker dev environment, CI4 scaffold, CI/CD pipeline, base layout, coding standards, testing setup, production server procurement.

### Phase 1 — Core Foundation Modules (Weeks 3–6)

Shared Auth, RBAC, Audit, Patient MPI, Encounter core, shared UI components, masterfiles seeding, admin screens.

### Phase 2 — ER + OECB MVP ⭐ (Weeks 7–12)

ER triage + consultation, minimal LIS/RIS/Pharmacy, minimal Billing + Cashier, minimal MRS, PhilHealth OECB end-to-end (eligibility → build → validate → sign → queue → submit → acknowledge), essential prints.

**Gate:** OECB end-to-end submission to PhilHealth UAT must succeed. No-go triggers scope re-plan.

### Phase 3 — OPD + Full Ancillaries (Weeks 13–16)

OPD module, full LIS (ref ranges, critical values, specimen tracking, batch entry, cumulative views), full RIS (worklist, dictation, templates), full Pharmacy (FIFO lot mgmt, expiry, reorder, ward stock, returns, interaction warnings), Queueing.

### Phase 4 — Full Billing, MRS, HMO (Weeks 17–19)

Senior/PWD/dependent discounts, HMO split billing, professional fees, adjustments, deposits, refunds, credit accounts. PhilHealth CF1/CF2/CF3/MCP/NCP/Z builders. HMO (Maxicare, Intellicare, PhilCare). MRS full (discharge summaries, chart completion tracking). Core reports (census, revenue, remittance, aging, top diagnoses).

### Phase 5 — IPD (Weeks 20–22)

Admission slip, bed assignment, deposits, ward bed board, transfers, nursing assessments, vitals charts, I&O, MAR, doctor orders with sets, rounds view, discharge workflow (order → clearance → summary → final billing → gate pass).

### Phase 6 — Polish + HR + UAT + Go-Live (Weeks 23–26)

HR module (employees, schedules, time logs, basic payroll). Expanded reports (ad-hoc, DOH/PHIC). Performance tuning, security hardening, penetration test, DPA audit, training, 2-week parallel UAT, data migration, cutover, support hotline, go-live.

### Post-Launch (Month 7+)

Bug fixes, feature backlog, cloud mirror (if elected), mobile app, HL7 lab integration, DICOM/PACS viewer, biometrics, patient portal.

### Team Assignment

- **Dev 1 — Clinical Lead:** ER, OPD, IPD, MRS.
- **Dev 2 — Integration Lead:** PhilHealth, HMO, Billing, LIS, RIS.
- **Dev 3 — Platform Lead:** Shared, Auth, Reports, HR, UI components, DevOps.

## 15. Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| PhilHealth API docs incomplete | HIGH | Kick off liaison meeting Week 1; obtain SDK before Phase 2. |
| Hospital stakeholder unavailable for UAT | HIGH | Recurring weekly demos from Phase 2 onward. |
| Scope creep from department staff | HIGH | Change-control freeze after Phase 2; new requests post-MVP. |
| Key dev leaves mid-project | MEDIUM | Pair programming, shared knowledge, daily code reviews. |
| Hardware procurement delay | MEDIUM | Order Week 1; cloud dev/staging as backup. |
| DPA compliance audit gaps | MEDIUM | Security review before Phase 4; external audit before go-live. |
| Network/LAN downtime in hospital | MEDIUM | Server-side graceful handling; documented SOP for manual fallback. |
| PhilHealth API changes mid-project | LOW | Isolate eClaims client behind interface; versioned endpoints. |

## 16. Pre-Development Blockers

Must be resolved before Phase 2 begins:

1. PhilHealth eClaims SDK and XML schemas obtained from hospital's PhilHealth liaison.
2. Production server hardware ordered and timeline confirmed.
3. Three developers contracted with role assignments.
4. Hospital stakeholder weekly demo slot booked.
5. Master data sources identified: ICD-10 master, CPT codes, hospital drug formulary, item catalog, rates.
6. Audit of any existing hospital systems and data migration requirements.
7. PhilHealth digital signing certificate issued to the hospital.
8. Hospital IT team confirms network capacity (LAN, internet, UPS).
9. DPA-compliant consent forms drafted with hospital legal.
10. Defined rollback plan if go-live fails.

## 17. Open Questions

- Exact PhilHealth eCS XML schemas (to confirm with liaison).
- Existing data migration scope and format.
- Hospital's preferred receipt printer models (affects print template design).
- PhilHealth digital cert storage: HSM vs encrypted file.
- Backup retention policy (how many years).
- Disaster recovery RTO/RPO targets without cloud mirror.
- UPS runtime targets during extended brownouts.

---

## Appendix A — Module Communication Example

**Incorrect (direct DB access across modules):**

```php
// In Billing\Controllers\CashierController
$db = \Config\Database::connect();
$lab = $db->query("SELECT * FROM lis_orders WHERE ...")->getRow();
```

**Correct (via service class):**

```php
// In Billing\Controllers\CashierController
$labService = service('LIS\LabOrderService');
$charges = $labService->getChargesForEncounter($encounterId);
```

## Appendix B — Environment Variables (sample)

```env
APP_ENV=production
APP_DEBUG=false
APP_TIMEZONE=Asia/Manila

DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=ihomes
DB_USER=ihomes_app
DB_PASS=...

REDIS_HOST=127.0.0.1
REDIS_PORT=6379

PHILHEALTH_MODE=production
PHILHEALTH_ENDPOINT=https://eclaims.philhealth.gov.ph/eclaims
PHILHEALTH_HCI_CODE=...
PHILHEALTH_SENDER_ID=...
PHILHEALTH_CERT_PATH=/secure/certs/phic.pem
PHILHEALTH_CERT_PASS=...

BACKUP_B2_KEY_ID=...
BACKUP_B2_APP_KEY=...
BACKUP_B2_BUCKET=ihomes-backup-encrypted
```

## Appendix C — Sample Permission Codes

```
patient.view
patient.create
patient.update
patient.delete

er.triage.view
er.triage.create
er.consultation.create
er.consultation.update.own
er.consultation.update.any

lis.order.create
lis.result.enter
lis.result.release

billing.charge.post
billing.charge.override
billing.payment.receive
billing.payment.void

philhealth.claim.build
philhealth.claim.submit
philhealth.claim.override

admin.users.manage
admin.system.config
audit.view
```
