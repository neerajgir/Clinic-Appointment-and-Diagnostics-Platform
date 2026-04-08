# Clinic Appointment and Diagnostics Platform — Database Design (ER Diagram)

A database schema designed for a modern clinic that manages doctors, patients, appointments, consultations, diagnostic tests, reports, and payments — in a clean, scalable, and normalized structure.

---

## Project Overview

This is a database design task only. The goal was to model the full clinical workflow of a clinic — from a patient booking an appointment, to a doctor conducting a consultation, prescribing diagnostic tests, generating reports, and finally collecting payment.

The design supports:

- Managing doctors across departments and specialties
- Patients booking appointments with specific doctors
- Tracking appointment status (scheduled, completed, cancelled, no-show)
- Recording consultations only when an appointment is actually attended
- Prescribing one or more diagnostic tests during a consultation
- Generating diagnostic reports per test, linked back to the patient and visit
- Billing patients per consultation with itemized test charges

No SQL queries, frontend, or backend were built. This submission covers **database design only**.

---

## File Structure

```
/
├── clinic_erd_part1_core.png        # Patients, doctors & appointments
├── clinic_erd_part2_clinical.png    # Consultations, diagnostics, reports & payment
├── clinic_erd_eraserio.txt          # Full Eraser.io paste-ready ERD code
└── README.md                        # This file
```

> The ERDs were designed using Mermaid.js and are also available as Eraser.io code for visual diagram submission.

---

## How to View

### Option 1 — Mermaid.js (browser)
Open the exported HTML files in any modern browser. No install needed. Requires an internet connection on first load for the Mermaid.js CDN.

