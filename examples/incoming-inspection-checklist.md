# Worked example — incoming material inspection

A complete, concrete instance of the pattern. Fabricated throughout; no real supplier, part or organisation
appears anywhere in this kit.

---

## The process being modelled

Material arrives from a supplier. Someone checks it against the purchase requirement. If it passes, it is
released to stores. If it fails, it is quarantined, a non-conformance is raised, the cause is worked, and a
corrective action is agreed with the supplier or internally before the record closes.

## Stages and gates

### `Draft` → `Inspected` · Gate 1

```
REQUIRE  Quantity_Checked  is not blank
     AND Quantity_Checked  <= Quantity_Received
     AND Result            in {Accept, Reject, Use-as-is, Rework}
     AND Inspector         is not blank

ON PASS  set Inspected_On = now()            (system clock, not user input)
         lock Quantity_Checked, Result, Inspector
```

Refusal message: *"Cannot mark inspected — record the quantity checked, the result, and who inspected it."*

### `Inspected` → `Dispositioned` · Gate 2

```
IF   Result = Accept
     REQUIRE nothing further

IF   Result in {Reject, Use-as-is, Rework}
     REQUIRE Nonconformance is linked
         AND Nonconformance.Description   is not blank
         AND Nonconformance.Containment   is not blank
         AND Nonconformance.Severity      is not blank
```

Refusal message: *"A non-accept result needs a linked non-conformance with a description, severity and the
containment action you took."*

### `Dispositioned` → `Closed` · Gate 3

```
IF   no Nonconformance linked
     REQUIRE nothing further

IF   Nonconformance linked
     REQUIRE Cause_Level_1, Cause_Level_2, Cause_Level_3  all not blank
         AND at least one CorrectiveAction exists
         AND CorrectiveAction.Owner        is not blank
         AND CorrectiveAction.Due          is not blank
         AND CorrectiveAction.Approved_By  is not blank
         AND CorrectiveAction.Approved_By  <> Nonconformance.Raised_By

ON PASS  lock the entire record
```

Refusal messages, separately:
*"Root cause needs three levels — keep asking why."*
*"A corrective action needs an owner and a due date."*
*"The approver cannot be the person who raised the non-conformance."*

**Three messages is what the process deserves and more than one layer can give you.** A SharePoint list holds
a single validation formula and a single message, so on Route A these three collapse into one sentence unless
you build the app — and each of them is refused by a different list anyway. Write them as three; deliver them
as three from the app; expect one from the list. `docs/10-canvas-app-gates.md` §3 is the argument for
building the app, and this is it.

## Walking a failing record through it

1. Receipt `RC-2041` logged, 500 units of `BRACKET-A1` from `Supplier Nine`. Stage `Draft`.
2. Inspector checks 50, finds 4 out of tolerance, sets `Result = Reject`. Gate 1 passes; `Inspected_On`
   stamped by the system; those fields lock. Stage `Inspected`.
3. Attempt to close immediately → **refused**, no non-conformance linked. Gate 2 does its job.
4. `NC-0312` raised: description, `Severity = Major`, containment "remaining 450 quarantined pending
   supplier response". Gate 2 passes. Stage `Dispositioned`.
5. Attempt to close with one causal level → **refused**. Three levels entered: *"dimension out of
   tolerance"* → *"tool wear not detected between batches"* → *"no in-process check defined at that
   operation"*. Note the third is a system cause, which is where useful corrective action lives.
6. `CA-0155` created: add in-process gauge check at that operation, owner assigned, due date set.
7. The raiser attempts to approve their own action → **refused**. A second person approves. Gate 3 passes;
   the record locks. Stage `Closed`.

Six weeks later an auditor asks for evidence of supplier control. It is one query, and every timestamp on it
was set by the system rather than typed by anyone.

## What to change for your shop

`Result` values, severity scale, and causal depth are the obvious ones. Some shops need a customer-
notification stage before closure; some need disposition approval above a scrap value threshold; most will
want `Reference` to match whatever their ERP already calls a receipt, so the two can be reconciled without a
lookup table.

Change the stages freely. Keep the three properties that make it work: gates enforced on the data rather
than on the screen, earlier stages locked once passed, and every evidentiary timestamp set by the system.

## Route A and Route B

The `REQUIRE` conditions and the `ON PASS ... lock` instructions have different fates on Route A. Read
`docs/09-microsoft-lists-build.md` §3 before building this one — it gives these three gates as actual
Microsoft Lists validation formulas.

- **Every condition above is enforced on Route A, but not all of them on the list you would expect.** A
  validation formula sees one row of one list, and these gates do not respect that boundary: Gate 3 talks
  about causal levels on `Nonconformance` and an approver on `CorrectiveAction` while deciding whether an
  `Inspection` may close. Written as one formula it is unbuildable on any list. Split by ownership and it
  works: Gate 1 and Gate 2 on `Inspection`, the causal-depth rule on `Nonconformance`, the approver
  comparison on `CorrectiveAction`. `docs/09-microsoft-lists-build.md` §3 gives all three formulas and says
  which list each belongs on.
- **The separation of duties is a refusal, one list down.** `CorrectiveAction` refuses an approval by the
  raiser or by the owner, on every write path. So by the time closure is attempted there is no invalid
  approval left to catch. That is a stronger control than the gate as written here — it stops the act rather
  than the consequence — but it is a different sentence, and the true one is "an approval cannot be recorded
  by the raiser", not "an inspection cannot close if it was self-approved".
- **Gate 2's sub-conditions are the partial ones.** Validation cannot read the `Nonconformance` list, and
  `Description` and `Containment` are multi-line text columns that validation cannot read even on their own
  list. So what `Inspection` enforces is that a non-conformance *reference* has been recorded. That the
  referenced record has a description, a severity and a containment action is checked by a flow.
- **Gate 3's cross-list part becomes a flag.** The flow evaluates "the non-conformance and its action are
  complete" and writes a `Closure_Ready` yes/no onto the inspection; validation refuses closure while that
  flag is unset. The refusal is genuine and holds on every write path — but it refuses on the flow's answer,
  so log every change of the flag and reconcile it.
- **The locks are the real gap.** On Route A there are no column-level permissions, so `lock
  Quantity_Checked, Result, Inspector` is not achievable at Gate 1. `lock the entire record` at Gate 3 *is*,
  through a flow that stops sharing the item — a genuine per-record lock, subject to the permission-scope
  ceiling.

**So say the accurate thing, and mind the qualifiers — they are the whole difference.** "An inspection
cannot be closed without a result, an inspector, and a non-conformance reference where the result was not
Accept" is true on both routes. "A closed inspection cannot be altered **by an ordinary user**" is true on
both, once the closure lock is in place — say those four words, because a site owner or tenant administrator
can still write to it and an auditor who finds that out afterwards will discount everything else you said.
"Nothing in the record changed between inspection and closure" is true only on Route B. Run
`docs/06-validation-and-test-plan.md` and describe what your build actually blocks.

## What this example does not do

It does not tell you what to inspect, how many to check, or against what sampling plan — those are process
decisions and this pattern has nothing to say about them. It carries no supplier rating, no certificate of
conformity handling, no goods-receipt posting to your ERP, and no cost recovery. It is the record of a check
having happened, with the evidence attached and the chronology trustworthy, and nothing more than that.
