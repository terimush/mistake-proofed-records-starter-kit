# Worked example — non-conformance intake

A non-conformance record that will not close without causal analysis somebody actually did. Fabricated
throughout; no real supplier, part, customer or organisation appears anywhere in this kit.

This is the second worked example. `examples/incoming-inspection-checklist.md` gates an inspection and
raises a non-conformance as a consequence. This one treats the non-conformance as the record in its own
right, because most of them do not come from incoming inspection at all — they come from an operator who
noticed something, a customer who complained, an internal audit finding, or a scrap ticket somebody
questioned.

---

## Why this one is worth building first

If you build only one thing from this kit, build this. Incoming inspection is the easier pilot, but the
non-conformance record is where the retrospective-records failure does the most damage. Nobody reconstructs
an inspection from memory six months later and calls it root cause analysis. People do exactly that with
corrective actions, and the reconstruction is what an auditor is looking at when they ask how you knew the
problem was fixed.

The gate that matters here is not "did you fill in the boxes". It is **"is the analysis deep enough to be
worth anything, and did a second person agree"**.

## The process being modelled

Something is found to be wrong — by anyone, from any source. It is contained so it stops getting worse.
The cause is worked to a depth that reaches something you can actually change. One or more corrective
actions are agreed, owned and dated. Evidence is gathered that the action worked. A second person approves.
Only then does the record close.

```
Raised ──▶ Contained ──▶ Analysed ──▶ Actioned ──▶ Closed
      gate 1        gate 2       gate 3       gate 4
```

Five stages rather than four, because containment and analysis are genuinely different activities happening
on different clocks — containment is today, analysis is this week — and collapsing them is how containment
quietly becomes the whole response.

## What changes in the data model

This uses `Nonconformance` and `CorrectiveAction` from `docs/01-data-model.md`, with a handful of additions
to `Nonconformance` so the record can stand alone rather than hanging off an inspection.

| Column | Type | Notes |
|---|---|---|
| `Source` | choice | `Incoming inspection` · `In-process` · `Final inspection` · `Customer complaint` · `Internal audit` · `Other`. Required at creation. |
| `Inspection` | lookup → `Inspection` | Now **optional**. Populated only when `Source = Incoming inspection`. |
| `Quantity_Affected` | number | How many units are implicated. Gate 1 requires it; enter 0 if genuinely none. |
| `Cause_Level_4`, `Cause_Level_5` | text | Required only when `Severity = Critical`. Gate 2 enforces that. |
| `Stage` | choice | `Raised` · `Contained` · `Analysed` · `Actioned` · `Closed`. Read-only to users; changed only by a gate. |
| `Contained_On`, `Analysed_On`, `Actioned_On`, `Closed_On` | date/time | One per transition, each **system-set** when its gate passes. Never user-typed. |

`CorrectiveAction` gains one column:

| Column | Type | Notes |
|---|---|---|
| `Verified_On` | date/time | System-set when the effectiveness evidence is recorded. Gate 4 requires it. |

Everything else — `Cause_Level_1` through `_3`, `Raised_By`, `Raised_On`, `Approved_By`, `Owner`, `Due` — is
unchanged.

**There is deliberately no separate "detected on" column.** It is the obvious thing to add and it is a trap:
the date somebody noticed a problem is not knowable by the system, so the column can only be user-typed, and
a user-typed date carrying evidentiary weight is the one thing this whole pattern refuses. `Raised_On` is
system-set and means what it says — when the record was created. If the gap between noticing and raising
matters to you, the honest fix is to shorten it, not to record a claim about it.

## Stages and gates

### `Raised` → `Contained` · Gate 1

Containment is the only stage with a clock on it, because an uncontained non-conformance is still producing
scrap while you think about it.

```
REQUIRE  Description   is not blank
     AND Severity      in {Minor, Major, Critical}
     AND Source        is not blank
     AND Containment   is not blank
     AND Quantity_Affected  is not blank        (enter 0 if genuinely none)

ON PASS  set Contained_On = now()               (system clock, not user input)
         lock Description, Severity, Source
```

