# Worked example — in-process production hard-gate checklist

A production checklist that cannot be completed at the end of the shift. Fabricated throughout; no real
supplier, part, customer or organisation appears anywhere in this kit.

This is the third worked example, and the hardest of the three to get right — not technically, but because
it is the one that touches people who are being paid to make parts rather than to fill in records. Read
`docs/05-rollout-runbook.md` before you build it.

---

## The problem this one solves

Incoming inspection happens at a desk, once per delivery, by someone whose job includes paperwork. In-process
production checks happen at a machine, several times a run, by someone whose job is the machine. That
difference is why production checklists are the ones that get completed retrospectively — the checks
genuinely happened, and the record of them was created on Thursday afternoon from a pad of notes and a
reasonable memory.

A record created on Thursday afternoon is not evidence that a check happened on Tuesday. It is a statement
that somebody believes it did. The two look identical in a filing cabinet and completely different under the
distribution check in `docs/02-hard-gate-pattern.md` §5.

The gate here is therefore not about what is in the record. It is about **when the record came into
existence relative to the work**.

## The process being modelled

A work order runs through one or more operations. Each operation has a first-off check before the run is
released, in-process checks at a defined frequency during it, and a last-off check before the operation is
signed off. The operation cannot be released without a passing first-off. It cannot be signed off without
the required number of in-process checks and a last-off. The work order cannot be marked complete until
every operation on it is signed off.

```
Operation:   Ready ──▶ Running ──▶ Verified ──▶ SignedOff
                  gate 1      gate 2       gate 3

Work order:  Open ────────────────────────────▶ Complete
                            gate 4
```

## What changes in the data model

This adds a fourth and fifth table to the three in `docs/01-data-model.md`. They are optional — nothing in
the other two worked examples needs them.

### Table 4 — `Operation`

One row per operation on a work order. This is the record that carries the stage and the gates.

| Column | Type | Notes |
|---|---|---|
| `Operation_Reference` | text | Unique. Usually work order plus operation number. |
| `Work_Order` | text | Your ERP's work order identifier, so the two reconcile without a lookup table. |
| `Item` | text | What is being made. |
| `Operation_Number` | number | Sequence within the work order. |
| `Check_Frequency` | number | How many parts between required in-process checks. Set from the process, not by the operator. |
| `Quantity_Planned` | number | |
| `Quantity_Made` | number | Entered at last-off. |
| `Required_Checks` | number | **Calculated**, not typed — see the note below. Gate 2 compares against it. Zero is a legitimate value. |
| `Operator` | person | Resolved directory identity. Not free text. |
| `SignedOff_By` | person | Gate 3 requires it, and requires it to differ from `Operator` and from every checker. |
| `Stage` | choice | `Ready` · `Running` · `Verified` · `SignedOff`. Read-only to users on Route B; **user-set on Route A**, where validation refuses any item illegal at the stage it claims — see `docs/09-microsoft-lists-build.md` §4. |
| `Released_On`, `Verified_On`, `SignedOff_On` | date/time | Each **system-set** when its gate passes. |
| `Check_Frequency_At_Release` | number | **Route A machinery.** The flow copies `Check_Frequency` here when Gate 1 passes, and `Required_Checks` is computed from this rather than from the live column — see Gate 1. |
| `Nonconformance` | lookup → `Nonconformance`, **multi-value** | Set when any check fails. Multi-value, because one operation can raise more than one — Gate 3 says "all linked non-conformance records" and means it. |

### Table 5 — `ProductionCheck`

One row per check actually performed. This is the table the whole example is about.

| Column | Type | Notes |
|---|---|---|
| `Check_Reference` | text | Unique. |
| `Operation` | lookup → `Operation` | |
| `Check_Type` | choice | `First-off` · `In-process` · `Last-off` |
| `Characteristic` | text | What was measured or verified. |
| `Value` | text | The reading, or the pass/fail for an attribute check. |
| `Result` | choice | `Pass` · `Fail` |
| `Checked_By` | person | Resolved directory identity. |
| `Checked_On` | date/time | **System-set at creation.** This is the single most important column in the example. |
| `Part_Count_At_Check` | number | Parts made at the moment of the check. Entered by the operator. |
| `Nonconformance` | lookup → `Nonconformance` | Set on a failing check. Gate 2 tests this; without it there is nothing for "a failed check has no non-conformance against it" to read. |
| `Operation_Released_On` | date/time | **Route A machinery.** The flow copies the parent operation's `Released_On` here at creation, which turns the anti-retrospective condition into a rule about one row — see Gate 2. Blank on a first-off, because the operation has not been released yet, which is why the condition excludes them. |

