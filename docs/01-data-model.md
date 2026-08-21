# The data model

Three tables. That is the whole thing. Build them in about twenty minutes.

Column types are written generically — *text*, *choice*, *number*, *date/time*, *person*, *lookup*, *yes/no* —
so this maps cleanly onto SharePoint lists, Dataverse tables, or any comparable store.

---

## Table 1 — `Inspection`

The record of a thing being checked. One row per unit, batch, lot or delivery, whichever your process
treats as the unit of inspection.

| Column | Type | Notes |
|---|---|---|
| `Reference` | text | Your identifier — receipt number, batch, serial, work order. Unique. |
| `Item` | text | What was inspected. |
| `Source` | text | Supplier, work centre, or internal origin. |
| `Quantity_Received` | number | |
| `Quantity_Checked` | number | Gate 1 requires this. |
| `Result` | choice | `Accept` · `Reject` · `Use-as-is` · `Rework` |
| `Inspector` | person | Gate 1 requires this. Do **not** make it free text. |
| `Inspected_On` | date/time | **System-set** when the gate passes. Never user-typed. |
| `Stage` | choice | `Draft` · `Inspected` · `Dispositioned` · `Closed` — read-only to users; changed only by a gate. |
| `Nonconformance` | lookup → `Nonconformance` | Set when `Result` ≠ `Accept`. |
| `Notes` | multi-line text | |

## Table 2 — `Nonconformance`

Raised when an inspection fails. Carries the causal analysis.

| Column | Type | Notes |
|---|---|---|
| `NC_Reference` | text | Unique. |
| `Inspection` | lookup → `Inspection` | Back-link. |
| `Description` | multi-line text | What was actually observed — not what caused it. |
| `Severity` | choice | `Minor` · `Major` · `Critical` |
| `Cause_Level_1` | text | Why did it happen? |
| `Cause_Level_2` | text | Why did *that* happen? |
| `Cause_Level_3` | text | And why did *that* happen? |
| `Raised_By` | person | System-set. |
| `Raised_On` | date/time | System-set. |
| `Containment` | multi-line text | What you did immediately. |

Three causal levels is a deliberate minimum, not a ceiling. Add `Cause_Level_4` and `_5` if your customers
expect a full five-why. Do not reduce it below three — two levels almost always stops at the symptom.

## Table 3 — `CorrectiveAction`

What you are changing so it does not recur. Separate from the non-conformance because one cause can
generate several actions, and because actions outlive the record that prompted them.

| Column | Type | Notes |
|---|---|---|
| `CA_Reference` | text | Unique. |
| `Nonconformance` | lookup → `Nonconformance` | |
| `Action` | multi-line text | |
| `Owner` | person | Gate 3 requires this. |
| `Due` | date | Gate 3 requires this. |
| `Completed_On` | date/time | System-set on completion. |
| `Effectiveness_Check` | multi-line text | What evidence shows it worked. |
| `Approved` | yes/no | The approver's own act, and the thing a rule can be anchored on. Defaults to No. |
| `Approved_By` | person | Gate 3 requires this **and** requires it to differ from `Raised_By` and from `Owner`. |

---

## Relationships

```
Inspection  1 ──── 0..1  Nonconformance  1 ──── 0..*  CorrectiveAction
```

An inspection may have no non-conformance. In the core model a non-conformance belongs to an inspection —
`examples/nonconformance-intake.md` relaxes that, so a non-conformance can stand alone or be raised against a
production operation. A non-conformance may carry several corrective actions.

## Two notes that matter more than they look

**Use a person/user column, not text, for anyone who signs anything.** A typed name is not attribution;
a resolved directory identity is. It also lets a gate compare two people, which is how Gate 3 blocks
self-approval.

**Never create a user-editable date for when something happened.** Every date in this model that carries
evidentiary weight — `Inspected_On`, `Raised_On`, `Completed_On` — is set by the platform at the moment the
transition occurs. `Due` is the only date a user types, and it is a plan rather than a record.

**Column types are chosen for the model, not for a platform, and one of them costs you on Microsoft Lists.**
A *multi-line text* column cannot be read by a SharePoint list validation formula at all, which affects
`Description`, `Containment`, `Action`, `Effectiveness_Check` and `Notes`. If a gate in your build has to
test one of those for emptiness, either make it a single line of text, or have a flow write a companion
length column that validation can read. `docs/09-microsoft-lists-build.md` §1 sets out the three options and
§3 shows which conditions it affects.

**Route A adds three kinds of machinery column** that are not part of the model and should be hidden from
every form: plain-text shadows of the person columns (`Inspector_Email`, `Raised_By_Email`, `Owner_Email`,
`Approved_By_Email`), plain-text or frozen copies of anything a formula needs but cannot reach — a lookup's
key (`NC_Reference_Text`), a parent's value (`Operation_Released_On`), a value that must stop changing
(`Check_Frequency_At_Release`) — and flow-maintained flags or counters standing in for cross-list conditions
(`Closure_Ready`, `InProcess_Check_Count`). They exist because validation formulas cannot read person
columns, lookups, or another list. `docs/09-microsoft-lists-build.md` §2 and §3 explain what each one is for,
and its §4 lists the flows that write them.

**One rule governs all of them, and getting it wrong is the most expensive mistake in this kit.** A machinery
column is written by a flow *after* the item is saved, so it is blank for a moment on every save and
permanently blank on any record whose corrective write the rules themselves refused. Never write a gate whose
condition is satisfied by a machinery column being absent — anchor every rule on a column the user set, and
have it *require* the machinery column rather than excuse it.
`docs/09-microsoft-lists-build.md` §3 formula 3 works through the failure in full.

---

## Extensions the worked examples add

Three tables are the whole core. Two of the worked examples extend it, and both extensions are optional —
nothing here depends on them.

| Where | What it adds | Why |
|---|---|---|
| `examples/nonconformance-intake.md` | `Source`, `Quantity_Affected`, `Cause_Level_4`/`_5`, a five-value `Stage` and four transition timestamps on `Nonconformance`; `Verified_On` on `CorrectiveAction`; `Inspection` becomes optional | So a non-conformance can stand alone rather than hanging off an inspection — most of them do not come from incoming inspection |
| `examples/production-hard-gate-checklist.md` | Two further tables, `Operation` and `ProductionCheck` | In-process production checks are a different record with a different failure mode, and forcing them into `Inspection` distorts both |

Build the three core tables first and get them working. Add either extension only when you are building the
example that needs it — with one cross-dependency worth knowing before you start.
**`examples/production-hard-gate-checklist.md` Gate 3 requires `Nonconformance.Stage`**, which is added by
the `examples/nonconformance-intake.md` extension rather than by the production one. If you are building the
production example and want that gate, take the five-value `Stage` column and its transition timestamps from
the non-conformance extension as well. Everything else in the production example stands alone.

**A few column names are reused across tables with different meanings.** Nothing breaks, but know it before
you write a query that joins them: `Result` is a disposition on `Inspection` (`Accept` · `Reject` ·
`Use-as-is` · `Rework`) and a pass/fail on `ProductionCheck`; `Source` is a supplier or work centre on
`Inspection` and a detection origin on the extended `Nonconformance`; `Verified_On` is the effectiveness
evidence timestamp on `CorrectiveAction` and a stage-transition timestamp on `Operation`. If that ambiguity
would bite in your reporting, rename them in your own build — none of the gates depend on the names.