Refusal message: *"Say what you found, how bad it is, where it came from, and what you did about it right
now. Analysis comes next — this stage is only about stopping the bleeding."*

**Why `Description` locks here.** The single most common way a non-conformance record becomes worthless is
that the description gets quietly rewritten later to match whatever the corrective action ended up being.
Lock it at containment, while nobody yet knows what the cause was.

**Two of these five conditions cannot be list validation on Route A**, and it is worth knowing which before
you build. `Description` and `Containment` are multiple-lines-of-text columns, and a SharePoint validation
formula cannot read that column type at all. `Severity`, `Source` and `Quantity_Affected` are choice and
number columns and validate normally. For the two narrative fields, pick one: make them a single line of text
and accept 255 characters, have the flow write companion length columns that validation *can* test, or accept
a flow check with revert. `docs/09-microsoft-lists-build.md` §1 sets out the trade.

**On `Quantity_Affected is not blank` when the honest answer is zero.** Confirm on your own tenant how
`ISBLANK` behaves against an empty number column before you rely on this one — an empty number and a typed
zero are not always distinguishable in a formula, and this gate depends on the difference. Test it: save a
record with the field empty, then one with `0`, and see which is refused. `docs/06-validation-and-test-plan.md`
T05 does the same test from the other direction.

### `Contained` → `Analysed` · Gate 2

This is the gate the whole example exists for.

```
REQUIRE  Cause_Level_1  is not blank
     AND Cause_Level_2  is not blank
     AND Cause_Level_3  is not blank
     AND Cause_Level_2  <> Cause_Level_1
     AND Cause_Level_3  <> Cause_Level_2
     AND length(Cause_Level_3) >= 20 characters

IF   Severity = Critical
     REQUIRE Cause_Level_4  is not blank
         AND Cause_Level_5  is not blank

ON PASS  set Analysed_On = now()
         lock Cause_Level_1..5
```

Refusal messages, separately:
*"Root cause needs three levels — keep asking why."*
*"Two of your causal levels are the same sentence. Each level should answer why the one above it happened."*
*"A critical non-conformance needs five levels."*

**The two comparison conditions are the interesting part.** Requiring three boxes to be non-empty gets you
three boxes containing "operator error", "operator error" and "operator error". Requiring each level to
differ from the one above costs one line of rule and defeats the most common way of satisfying the gate
without doing the work.

**Be honest about what this cannot do.** A determined person will write "operator error", "the operator made
a mistake" and "a mistake was made by the operator". No condition you can express in a form will catch that.
The gate raises the cost of a fake analysis above the cost of a real one for most people on most days, and
that is the whole claim. If someone is determined to defeat it, the problem is not the software.

**On the length condition.** Twenty characters on the third level only — not the first two. It is a blunt
instrument aimed at the specific failure of a one-word third level, and it should be the first thing you
remove if your people find it insulting. Do not apply it to every field; a genuinely short first-level
observation ("burr on face") is often exactly right.

### `Analysed` → `Actioned` · Gate 3

```
REQUIRE  at least one CorrectiveAction exists
     AND every CorrectiveAction.Action  is not blank
     AND every CorrectiveAction.Owner   is not blank
     AND every CorrectiveAction.Due     is not blank
     AND every CorrectiveAction.Due     >= today
     AND every CorrectiveAction.Owner   is a resolved directory identity

ON PASS  set Actioned_On = now()
```

Refusal message: *"Every corrective action needs someone who owns it and a date it is due. An action with no
owner is a wish."*

**`Due >= today` matters more than it looks.** Without it, the fastest way through this gate is a due date
in the past, which closes the stage and creates an action that is overdue the moment it exists. That
record then sits in the overdue panel of `examples/compliance-status-dashboard.md` forever, and people stop
trusting the panel rather than fixing the record.