**On Microsoft Lists, keep the actual count on the record too.** A gate that counts rows in another list is
the hardest thing in this example to build correctly — a canvas app cannot count a large list reliably, and
list validation cannot see a second list at all. Have the flow maintain an `InProcess_Check_Count` column on
`Operation`, incremented **only when a check with `Check_Type = In-process` is created**, and decremented if
one is deleted. Counting every check instead reads three when one in-process check was performed, and the
gate then passes an operation that was never checked — which is worse than having no gate, because the number
looks like evidence. The comparison then happens between two numbers on one record, which validation can
enforce and no query can get wrong. `docs/10-canvas-app-gates.md` §5 step 5 has the technique and the
reconciliation job that keeps the counter honest.

**`Required_Checks` must be calculated, never typed.** Set it as

```
Required_Checks = max(0, ceiling(Quantity_Made ÷ Check_Frequency) − 1)
```

evaluated at Gate 2. If the operator can type the number of checks their own operation requires, the gate
enforces nothing — they will type the number they have. Define it in exactly one place and have Gate 2
reference the column, rather than restating the arithmetic inside the rule where the two can drift apart.

As a SharePoint calculated column that is
`=MAX(0, CEILING([Quantity_Made]/[Check_Frequency], 1) - 1)` — note that `CEILING` there takes a second
argument, the significance, and that `Check_Frequency` of zero or blank produces a division error rather than
a zero. Gate 1 requires `Check_Frequency > 0` for exactly that reason, so keep those two conditions together
when you adapt them.

**And know what the calculation still rests on.** `Quantity_Made` is typed by the operator at last-off, and
on Route A there are no column-level permissions, so `lock Quantity_Made` at Gate 2 is not achievable. An
operator who made a hundred parts, performed no in-process checks, and enters `Quantity_Made = 25` computes a
requirement of zero and passes the gate — with a system-set `Verified_On` on top, which makes it look
checked. The coverage gate is only ever as good as that number. Two mitigations, and use both: have the flow
compare `Quantity_Made` against the `Part_Count_At_Check` on the last-off row and log a mismatch, and put the
count on the dashboard where a supervisor sees it against planned quantity. Neither is a refusal. Say so.

**`Checked_On` must be set at creation, not at save-and-close.** On some low-code platforms a record can be
opened, left, and committed later, which would stamp the time the operator finished typing rather than the
time they checked the part. If your platform does that, stamp the value in the rule that creates the record.

## Stages and gates

### `Ready` → `Running` · Gate 1 — first-off

```
REQUIRE  at least one ProductionCheck exists with Check_Type = First-off
     AND the most recent First-off check has Result         = Pass
     AND the most recent First-off check has Checked_By     not blank
     AND the most recent First-off check has Characteristic not blank
     AND Operator                is not blank
     AND Check_Frequency         is not blank AND > 0

ON PASS  set Released_On = now()
         lock Check_Frequency
```

Refusal message: *"This operation needs a passing first-off check before the run starts. Record what you
measured and what you got."*

**A failing first-off does not block the gate by being absent — it blocks by failing.** If the first-off
result is `Fail`, the operation stays in `Ready` and a non-conformance is raised. Do not let a failed
first-off be deleted and re-entered; that is the exact behaviour the pattern exists to prevent. If your
platform allows the operator to delete their own records, this gate is decorative no matter how the
condition is written.

**This is why the condition reads "at least one" and "the most recent", not "exactly one".** A failed
first-off is retained, the setup is corrected, and a second first-off is recorded — so the table legitimately
holds two. A rule demanding exactly one would leave the operation unreleasable forever the first time
somebody found a genuine problem at setup, which is a spectacular way to teach people to delete records.

**`Check_Frequency` locks at release** for the same reason `Required_Checks` is calculated. A frequency that
can be relaxed at the end of the run to match the number of checks performed is not a frequency.

**On Route A that lock is not available, and the consequence is worse than it looks.** There are no
column-level permissions, so nothing stops the frequency being edited after release — and because
`Required_Checks` is calculated from it, raising the frequency at the end of a run retroactively *lowers* the
number of checks the run required. The row stays legal, the gate passes, and no rule was broken: an operator
who made a hundred parts at a frequency of 25 and performed one check can set the frequency to 100 and owe
none. It is the same defeat as understating `Quantity_Made` and it is quieter, because nobody looks at a
frequency twice.

