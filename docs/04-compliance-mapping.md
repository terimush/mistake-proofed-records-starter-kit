# Compliance mapping

Which obligations this pattern helps you evidence — and, just as importantly, which it does not.

**Read the boundaries section first.** A mapping document that overstates its reach is worse than none,
because it encourages you to stop looking for the gaps.

**And read this before you show the table to anyone.** Everything below describes what the pattern
contributes *when the gates are actually enforced*, which means the conditions live on the data rather than
in the form. On Route B (Dataverse) that is straightforward. On Route A (Microsoft Lists) it is achievable
and commonly not achieved: gate conditions written as list validation formulas are enforced by the server on
save and hold whichever client wrote the record, but the same conditions written into a form are worth
nothing. `docs/09-microsoft-lists-build.md` shows the difference.

**Two rows below are weaker on Route A whatever you do, and they are named here rather than left for you to
find.** The **7.5.2 / 7.5.3** row says earlier stages lock on gate passage: on Route A they do not, because
there are no column-level permissions, and the only lock available is the whole record at closure. The
**8.7** row rests on conditions that span records — an inspection's disposition against its non-conformance
— which on Route A are enforced through a flag a flow maintains rather than by a formula reading the other
list directly; the refusal is real, the evaluation behind it is automated rather than declarative, and an
auditor may reasonably want to see the flag's log.

Run `docs/06-validation-and-test-plan.md`, establish which of these is true of your build, and qualify the
table accordingly. This is the document most likely to be pasted into a supplier questionnaire, which is
exactly why it should not be the one that overstates.

---

## ISO 9001:2015

This is where the fit is genuine. The pattern produces the kind of evidence these clauses ask for.

| Clause | What it requires | How the pattern helps |
|---|---|---|
| **7.5.2 / 7.5.3** Documented information | Records created, identified, controlled, protected from unintended alteration | Records are created in-process; a record cannot reach a stage without the evidence that stage requires, on every write path; closed records are locked; the platform supplies version history and access control. **Route A:** earlier stages do *not* lock on passage — only the whole record, at closure |
| **8.4.2** Control of external providers | Verification that purchased product meets requirements | The `Inspection` table *is* that verification record, per receipt |
| **8.5.1** Control of production and service provision | Controlled conditions, including monitoring and verification at appropriate stages | Gates are those verification points, and where the conditions are enforced on the data they cannot be skipped on either route |
| **8.5.2** Identification and traceability | Outputs identified; traceability where required | `Reference`, `Item` and `Source` carried on every record and linked forward to any non-conformance |
| **8.6** Release of products and services | Evidence of conformity, traceable to the person authorising release | `Result` plus a resolved `Inspector` identity plus a system timestamp |
| **8.7** Control of nonconforming outputs | Nonconforming output identified and controlled; action taken; records retained | The `Nonconformance` table, with disposition captured in `Result` |
| **10.2** Nonconformity and corrective action | React, evaluate the need for action, determine causes, implement, review effectiveness | Enforced causal depth — refused outright on both routes — plus `CorrectiveAction` with owner, due date and `Effectiveness_Check`, and an approval that cannot be given by the raiser or the owner |
| **9.1.3** Analysis and evaluation | Analyse data on conformity and on the effectiveness of the QMS | Structured records are queryable; the checks in `02-hard-gate-pattern.md` §5 are a starting analysis |

**What it does not cover in ISO terms.** Clauses 4 and 5 (context, leadership, policy), 6 (risk and
opportunity planning), 7.1.5 (calibration of monitoring and measuring resources), 7.2 (competence), 9.2
(internal audit) and 9.3 (management review). Those are organisational, not a records problem. A certificate
depends on them at least as much as on anything here.

**And a currency warning that applies to this table specifically.** Every clause reference above is to
ISO 9001:2015. The revision is at the publication stage and is expected in **September 2026**, so this table
will be describing a superseded edition within weeks of this document being published. Clause numbering for
documented information, nonconforming outputs and corrective action is not expected to move much, but do not
take that from here — read the current standard, or ask your certification body which edition your next audit
is against. `START-HERE.md` carries the same warning and the same date.

---

## CMMC and NIST SP 800-171

**Be careful here, and be conservative.** CMMC is a *cybersecurity* framework about protecting Federal
Contract Information and Controlled Unclassified Information. This kit is a quality-records pattern. It is
not a security control set and it does not implement one.

The honest position is narrow: **if** you store contract-related production records in this system, then the
platform underneath it — not this pattern — carries most of the relevant controls, and the pattern
contributes to a handful of practices around records and accountability.

| Area | Practice family | What actually contributes |
|---|---|---|
| Access control | **AC** | Inherited from the platform: who can see and edit records. On Route B `Stage` is not user-editable at all; on Route A it is writable but an invalid value is refused by validation |
| Identification & authentication | **IA** | Inherited: resolved directory identities rather than typed names, which is what makes attribution meaningful |
| Audit & accountability | **AU** | The pattern's contribution — system-generated timestamps, actions traceable to individuals, and closed records locked against edit by ordinary users (by column security on Route B, by item permissions at closure on Route A) |
| Media protection | **MP** | Inherited: records held in a controlled system rather than on paper and local drives |

**What this kit does not do, and you should not claim it does.** It does not perform system security
planning, incident response, configuration management, physical protection, personnel screening, media
sanitisation, boundary protection, encryption of CUI at rest or in transit, or any of the assessment
activities CMMC requires. It does not produce an SSP or a POA&M. It does not substitute for assessment by a
certified third-party assessment organisation, and no document you generate from it is evidence of CMMC
compliance.

If your contracts carry CMMC obligations, treat this as one small input to a records story inside a much
larger programme — and get proper advice on the rest.

---

## Before you rely on any of this

Standards get revised, contract clauses change, and flow-down requirements differ by customer and by
programme. Nothing in this table is a legal or contractual interpretation, and no third-party summary —
including this one — is a substitute for reading the standard your certificate is issued against and the
clauses in your own contracts.

Free help exists. **APEX Accelerators** (U.S. Department of Defense funded, formerly PTACs) counsel small
businesses on federal contracting obligations at no charge, with a centre serving every state. **MEP
centres** do the equivalent on the manufacturing side. Ask them before you ask a vendor.