**It is also the one condition here that must be evaluated at the transition and nowhere else.** Every other
condition in this example is safe to write as a statement about which states are legal — the shape
`docs/09-microsoft-lists-build.md` §3 recommends, and the reason those rules hold on every write path. This
one is not, because its truth changes on its own. Write `Due >= today` into a state rule and a record at
`Actioned` becomes retrospectively illegal the day its action falls overdue, through nobody's action: the
next write to it is refused, or a revert flow pushes it back to `Analysed`, and it can never reach `Closed`.
Evaluate it in the flow that performs the transition, and leave it out of the validation formula.

### `Actioned` → `Closed` · Gate 4

```
REQUIRE  every CorrectiveAction.Completed_On       is not blank
     AND every CorrectiveAction.Effectiveness_Check is not blank
     AND every CorrectiveAction.Verified_On         is not blank
     AND every CorrectiveAction.Approved_By         is not blank
     AND every CorrectiveAction.Approved_By  <> Nonconformance.Raised_By
     AND every CorrectiveAction.Approved_By  <> CorrectiveAction.Owner
     AND Verified_On  >  Completed_On

ON PASS  set Closed_On = now()
         lock the entire record
```

Refusal messages, separately:
*"An action is not complete until there is evidence it worked. What changed, and how do you know?"*
*"The approver cannot be the person who raised the non-conformance."*
*"The approver cannot be the person who owned the action."*
*"The effectiveness check was recorded before the action was completed. Check the sequence."*

**Two separations, not one.** `examples/incoming-inspection-checklist.md` blocks the raiser from approving.
This example also blocks the *owner* from approving their own action, which is the more common real-world
version — the person who fixed it is usually the person best placed to say it is fixed, and that is exactly
why they should not be the one who says it.

**`Verified_On > Completed_On` is a sequence check, not a date check.** Both are system-stamped, so this
costs nothing and catches the case where somebody recorded the effectiveness evidence before doing the
action. That ordering is impossible if the work happened as described.

## Walking a record through it

1. `NC-0401` raised by an operator. `Source = In-process`, description *"three of twelve housings show
   scoring on the sealing face"*, `Severity = Major`, quantity affected 3. Containment: *"remaining nine
   held at the cell; run stopped"*. Gate 1 passes; `Contained_On` stamped; the description locks. Stage
   `Contained`.
2. Someone attempts to close it here, because the parts were quarantined and the immediate problem is over.
   **Refused.** This is the single most valuable refusal in the example — containment is not correction, and
   a system that lets you stop at containment will collect a year of contained non-conformances and no
   causal analysis at all.
3. Causal levels entered: *"sealing face scored during transfer"* → *"parts transferred in an open tote with
   no separators"* → *"tote specification was never defined for this part after the fixture change"*.
   Each differs from the one above; the third is a system cause, which is where useful corrective action
   lives. Gate 2 passes; the levels lock. Stage `Analysed`.
4. One corrective action created with an owner and a due date two weeks out. Gate 3 passes. Stage
   `Actioned`.
5. The owner marks it complete and attempts to approve it themselves. **Refused** — owner and approver must
   differ.
6. A second person records the effectiveness evidence — *"forty consecutive parts across four runs with no
   scoring"* — and approves. `Verified_On` is stamped after `Completed_On`. Gate 4 passes; the record locks.
   Stage `Closed`.

Twelve months later, someone asks how you know the scoring problem was fixed. The answer is a record whose
causal analysis was locked before anyone knew what the action would be, whose effectiveness evidence
post-dates the action, and whose approval came from someone other than the two people with a reason to want
it closed. Every timestamp on it was set by the system.

## Route A and Route B

Same split as everywhere else in this kit, and it bites harder here because this record carries more weight.

- **The causal-depth conditions are enforceable on either route.** Written as a list validation formula, the
  three-levels-all-different rule holds on Microsoft Lists on every write path. This is the single most
  valuable condition in the kit and the free platform enforces it — see `docs/09-microsoft-lists-build.md` §3.