The fix is a copy, not a permission. Have the flow stamp `Check_Frequency_At_Release` on the operation when
Gate 1 passes, and compute `Required_Checks` from *that* column rather than from the live one. It is then a
value written by the flow at a moment the operator does not control, and a later edit to `Check_Frequency`
changes nothing the gate reads. Add the pair to the reconciliation job so a divergence is reported rather
than merely tolerated.

### `Running` → `Verified` · Gate 2 — in-process coverage

This is the gate that carries the example.

```
REQUIRE  Quantity_Made  is not blank
     AND count(ProductionCheck where Check_Type = In-process)  >=  Required_Checks
     AND at least one ProductionCheck exists with Check_Type = Last-off
     AND every ProductionCheck where Check_Type in {In-process, Last-off}
             has Checked_On  >  Operation.Released_On
     AND no ProductionCheck has Result = Fail without a linked Nonconformance

ON PASS  set Verified_On = now()
         lock Quantity_Made
```

Refusal messages, separately:
*"This run needs {Required_Checks} in-process checks for {Quantity_Made} parts at a frequency of
{Check_Frequency}. You have {n}."*
*"This operation needs a last-off check before it can be verified."*
*"A check was recorded before the run was released. Check the sequence."*
*"A failed check has no non-conformance against it."*

**The `− 1` is deliberate and you should understand it before you copy it.** A run of 100 parts at a
frequency of 25 needs checks at 25, 50 and 75 — the first-off covers the start and the last-off covers the
end. Getting this arithmetic wrong in either direction is the fastest way to make operators hate the system:
demand one check too many and every single operation refuses at the end of the run for no reason anyone can
see. Work it through against your own frequency convention before you build it, and put the required and
actual counts in the refusal message so the operator can see the arithmetic rather than guess at it.

**The `max(0, …)` is not decoration.** A run shorter than one frequency interval legitimately needs no
in-process checks, and a run that made nothing at all would otherwise compute a requirement of minus one —
which is harmless as a comparison and embarrassing in a refusal message. An aborted setup that made zero
parts should not be dragged through this gate anyway; give it a `Cancelled` stage with a reason, so it leaves
the open-work panel without pretending to be a completed operation.

**Note what the count does not check: spacing.** Three in-process checks all recorded at part 99 satisfy the
condition exactly. `Part_Count_At_Check` exists so you can close that gap if it matters to you —
`the in-process checks' Part_Count_At_Check values are spaced no more than Check_Frequency apart` — but think
before you add it. It is a real tightening and it will refuse legitimate runs where an operator checked early
because the machine sounded wrong, which is behaviour you want to encourage rather than punish.

**Which layer enforces this one, because the example turns on it.** `Checked_On` is on `ProductionCheck` and
`Released_On` is on `Operation`, so as written it spans two lists and is a validation formula nowhere. Use the
same copy-down move as everywhere else in this kit: have the flow that creates a check stamp the parent's
`Released_On` onto the check row as `Operation_Released_On`, and the condition becomes
`[Checked_On] > [Operation_Released_On]` — one row, one list, refused by the server on every write path. Do
not leave it as a flow check if you can avoid it; this is the condition the example exists for and it should
be the one you can prove hardest.

**`Checked_On > Released_On` is the anti-retrospective condition.** It is the reason this example exists.
An in-process or last-off check cannot have happened before the run it belongs to was released. The first-off
is deliberately excluded, because it necessarily precedes release — it is the evidence Gate 1 needed in order
to release at all, and including it in this condition would make the gate unsatisfiable. This costs one
comparison between two system-set values and it defeats the bulk-entry-on-Thursday failure entirely — not
because it detects
dishonesty, but because it makes the honest late entry visibly impossible, which is the conversation you
want to have with a supervisor rather than an auditor.

**What this condition does not catch**, and you should say so out loud when you present this: checks entered
in bulk *during* the run, twenty minutes before the end. The timestamps are all after release and all
plausible. The distribution and dwell checks in `docs/02-hard-gate-pattern.md` §5 are what catch that, and
they are analytical rather than preventive. No gate can distinguish a check recorded promptly from a check
recorded promptly-ish.

### `Verified` → `SignedOff` · Gate 3 — independent sign-off

```
REQUIRE  SignedOff_By          is not blank
     AND SignedOff_By          <> Operator
     AND SignedOff_By          <> every ProductionCheck.Checked_By
     AND all linked Nonconformance records are at Stage = Closed
             OR carry a documented concession reference

ON PASS  set SignedOff_On = now()
         lock the entire operation record and all its checks
```

