# Design Doc: Rx Care Plan Generator

**Status:** Day 1 draft — requirement clarified, no tech stack decided yet
**Last updated:** 2026-11-20

## 1. Problem Statement

A specialty pharmacy's pharmacists currently spend 20–40 minutes per patient manually writing care plans. This is required for:
- Compliance
- Reimbursement from Medicare (for Medicare-covered patients) and from drug manufacturers (pharma) who require documentation of pharmacist clinical services

The pharmacy is short-staffed and backlogged on this task. We need a tool that lets pharmacists input patient/medication data and automatically generates a care plan via LLM.

## 2. Users

- **Primary user:** CVS pharmacists at the counter. They input clinical data and generate/print the care plan for the patient.
- **Patients:** Do **not** interact with the system directly. They receive a printed care plan from the pharmacist.
- **Out of scope as direct users:** physicians (referring providers), insurance reviewers — they are referenced by data, not system users.

## 3. Core Domain Concepts

| Term | Meaning |
|---|---|
| **Patient** | The person receiving the medication. Identified uniquely by MRN (Medical Record Number) within this pharmacy's system. |
| **Referring Provider** | The physician who diagnosed the patient and referred them to this pharmacy for the medication. Identified uniquely by NPI (National Provider Identifier, government-issued, 1:1 with a provider). |
| **Order** | A request for one medication for one patient. |
| **Care Plan** | A pharmacist-authored clinical document generated for **one order (one medication)** — not one per patient. A patient on multiple medications has multiple care plans. |

## 4. Inputs

### 4.1 Structured fields (confirmed from requirement doc)
- Patient First Name (string)
- Patient Last Name (string)
- Patient MRN (unique 6-digit number)
- Referring Provider (string)
- Referring Provider NPI (10-digit number)
- Patient Primary Diagnosis (ICD-10 code)
- Additional Diagnosis (list of ICD-10 codes)
- Medication Name (string)
- Medication history (list of strings)
- Patient Records (string OR PDF document) — free-text/unstructured clinical details (e.g. sex, weight, allergies, DOB, recent history) that aren't broken into their own fields in the original requirement

### 4.2 Open question (not yet resolved with customer)
The example patient record includes fields not listed as required inputs (Sex, Weight, Allergies, DOB). **Decision needed:** should these become their own structured fields, or stay inside the free-text `Patient Records` field? This affects the DB schema (dedicated columns vs. a single text/blob field) and will be revisited once we get to database design.

## 5. Output: Care Plan Structure

Fixed structure, required for every care plan regardless of medication:
1. **Problem list / Drug therapy problems (DTPs)**
2. **Goals (SMART)**
3. **Pharmacist interventions / plan**
4. **Monitoring plan & lab schedule**

Delivered as a downloadable text file.

## 6. Duplicate Detection Rules

| Scenario | Handling | Rationale |
|---|---|---|
| Same patient + same medication + **same day** | ❌ ERROR — block | Virtually certain duplicate submission |
| Same patient + same medication + **different day** | ⚠️ WARNING — confirm to proceed | Could be a legitimate refill |
| MRN match, name or DOB differs | ⚠️ WARNING — confirm to proceed | Possible data entry error |
| Name + DOB match, MRN differs | ⚠️ WARNING — confirm to proceed | Possibly the same person under a different MRN |
| NPI match, provider name differs | ❌ ERROR — must be corrected, no override | NPI is a unique national identifier; mismatch is always a data error |

**ERROR** = cannot be a legitimate case, must block.
**WARNING** = could be legitimate, pharmacist confirms and proceeds.

## 7. Functional Requirements (Phase 1 — all must-have, no deferred features)

- [ ] Web form for pharmacist to input patient/order data
- [ ] Input validation (NPI format, MRN format, ICD-10 format, etc.)
- [ ] Patient duplicate detection
- [ ] Order duplicate detection
- [ ] Provider duplicate detection (one NPI = one provider, enforced)
- [ ] Care plan generation via LLM call
- [ ] Care plan download (text file)
- [ ] Export for pharma reporting

## 8. Production-Readiness Requirements

- Every input is validated
- Integrity rules always enforce consistency (duplicate/provider rules above)
- Errors are safe, clear, and contained — no stack traces or PHI exposed to the user
- Code is modular and navigable
- Critical business logic covered by automated tests
- Project runs end-to-end out of the box (Docker)

## 9. Explicitly Out of Scope (Phase 1)

- Patients as direct system users
- Physician-facing interface
- Insurance/payer-facing interface (Medicare, pharma are reimbursement context, not system integrations)

## 10. Open Questions for Customer

1. Should Sex/Weight/Allergies/DOB be dedicated structured fields, or remain inside `Patient Records` free text?
2. Is "Referring Provider" in the input fields the same entity as "Provider" in the duplicate-detection rules? (Assumed yes — same entity, different label in different sections of the doc.)

## 11. Tech Stack

Not decided yet — intentionally deferred. Per the course design, each technology is introduced only once its corresponding pain point is hit (e.g. message queue once synchronous LLM calls become a blocking problem). Will be documented here incrementally as decisions are made.