### Option 2 — Eraser.io (recommended for submission)
1. Go to [eraser.io](https://eraser.io) and create a new diagram
2. Choose **Entity Relationship** as the diagram type
3. Paste the code from `clinic_erd_eraserio.txt` into the left-side editor
4. The diagram auto-renders with all tables, attributes, icons, and relationship lines

---

## Entities

The schema is split into two logical layers.

### Layer 1 — People & Scheduling

| Entity | Description |
|---|---|
| `DEPARTMENT` | Clinic departments — Cardiology, Dermatology, General Medicine, etc. |
| `DOCTOR` | Doctor profile — specialty, qualification, contact info, linked to a department |
| `PATIENT` | Patient demographics — name, DOB, gender, blood group, contact |
| `APPOINTMENT` | A booking made by a patient with a doctor — tracks status and may or may not result in a visit |

### Layer 2 — Clinical Outcomes

| Entity | Description |
|---|---|
| `CONSULTATION` | The actual doctor visit — symptoms, diagnosis, prescription, notes. Only exists if the appointment was attended |
| `DIAGNOSTIC_TEST` | Master catalogue of available tests — CBC, X-ray, MRI, etc. — with category and base price |
| `TEST_PRESCRIPTION` | Junction record linking a test to a consultation — with urgency level and instructions |
| `DIAGNOSTIC_REPORT` | Result of one prescribed test — summary, detailed result, file URL, generation timestamp |
| `PAYMENT` | Bill for a consultation — itemizes consultation fee and test charges separately |

---

## Relationships

| Relationship | Cardinality | Notes |
|---|---|---|
| DEPARTMENT → DOCTOR | One to zero-or-more | A department has many doctors |
| PATIENT → APPOINTMENT | One to zero-or-more | A patient can book many appointments over time |
| DOCTOR → APPOINTMENT | One to zero-or-more | A doctor is assigned to many appointments |
| APPOINTMENT → CONSULTATION | One to zero-or-one | Only attended appointments produce a consultation record |
| CONSULTATION → TEST_PRESCRIPTION | One to zero-or-more | A consultation may prescribe zero or many tests |
| DIAGNOSTIC_TEST → TEST_PRESCRIPTION | One to zero-or-more | A test can be prescribed across many consultations |
| TEST_PRESCRIPTION → DIAGNOSTIC_REPORT | One to zero-or-one | Each prescribed test generates at most one report |
| CONSULTATION → PAYMENT | One to zero-or-one | Payment may still be pending after the visit |

---

## Joins Used

| Relationship | Join type | Reason |
|---|---|---|
| Doctor ↔ Department | `INNER JOIN` | Every doctor must belong to a department — no nulls |
| Appointment ↔ Patient / Doctor | `INNER JOIN` | Every appointment must have a valid patient and doctor |
| Appointment → Consultation | `LEFT JOIN` | Cancelled or missed appointments have no consultation row |
| Consultation → Test prescription | `LEFT JOIN` | Many consultations have no tests prescribed |
| Test prescription → Diagnostic test | `INNER JOIN` | Every prescription references a valid test in the catalogue |
| Test prescription → Report | `LEFT JOIN` | Reports may not be generated yet when queried |
| Consultation → Payment | `LEFT JOIN` | Payment may be pending — left join catches unpaid visits |

**General rule:** join inward toward a master/catalogue table with `INNER JOIN`. Join forward along the clinical workflow with `LEFT JOIN`, because each step is optional and may not have happened yet.

### Eraser.io relationship syntax

In Eraser.io, all relationships use the `>` operator — pointing from the FK side to the PK side:

```
doctors.department_id              > departments.id
appointments.patient_id            > patients.id
appointments.doctor_id             > doctors.id
consultations.appointment_id       > appointments.id
test_prescriptions.consultation_id > consultations.id
test_prescriptions.test_id         > diagnostic_tests.id
diagnostic_reports.prescription_id > test_prescriptions.id
payments.consultation_id           > consultations.id
```

The `>` always points from the "many" table to the "one" table. Inner vs left join is a query-time decision — the diagram line symbol is the same for both.

---

## Key Design Decisions

### Appointment and consultation are separate entities

An appointment is a booking — it tracks status (`scheduled`, `completed`, `cancelled`, `no_show`). A consultation is the actual clinical event and only exists when the appointment was attended. The two are linked one-to-zero-or-one using a `LEFT JOIN` in queries. This is the most important distinction in the design: a naive schema would store everything in one table, losing all data about missed or cancelled visits.

### Diagnostic tests belong to the consultation, not the appointment

Tests are prescribed during the doctor's examination inside the consultation. Linking them to the appointment would be incorrect — the doctor has not yet seen the patient at that point. `TEST_PRESCRIPTION` is the junction record between `CONSULTATION` and `DIAGNOSTIC_TEST`.

### DIAGNOSTIC_TEST is a catalogue, not a prescription record

`DIAGNOSTIC_TEST` stores the master list of available tests with base prices. It is reused across thousands of consultations. `TEST_PRESCRIPTION` is the actual event — it links a test to a specific consultation with urgency and instructions for that particular patient.

### DIAGNOSTIC_REPORT links to the prescription, not the consultation

A report is the result of one specific test prescription. Linking it to `TEST_PRESCRIPTION` means every report traces back to the exact instruction, doctor, and visit that generated it. The `patient_id` and `test_id` fields on `DIAGNOSTIC_REPORT` are intentional denormalized shortcuts, enabling fast queries like "all reports for patient X" without multi-table joins every time.

### Payment belongs to the consultation, not the appointment

Charges are generated by the clinical visit, not the booking. The payment record stores `consultation_fee` and `test_charges` as separate fields so the clinic can report on clinical revenue vs diagnostic revenue independently.

### Department as a separate entity

Doctor specialty is a simple attribute on the doctor (one doctor, one specialty). Department is a separate entity because multiple doctors share a department and the clinic may need to query, report, or manage departments independently.

---

## Evaluation Coverage

| Criteria | How it is addressed |
|---|---|
| Workflow understanding | Full appointment → consultation → test prescription → report chain modelled correctly |
| Entity identification | All 9 entities present — department, doctor, patient, appointment, consultation, diagnostic test, test prescription, report, payment |
| Relationship modelling | All 1:N and 1:0-or-1 relationships defined with correct cardinality and join logic |
| PK / FK quality | Every entity has a PK; all FKs on the correct side; denormalized FKs on report are explained |
| Attribute design | Attributes separated by concern; no bloated or mixed-purpose entities |
| Business logic accuracy | Appointment vs consultation separation; tests in consultation not appointment; payment tied to the visit |
| Diagram presentation | Split into two focused layers — each diagram stays readable with no overcrowding |

---

## Tools Used

- **Eraser.io** — paste-ready ERD code for visual diagram submission