Refusal messages, separately:
*"The person signing off cannot be the person who ran the operation or performed the checks."*
*"This operation has an open non-conformance. Close it, or record the concession."*

**This gate needs a column the core model does not have.** `Nonconformance.Stage` is added by the
`examples/nonconformance-intake.md` extension, not by this one. If you want the last condition, take that
extension's five-value `Stage` and its transition timestamps as well — `docs/01-data-model.md` notes the
dependency. There is no concession field in the model either; if you use the concession branch, add one, and
make it a reference to a document rather than a free-text box.

**This is the gate most shops will want to relax, and the one to think hardest about before relaxing.** On a
night shift with two people, `SignedOff_By <> Operator` may be genuinely impossible some of the time. The
honest options are to accept a documented exception with a named approver and a reason — recorded as a
field, not as a habit — or to weaken the condition and say so plainly in what you tell an auditor. What is
not honest is leaving the condition in place and letting people share a login to satisfy it. If you see
shared logins appear after you build this, that is the system telling you the condition does not fit the
shift pattern.

### `Open` → `Complete` · Gate 4 — the work order

```
REQUIRE  every Operation on this Work_Order  is at Stage = SignedOff

ON PASS  set Completed_On = now()
```

Refusal message: *"Operations {list} are not signed off."*

**Where this gate lives depends on where your work order lives.** If the work order is a record in your ERP —
which it usually is — do not rebuild it here. Evaluate the condition over the `Operation` rows and surface
the answer as a status your ERP or your dispatcher can read, rather than creating a second work order master
that will drift from the first one within a month.

Which means the `ON PASS` above is conditional on a decision you have not made yet: if there is no work order
record in this system, there is nowhere to stamp `Completed_On`, and the right output is a derived status on
a view — the latest `SignedOff_On` across the operations — rather than a stored column. Only create a work
order list, with a `Completed_On` on it, if your work orders genuinely do not live anywhere else.

One line, and it is what stops a work order shipping with an unrecorded operation. It also means an operator
who skips the record entirely gets caught by someone downstream who cannot close the order — which is a much
better discovery point than an audit.

## Walking a run through it

1. Work order `WO-7712`, operation 20, `SHAFT-D7`, 100 planned, `Check_Frequency = 25`. Stage `Ready`.
2. First-off check recorded: diameter 12.02 against a 12.00 ± 0.05 requirement, `Pass`. Gate 1 passes;
   `Released_On` stamped; `Check_Frequency` locks. Stage `Running`.
3. In-process checks recorded at 25, 50 and 75 parts. One reads 12.06 — outside tolerance — and is entered
   as `Fail`.
4. The operator attempts to verify the operation. **Refused**: a failed check with no non-conformance
   against it. `NC-0402` is raised against the operation and the affected parts are contained. The failed
   check is not deleted, which is the point.
5. Last-off recorded. `Quantity_Made = 100`. Gate 2 evaluates: three in-process checks against
   `Required_Checks = max(0, ceiling(100 ÷ 25) − 1) = 3`. Passes. `Verified_On` stamped. Stage `Verified`.
6. The operator attempts to sign off their own operation. **Refused.**
7. The cell leader signs off — but not until `NC-0402` is closed, which takes five weeks, so the operation
   sits at `Verified` in the meantime. That delay is the gate working, and it is what the sample data shows.
   Gate 3 then passes and the operation record locks. Stage `SignedOff`. (On Route A the five check records
   were already locked, individually, at the moment each was created — see below. On Route B they lock here.)
8. Operations 10, 20 and 30 are all signed off, so `WO-7712` can be completed. Gate 4 passes.

An auditor asks for evidence that in-process checks were performed at the required frequency on that work
order. It is one query. Every check timestamp falls between the release of the operation and its
verification, and none of them was typed by a person.

## Route A and Route B

- **Gates 1 and 3 are only partly expressible as Lists validation**, because both count or compare records
  in the `ProductionCheck` list and a validation formula cannot read a second list. The parts that live on the
  `Operation` record itself — `Quantity_Made` present, `Check_Frequency` set, `SignedOff_By` differing from
  `Operator` via the shadow-column technique — do validate. The counting is flow-checked and reverted. See
  `docs/09-microsoft-lists-build.md`.
