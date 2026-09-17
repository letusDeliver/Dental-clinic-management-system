# Business Rules Catalogue

Dedicated catalogue of domain rules, cross-referenced from the module
requirement docs rather than duplicated into them. Every question below
is explicitly marked **RESOLVED** (with the answer and rationale) or
**UNRESOLVED** (needs clinic/stakeholder input before the relevant module
is implemented) — none are answered silently.

Format: question → status → answer/rationale → owning module.

---

## Appointments & Scheduling

**Can a patient have multiple appointments on the same day?**
RESOLVED — Yes. No system-enforced limit; a patient may need consultation
+ a separate procedure slot. Reception can use judgment. Owning module:
[appointments.md](requirements/appointments.md).

**Do cancelled appointments free their slot?**
RESOLVED — Yes, immediately. Enforced via a partial unique index that
excludes `CANCELLED` rows (see appointments.md Concurrency section).

**Can completed appointments be edited?**
RESOLVED — No. `COMPLETED` is terminal. Corrections to what happened
during a visit are made on the `Visit` record (visits.md), not by
reopening the appointment.

**Who can reschedule / cancel an appointment?**
RESOLVED — ADMIN and RECEPTIONIST for any appointment; DOCTOR only for
their own. See the permission matrix in
[staff-auth.md](requirements/staff-auth.md). Patients cannot self-
reschedule/cancel via API in V1 (no patient portal) — they must call the
clinic.

**Can an appointment exist without a patient?**
RESOLVED — No. `Appointment.patientId` is required; a walk-in with no
prior record gets a minimal `Patient` created first (see Patients below).

**Can a patient book multiple future appointments in advance?**
RESOLVED — Yes, no cap enforced in V1. Revisit if abuse/no-show patterns
emerge (would be a Change Control decision, not silent enforcement).

**What happens when the clinic is closed / on a holiday?**
RESOLVED — No slots are generated for that day; `clinic-config.md` is the
source of truth for hours/holidays and `appointments.md` derives
availability from it.

**What happens if the doctor is unavailable (leave, etc.)?**
UNRESOLVED — V1's `DoctorAvailability` (in clinic-config.md) models
recurring weekly hours only. One-off doctor unavailability (sick day, half
-day leave) has no dedicated entity yet. **Needs clinic input**: is an
ad-hoc "block this doctor's time" feature needed for V1, or is manually
declining/not-generating slots for that day an acceptable manual
workaround initially? Tracked until resolved.

**Can walk-in patients be created?**
RESOLVED — Yes. RECEPTIONIST/ADMIN can create a `Patient` and an
`Appointment` in the same flow without going through public booking.

**Can staff create appointments manually (not via public booking)?**
RESOLVED — Yes, ADMIN/RECEPTIONIST, same underlying booking
service/concurrency guarantee as public booking.

**What happens when a patient does not show up?**
RESOLVED — Staff (RECEPTIONIST/ADMIN) manually marks `NO_SHOW`; there is
no automatic timeout-based transition in V1 (would require a background
job, not justified yet — see architecture-context.md non-goals).

**Is appointment confirmation (BOOKED → CONFIRMED) automatic or manual?**
UNRESOLVED — See `appointments.md` state machine note. **Needs clinic
input**: does reception want a manual "confirm by phone call" step, or
should public bookings auto-confirm and `CONFIRMED` only exist for a
future SMS/WhatsApp-confirm-reply flow? Until resolved, default behavior
is: public bookings start `CONFIRMED` automatically (no manual step
required), and the `BOOKED` state is reserved for a case explicitly
flagged "pending confirmation" if the clinic later asks for one.

---

## Patients & Visits

**Can a patient have treatment without a visit?**
RESOLVED — No. `PatientTreatment` requires a `visitId` (treatments.md) —
every treatment is administered in the context of a recorded visit, which
is itself the clinical audit trail.

**Can a visit exist without an appointment?**
RESOLVED — Yes. Walk-in emergencies or phone-triaged same-day care may
need a `Visit` without a prior `Appointment` — `Visit.appointmentId` is
nullable (visits.md).

**Can receptionist modify medical records (notes/diagnosis)?**
RESOLVED — No. RECEPTIONIST can view/manage patient contact info and
appointments, but `Visit`, `Diagnosis`, and `Prescription` writes are
DOCTOR/ADMIN only (permission matrix in staff-auth.md).

**Can compounder see diagnosis?**
RESOLVED — No, not the free-text/clinical detail. COMPOUNDER's patient
view is demographics + today's procedure type only (patients.md /
staff-auth.md matrix), consistent with least privilege.

**Can doctor modify payment information?**
RESOLVED — No. DOCTOR has read-only access to billing (billing.md
permission row) — separates clinical and financial responsibility.

---

## Payments & Invoicing

**Can a payment be partially paid?**
RESOLVED — Yes. `Invoice.amountPaid` accumulates across multiple
`Payment` rows; status derives to `PARTIALLY_PAID` until `amountPaid ==
total` (billing.md).

**Can invoices be edited after a payment has been recorded?**
RESOLVED — No. Line items become immutable once any payment exists;
corrections go through voiding + reissue (billing.md Business Rules).

---

## Cross-Cutting

**Can an appointment/visit/invoice be hard-deleted?**
RESOLVED — No. Nothing clinical or financial is hard-deleted anywhere in
the system; soft-delete (`deletedAt`) or a terminal status (`CANCELLED`,
`VOID`) is used instead, to preserve audit and financial history.

**What is the appointment slot duration?**
UNRESOLVED — Modeled as configurable per doctor
(`DoctorAvailability.slotMinutes`, default 30) in clinic-config.md, but
the clinic's actual real-world slot length (per doctor, possibly per
treatment type) needs confirmation before Foundation/Appointments
implementation locks in a default.

**What is the clinic's operating timezone?**
UNRESOLVED — Needed as a real config value before Appointments is
implemented (already tracked in [progress-tracker.md](../progress-tracker.md)).

---

## How to Use This File

- When implementing a module, check this file for rules relevant to that
  module's scope before inventing an answer.
- If an **UNRESOLVED** item blocks implementation, surface it for a
  decision (per `ai-workflow-rules.md` Architectural Conflicts) rather
  than guessing — then move it to RESOLVED here with the answer and
  update the owning module's requirement doc.
- Do not let this file grow into a duplicate of the requirement docs —
  only rules that are genuinely ambiguous or cross-cutting belong here;
  module-local rules stay in that module's doc.