- **Both approver separations are enforceable on Lists — on the `CorrectiveAction` list, not on this one, and
  only if the rule is anchored correctly.** `Approved_By` and `Owner` both live on `CorrectiveAction`, so
  `Approved_By <> Owner` is a comparison within one row. `Raised_By` lives on `Nonconformance`, so for the
  second comparison the flow that links an action to its non-conformance must also copy `Raised_By_Email`
  down onto the action. Both comparisons are then within one row and validation refuses an invalid approval
  outright, on every write path — **provided the rule keys on an `Approved` yes/no the user sets, and not on
  whether the approver's shadow email is present.** The second shape fails open, silently, for the reason
  `docs/09-microsoft-lists-build.md` §3 formula 3 sets out. Read that before you write this one. Note where the
  refusal lands: it stops the approval being *recorded*, rather than stopping the non-conformance closing
  afterwards. That is the stronger control and the more accurate sentence.
- **The comparisons run on shadow text columns**, because a validation formula cannot compare person columns.
  A determined grid user could edit those columns, which raises the cost of defeating the check without
  making it impossible. Say the true version.
- **Gate 3's "a corrective action exists" cannot be validated on this list**, because it reads a second list.
  Have the flow maintain a count or a yes/no on the non-conformance and validate against that, per
  `docs/09-microsoft-lists-build.md` §3, or accept a flow check with revert.
- **Gate 4 is almost entirely cross-list.** Every condition except `Approved_By <> Raised_By` reads
  `CorrectiveAction` columns, so from this list it is a flow check or a flag. The refusals that matter —
  approver ≠ owner, approver ≠ raiser — happen on `CorrectiveAction` when the approval is entered, which is
  earlier and better.
- **The mid-stage locks do not hold on Route A.** You cannot prevent someone editing `Description` after
  containment or `Cause_Level_1` after analysis. The closure lock at Gate 4 *is* available, through a flow
  that stops sharing the item.
- **Say which one you have.** "Our non-conformance records cannot be closed without three levels of causal
  analysis and independent approval" is true on both routes. "Our causal analysis cannot be altered **by an
  ordinary user** between analysis and closure" is true only on Route B — and it needs those four words even
  there, because no platform in this kit puts a record beyond a sufficiently privileged administrator. The
  second sentence is the one people want to say, so it is the one worth being careful about.

The test suite in `docs/06-validation-and-test-plan.md` is written against
`examples/incoming-inspection-checklist.md`, so adapt rather than copy. T09 and T10 cover the separation of
duties and the display-name trap and transfer directly. Nothing in that suite covers this example's
distinguishing conditions — the second separation (`Approved_By <> Owner`), the causal-difference checks, the
critical five-level branch, or `Verified_On > Completed_On` — so write those four tests yourself before you
describe this system to anybody. They are the reason to build this example rather than the simpler one.

## What to change for your shop

**Severity scale.** Three levels is a minimum. If your customers use a four- or five-point scale, use
theirs — arguing about severity mapping during an audit is a waste of everyone's afternoon.

**Causal depth by severity.** Five levels for critical and three for everything else is a defensible default.
Some shops want five for major as well. Very few should go below three.

**A customer-notification stage.** If you have contractual notification obligations, they belong between
`Contained` and `Analysed`, with a gate requiring the notification reference and the date it was sent —
system-stamped, obviously.

**A disposition or scrap-value approval.** If scrapping above a threshold needs sign-off, that is a gate on
its own, not a note in the containment field.

**Repeat detection.** Not a gate, but the highest-value report you can build on this table: group closed
non-conformances by `Cause_Level_3` and count. A third-level cause that appears four times is telling you
the corrective action did not work, whatever the effectiveness check said.

## What this example does not do

It does not decide severity for you, it does not tell you what an adequate corrective action looks like for
your process, and it does not make the analysis good — it only makes shallow analysis harder to submit than
real analysis. It carries no risk assessment, no cost-of-quality capture, and no link to preventive action
or management review. Those are quality-system questions and this is a records pattern.

And it will not survive a culture where non-conformances are punished. If raising one gets someone shouted
at, they will stop raising them, and you will have built a very well-controlled record of the problems
nobody is afraid to report.