- **Gate 2 splits into three, and it is worth knowing which third is which.** The coverage comparison becomes
  a validation formula once the flow maintains `InProcess_Check_Count` on the operation — two numbers on one
  row. The anti-retrospective condition becomes one too, once the flow copies `Released_On` down onto each
  check. What does *not* validate is "at least one last-off exists" and "no failed check is without a
  non-conformance", both of which count or inspect rows in the other list; those are flow-checked and
  reverted. So the headline condition of this example is a refusal on Route A, and the two supporting ones
  are detections. Say it that way round.
- **The approver-style rules here need the same anchoring care as `docs/09-microsoft-lists-build.md` §3
  formula 3.** `SignedOff_By <> Operator` compares two shadow columns a flow writes after the save. Key the
  rule on `Stage = SignedOff`, which the user sets, and require both shadow columns to be present — never on
  whether the sign-off email happens to be there, or the gate passes every sign-off made in a single action.
- **The locks are the difference, and here they matter more than in the other two examples**, because the
  thing being protected is a timestamp. On Route B, `Checked_On` can be protected by a server-side rule that
  rejects any update to it once the record exists — read `docs/03-implementation-guide.md` step 4 first,
  because column security profiles do **not** do what people assume here. On Route A there is no equivalent
  while the check record is open: validation cannot see a field's previous value, so a user with grid access
  can set a check's timestamp to any value they like. What you *can* do on Lists is lock each check record
  immediately on creation, through a flow that stops sharing the item **and then grants read access back** —
  checks are short-lived, single-purpose records that nobody should edit after the fact, which makes them the
  one place where locking on creation is the natural design rather than a workaround. Do not skip the second
  half. `Stop sharing an item` removes every permission except site owners, so a check locked and left there
  is invisible to the operator who recorded it, to the person signing the operation off, to Gate 2's own
  refusal message and to the coverage panel on the dashboard. Follow it with `Grant access` at the
  **Visitors** role — read, not Members, which is edit — exactly as the closure lock in
  `docs/09-microsoft-lists-build.md` §3 does. Do that, and the anti-retrospective condition holds. Watch the
  permission-scope count and the daily request allowance, because this is the highest-volume table in the kit
  and it reaches both ceilings first.
- **The blunt version:** if the claim you need to make is "our in-process check records were created while
  the run was happening", Route A does not support it and Route B does. There is no wording that fixes this;
  it is a platform fact.

The test suite in `docs/06-validation-and-test-plan.md` is written against
`examples/incoming-inspection-checklist.md`, so adapt rather than copy: run T11 (direct table write) against
`Operation.Stage`, T13 (timestamp integrity) against `ProductionCheck.Checked_On` rather than `Inspected_On`,
and T09 (separation of duties) against `SignedOff_By` versus `Operator` and versus every `Checked_By`. The
timestamp test is not optional here — it is the one this example depends on.

## What to change for your shop

**Frequency by characteristic, not by operation.** Many processes need a dimensional check every 25 parts
and a visual check every part. Model that as two frequencies rather than forcing everything to the tightest
one — a gate that demands more checking than the process requires gets routed around within a fortnight.

**Time-based rather than count-based frequency.** Continuous or batch processes often check every two hours
rather than every N parts. Same gate, different arithmetic:
`ceiling(run duration ÷ interval) − 1`.

**Attribute versus variable data.** `Value` as text handles both, at the cost of not being able to chart the
variable data without parsing. If SPC matters to you, split it into a numeric column and an attribute
column, and accept that half your rows will have one of them empty.

**Setup approval above a threshold.** High-value or safety-critical operations often need a second person to
approve the first-off, not just record it. That is a fifth stage between `Ready` and `Running`, with its own
separation-of-persons condition.

**Do not add a gate for gauge calibration status.** It is the most commonly requested addition here and it
belongs in your calibration system, which almost certainly already exists. A gate that reads a stale
calibration list is worse than no gate, because it produces confident wrong answers.

## What this example does not do

It is not SPC, it does not do sampling plan selection, it does not know your tolerances, and it will not
tell you whether your check frequency is statistically defensible — that is a process engineering question
and this pattern has nothing to say about it. It carries no gauge management, no traceability to material
lot beyond whatever you put in `Item`, and no machine data capture.

It also asks more of shop-floor people than the other two examples, on a shorter clock, in an environment
where a form that takes ninety seconds too long simply will not get used. If you build one thing from this
kit and you have never done this before, build `examples/nonconformance-intake.md` first and come back to
this one when the organisation has some evidence that the gates were worth it.
