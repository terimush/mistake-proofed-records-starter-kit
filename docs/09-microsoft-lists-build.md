# Building it on Microsoft Lists

The whole pattern, on the licence you already have. This is the route to start on, and it goes further than
most people building on Lists realise.

---

## Why this document exists

Earlier versions of this kit described Lists as a pilot platform whose gates were advisory, and pointed at
Dataverse for anything you would show an auditor. That was too pessimistic, and in one respect it was simply
wrong.

**Microsoft Lists enforces validation on the server, when the item is saved.** Not in the form. On the item.
That means a validation rule holds whichever client wrote the record — the modern form, the grid, a flow, and
the ordinary write paths of the API. Prove the ones you actually use rather than taking that list on trust:
it is an inference from where the rule lives, not a set of paths Microsoft enumerates, and bulk-migration
interfaces in particular are documented to behave differently from ordinary writes.

The thing this kit says matters most — *enforcement living with the data rather than with the screen* — is
achievable on a standard Microsoft 365 subscription.

What Lists cannot do is lock a single column against a single user. That is a real limit and it shapes the
build below, but it is a narrower limit than "the gates are advisory", and the difference is most of the
value.

**A note on names.** Microsoft Lists is the app; the lists themselves are SharePoint lists, and everything
below lives in list settings. Build these in a list on a SharePoint team site, not as a personal list — the
validation settings, the flows and the permission actions all need the SharePoint-backed version.

---

## 1 · What Lists actually enforces

Four mechanisms, in descending order of strength. Use the strongest one that will express your rule.

### Mechanism 1 — List validation formulas · genuinely enforced

**List settings → Validation settings** takes a formula that must evaluate true before the item will save.
Unlike column validation, a *list* validation formula can reference several columns at once, which is exactly
what a gate is: a condition across the fields of one record.

This is the workhorse. It runs server-side on save, so it applies to every write path rather than to the form
you built.

**What it cannot do**, and these limits determine everything else in this document:

- **It cannot see another item or another list.** Microsoft's formula documentation says of calculated
  fields that they "can only operate on their own row, so you can't reference a value in another row, or
  columns contained in another list or library", and validation formulas are evaluated the same way. So any
  gate requiring *a related record to exist* — an inspection needing a linked non-conformance, a
  non-conformance needing a corrective action — cannot be a validation formula.
  **There is a way round this worth knowing:** if a flow maintains a counter or a flag *on the record itself*
  — a child count, a "has an approved action" yes/no — then the condition becomes a statement about the
  record's own columns and validation can enforce it after all. §3 below uses it, and
  `docs/10-canvas-app-gates.md` §5 step 5 describes how to keep the value honest.
- **A formula is a statement about one row of one list, so decide which list it belongs on before you write
  it.** A condition whose columns live on two different lists is not a validation formula anywhere. Split it:
  put each half on the list that owns its columns, and carry the result across with a flag the flow
  maintains. §3 does exactly that with the separation-of-duties rule.
- **Lookup columns are not supported in a formula.** Documented plainly.
- **Multiple lines of text columns cannot be used at all.** They do not appear in the validation column
  picker, and Microsoft's support answer on this is direct: SharePoint Online "does not support validation
  rules on Multiple lines of text column at this time". That rules out `Description`, `Containment`,
  `Action`, `Effectiveness_Check` and `Notes` in `docs/01-data-model.md`. Three ways round it, in order of
  preference: make the column a single line of text and accept 255 characters; have the flow write a
  companion number column (`Description_Length`) that validation *can* read; or accept a flow check with
  revert. Do not quietly drop the condition.
- **Person columns behave the same way in practice** — and this is the one claim in this document with no
  Microsoft page behind it. Everything else here is documented; this is a behaviour I have found in practice
  and cannot cite. The whole shadow-column technique rests on it, so test it on your own tenant early: try to
  save a validation formula that references a person column, and see what happens. If it works on your
  tenant, most of §2 becomes unnecessary for you and an issue saying so would improve this kit. Until then,
  treat them as unusable and design around it, per mechanism 3 below.
- **It cannot see the previous value of a field.** The formula evaluates the item as submitted, so "this
  field may not change once the stage has passed" is not expressible. That is what mechanism 2 is for.
- **One list gets one formula and one message.** This is a hard constraint and it shapes §3 more than
  anything else on this list. Validation settings hold a single formula for the whole list, so every gate on
  that list is one `AND(...)`, and whatever a user got wrong they get the same sentence back.
  `docs/10-canvas-app-gates.md` §3 is where per-gate messages live.

### Mechanism 2 — Per-item permission lock via flow · genuinely enforced

A flow running under a **list owner** connection can call **Stop sharing an item or a file**, which removes
all permissions on that item except site owners, and **Grant access to an item or folder** to hand back read
access. Run at gate passage, that is a real per-record lock: the record becomes read-only to the people who
created it, on every client, not just on the form.

**Name the role on the re-grant, and check it twice.** The `Roles` property on **Grant access** maps to the
standard SharePoint groups: **Members** is edit, **Visitors** is read. Choosing `Members` re-grants edit on
the item you have just locked, and the flow reports success either way. You want **Visitors**, or a custom
read-only role passed as `role:<role-id>`. This is the one configurable value in the strongest control on
this route; get it wrong and the lock is decorative.

It is coarser than column security — it locks the *whole item*, not the three fields the stage required —
which is why the build below only uses it at closure, where locking everything is what you want anyway.

**It also has a ceiling, and you must plan for it.** Every item whose inheritance is broken is a unique
security scope. Microsoft's documented limit is **50,000 unique permission scopes per list, with a
recommended general limit of 5,000**, and performance degrades once the count passes the list view threshold
because viewing the list starts costing extra database round trips. Treat 5,000 as the number that matters rather than
50,000: it is the figure Microsoft recommends, and the guidance for SharePoint Server adds that where many
scopes sit at the same level the effective number can fall lower still. (That last point is documented for
the on-premises product, so take it as a reason to stay well under the recommendation rather than as a figure
for SharePoint Online.) A shop closing 2,000 records a year reaches 5,000 in under three years.

**Do the arithmetic for the list you are actually locking.** Three years is the inspection list. The check
table in `examples/production-hard-gate-checklist.md` is a different order of magnitude, because that example
locks *every check record on creation*: five checks per operation and three operations per work order is
fifteen scopes per work order, so fifty work orders a week is roughly **39,000 scopes a year** — past the
recommended 5,000 in about seven weeks and past the hard limit inside sixteen months. Locking on creation is
still the right design there; it just makes the archive job a prerequisite rather than a follow-up.

Plan the archive before you need it: move closed records older than N months to an archive list on a
schedule, and the scope count stays flat. Do not discover this at 5,000.

### Mechanism 3 — Flow-based check and revert · detection, not prevention

For the rules validation cannot express — anything crossing records, anything involving a person column — a
flow triggered on change re-evaluates the gate and reverts an invalid transition.

**Be honest about what this is.** The bad write succeeds and is then undone, seconds later. That is
detection with automatic correction, and it is genuinely useful, but it is not the same claim as a refusal
and you should not describe it as one. Log every revert; the log is the evidence that the control operates.

Set a trigger condition on the flow before it ever runs, or it will retrigger itself — see the anti-pattern
warning in `docs/03-implementation-guide.md`.

### Mechanism 4 — Form and view configuration · courtesy only, unless you close the other doors

Hiding `Stage` from the form, greying out an advance button, conditional formatting to show what is missing.
Worth doing, all of it, because a user should learn what a gate needs before they hit it. On its own none of
it is a control — anyone in grid view goes straight past it.

**It would stop being courtesy only if the app were the sole write path**, which means users holding read
access to the list while writes go through a flow running under an identity they do not have. The appendix to
`docs/10-canvas-app-gates.md` sets that arrangement out, what it would cost — chiefly that the platform's own
created-by and modified-by fields stop identifying anybody — and the assumption underneath it that nobody has
confirmed. Until somebody does, treat this mechanism as courtesy.

---

## 2 · The three tables on Lists

Build the model in `docs/01-data-model.md` with three changes forced by the mechanisms above.

**Relationships.** Lookup columns work fine for navigation and are what you want for `Nonconformance` on
`Inspection`. But because a formula cannot read a lookup, **also store the related record's key as plain
text** — a `NC_Reference_Text` column alongside the `Nonconformance` lookup. The flow that creates the link
writes both. Validation can then test the text column for emptiness, which is enough for "a non-conformance
has been linked" even though it cannot verify that the linked record is any good.

**People.** Keep the person columns — they are still the right thing for attribution, and the flow and the
form both resolve them properly. Then have the flow **also write the resolved email into a plain text
column**: `Inspector_Email` on `Inspection`, `Raised_By_Email` on `Nonconformance`, `Owner_Email` and
`Approved_By_Email` on `CorrectiveAction`. Validation can compare text columns, so a separation-of-duties
condition becomes expressible.

**And one of those columns has to travel.** `Approved_By` lives on `CorrectiveAction` and `Raised_By` lives
on `Nonconformance`, so "the approver is not the raiser" spans two lists and is not a formula anywhere as it
stands. Have the flow that links an action to its non-conformance **copy `Raised_By_Email` down onto the
action row** as well. It is then a comparison between two columns of one row, and §3's formula 3 can refuse
it. Copying a value to make a rule local is the same move as the counter in
`docs/10-canvas-app-gates.md` §5 step 5, and it carries the same obligation: the flow owns the copy, and a
scheduled job checks the copies still match their source.

**Flags.** One more machinery column, for the conditions no amount of copying makes local: `Closure_Ready`,
a yes/no on `Inspection` that a flow sets when the linked non-conformance and its corrective action are
complete. Validation reads it; the flow owns it; §3 sets out both.

Hide the text columns from every form. They are machinery, not fields anyone should type into.

**Name the columns exactly as `docs/01-data-model.md` writes them**, underscores and all — `Quantity_Checked`,
not "Quantity Checked". A validation formula refers to a column by its **display name** in square brackets,
case-sensitively, and the settings page refuses to save a formula naming a column it cannot resolve. Creating
the columns with the model's names means the formulas below can be pasted unchanged, and it sidesteps the
`_x0020_` encoding entirely. (Internal names, with the `_x0020_`, are what JSON formatting and the REST API
want. They are not what validation settings want, and putting one here produces a save error rather than a
silent failure.)

**One consequence of the shadow columns you must design around: they are written after the save, so a stage
change and the person column it depends on cannot happen in the same save.** The flow that resolves
`Inspector` into `Inspector_Email` is triggered by the item being written; validation runs while it is being
written. So a user who fills in the inspector *and* sets `Stage` to `Inspected` in one action is refused —
the email column is still blank at the moment the rule reads it.

The sequence that works is two saves:

1. Save the person column with the record still at its current stage. Gate conditions for that stage do not
   read the email column, so the save succeeds and the flow runs.
2. Save the stage change, a few seconds later, once the shadow column is populated.

A canvas app hides this by writing the person column and the shadow column in the same `Patch` and only then
advancing the stage — which is the strongest practical argument for building the app rather than living in
the list forms. If you are not building an app, say the two-step sequence out loud in training, because the
refusal message cannot explain it: the column the rule is complaining about is hidden from the form by the
instruction above.

**Yes, a determined person could edit those text columns in grid view** — unless you have removed write
access to the list entirely — an untested arrangement set out in the appendix to
`docs/10-canvas-app-gates.md`, which you should assume you have not built. On that assumption this is the
honest boundary of the whole route: it raises the cost of defeating a gate from
"click the status field" to "work out which hidden columns the rule reads, and edit them consistently, in a
system that logs the change against your name".
That is a meaningful improvement and it is not the same as impossible. Say the true version.

---

## 3 · The gates, written as Lists validation

For `examples/incoming-inspection-checklist.md`. Three lists, and therefore **three formulas — one per list,
not one per gate.** Which formula goes on which list is the part that decides whether this works, so it is
the first thing below rather than a footnote.

### Which condition lives on which list

A validation formula sees one row of one list. The example's gates do not respect that boundary: Gate 3 talks
about causal levels on `Nonconformance` and an approver on `CorrectiveAction` while deciding whether an
`Inspection` may close. Written as one formula it is unbuildable on any list. Split by ownership and almost
all of it survives.

| Condition | Owns the columns | How it is enforced |
|---|---|---|
| Quantity, result, inspector present at `Inspected` | `Inspection` | List validation · **refused** |
| Non-conformance reference present for a non-Accept disposition | `Inspection` | List validation · **refused** |
| Three causal levels, each different from the one above | `Nonconformance` | List validation · **refused** |
| Approver is neither the raiser nor the owner of the action | `CorrectiveAction` | List validation · **refused** |
| The linked non-conformance and its action are actually complete | spans three lists | Flow sets a yes/no flag on the `Inspection` row; validation reads the flag · **refused on the flag, checked by the flow**|

The last row is the honest compromise and it is worth understanding rather than skimming. Validation cannot
read another list, but it can read a column on its own row that a flow wrote — the technique in
`docs/10-canvas-app-gates.md` §5 step 5. So the flow evaluates the cross-list part and writes
`Closure_Ready` (yes/no) onto the inspection; validation then refuses closure whenever that flag is not set.
The *refusal* is genuine and holds on every write path. What it refuses on is the flow's answer, so the flag
is only as trustworthy as the flow that maintains it — which is why the flag must be written by the flow and
by nothing else, and why the reconciliation job in `docs/10-canvas-app-gates.md` §5 step 5 is not optional.

Two conditions from the example do **not** survive the split, and you should say so rather than imply
otherwise:

- **"The non-conformance has a description, a severity and a containment action."** `Description` and
  `Containment` are multiple lines of text, which validation cannot read at all. Either make them single
  lines of text, or have the flow write companion length columns, or accept a flow check with revert.
- **"An inspection cannot close unless its approver differs from its raiser."** The refusal exists, but it
  happens one list down: `CorrectiveAction` refuses the approval. By the time closure is attempted there is
  no invalid approval left to catch. That is a better control than the one the example describes — it stops
  the act rather than the consequence — but it is a different sentence and you should say the true one.

### Formula 1 — the `Inspection` list

Validation settings hold **one** formula for the list, so all three of this list's gates are one `AND(...)`.
Read the shape before you copy it.

```
=AND(
   IF( [Stage]="Draft", TRUE,
       AND( NOT(ISBLANK([Quantity_Checked])),
            [Quantity_Checked] <= [Quantity_Received],
            NOT(ISBLANK([Result])),
            NOT(ISBLANK([Inspector_Email])) ) ),
   IF( OR([Stage]="Draft", [Stage]="Inspected"), TRUE,
       IF( [Result]="Accept", TRUE,
           NOT(ISBLANK([NC_Reference_Text])) ) ),
   IF( [Stage]<>"Closed", TRUE,
       OR( ISBLANK([NC_Reference_Text]), [Closure_Ready]=TRUE ) )
)
```

**Read the shape, because every formula below reuses it.** Each clause is not "when advancing, require X". It
is **"if the record is at or past this stage, X must be true"** — a statement about which states are legal
rather than about which transitions are. Written that way, one formula defends the record against every route
in: the form, the grid, the import, the API. A user who sets `Stage` to `Inspected` in grid view without
filling the fields gets the same refusal as one using the form, because the item as submitted is not a legal
item.

Since each condition is expressed over the whole region at or past its stage, it also blocks the reverse
attack — clearing `Result` on a record that has already been inspected fails the same test.

**One caution about writing rules this way.** A state rule must stay true for as long as the record sits in
that state. Never put a condition into one of these clauses whose truth changes on its own — a comparison
against today's date is the usual offender. `examples/nonconformance-intake.md` Gate 3 requires a due date in
the future; that is a *transition* rule and it must be evaluated by the flow at the moment of the transition,
not written into the state formula, or every record whose action falls overdue becomes retrospectively
illegal and gets reverted out of the stage it legitimately reached.

**Validation message** (one for the list, so it has to cover all three):

> *"Something a stage this record has already passed still needs filling in. To mark it inspected: the
> quantity checked (not more than received), the result, and the inspector. To disposition a non-Accept
> result: a linked non-conformance. To close it: the linked non-conformance and its corrective action have to
> be complete — the panel in the app says which part is outstanding."*

That message is worse than three targeted ones and there is nothing to be done about it on the list layer.
Per-gate refusals are what the canvas app is for; `docs/10-canvas-app-gates.md` §3 says so and it is the
strongest reason to build one.

### Formula 2 — the `Nonconformance` list

Causal depth, genuinely enforced, on the list that owns the columns. It needs no stage column and no copying,
because it is written as a rule about *coherence* rather than about completeness: you may have entered no
causal analysis yet, or you may have entered three distinct levels, and there is nothing in between.

```
=OR(
   AND( ISBLANK([Cause_Level_1]), ISBLANK([Cause_Level_2]), ISBLANK([Cause_Level_3]) ),
   AND( NOT(ISBLANK([Cause_Level_1])),
        NOT(ISBLANK([Cause_Level_2])),
        NOT(ISBLANK([Cause_Level_3])),
        [Cause_Level_2] <> [Cause_Level_1],
        [Cause_Level_3] <> [Cause_Level_2] ) )
```

**Validation message:** *"Root cause goes in three levels or none — keep asking why, and make each level
answer why the one above it happened."*

This is the single most valuable condition in the kit and the free platform refuses it outright, on every
write path. Note what the shape buys you: a partially-filled analysis cannot be saved at all, so nobody
accumulates records with one level in them waiting for someone to come back. If you want the causal levels
required *only* at a particular stage, add the `Stage` column from
`examples/nonconformance-intake.md` and wrap this in the same `IF([Stage]=...)` shape as formula 1.

### Formula 3 — the `CorrectiveAction` list

Separation of duties, on the list where the approval actually happens. It needs two pieces of machinery, and
the second one is the difference between a control and a decoration.

**First, a copy.** The flow that links a corrective action to its non-conformance must also copy the
non-conformance's `Raised_By_Email` onto the action row. That is a copy on the same row, not a cross-list
read, and validation can see it.

**Second, a state marker the user sets — `Approved`, a yes/no column.** It exists because of a trap that will
otherwise turn this gate into nothing. Read the next two paragraphs before you write the formula.

> **The shape that fails open, and why you must not use it.** The obvious formula is *"if there is an
> approver's email, it must differ from the raiser and the owner"* — `IF(ISBLANK([Approved_By_Email]), TRUE,
> AND(...))`. It does not work, and it fails in the direction that lets things through. The user sets the
> `Approved_By` **person** column and saves. At that instant the shadow column is still blank, because the
> flow that populates it runs after the save — so `ISBLANK` is true, the formula returns TRUE, and the
> self-approval is saved. Then the flow tries to write the raiser's address into `Approved_By_Email`, *that*
> write is the one validation refuses, and the shadow column stays blank for ever. The record now holds a
> self-approval that satisfies the rule permanently, and any query looking for
> `Approved_By_Email = Raised_By_Email` returns nothing, because the column it reads was never written.
>
> **The general rule this is an instance of: never write a state rule whose condition is satisfied by the
> shadow column being absent.** The shadow column is absent for a few seconds after every single save, and
> absent for ever on any record whose corrective write the rule itself refused. Every clause must be
> anchored on something the *user* wrote, and must require the shadow column rather than excusing it.

```
=IF( [Approved] <> "Yes", TRUE,
     AND( NOT(ISBLANK([Approved_By_Email])),
          [Approved_By_Email] <> [Raised_By_Email],
          [Approved_By_Email] <> [Owner_Email] ) )
```

**Validation message:** *"An approved action needs an approver, and the approver cannot be the person who
raised the non-conformance or the person who owns the action. Record the approver, wait for their name to
resolve, then mark it approved."*

Now it fails closed. Marking an action approved before the shadow column has been written is refused;
marking it approved when the approver is the raiser or the owner is refused; and clearing the shadow column
on an already-approved action is refused too, because the condition covers the whole `Approved = Yes` region
rather than the moment of approval. This is the condition `docs/02-hard-gate-pattern.md` §3 calls the one
that does most for audit credibility, and on Lists it is a refusal rather than a detection.

**It costs the two-save sequence from §2, and here that is a feature.** Record the approver; wait a moment
for the name to resolve into the hidden column; then set `Approved` to Yes. If you set both at once you are
refused with the message above. A canvas app collapses the two into one action.

**One residual gap, and say it rather than let an auditor find it.** Between the flow writing
`Approved_By_Email` and the user setting `Approved`, a determined person could change `Approved_By` back to
themselves and mark it approved in the same save — the stale shadow value would pass. The reconciliation job
below is what catches that: it re-reads every approved action, compares the shadow columns against the person
columns they came from, and flags every row where they disagree.

**What it rests on.** The comparison is between text columns a flow populated from person columns. A user
with edit access to the `CorrectiveAction` list could edit those hidden columns in grid view and defeat it —
unless you have removed write access to the list, which is the untested arrangement in the appendix to
`docs/10-canvas-app-gates.md`. Assuming you have not, this raises the cost of defeating the check from "type
your own name" to "work out which hidden columns the rule reads and edit them consistently, in a system that
logs the change against your name". That is a
meaningful improvement and it is not the same as impossible. Say the true version.

**On the `ME()` function.** SharePoint's formula reference includes an `ME` function for the current user. If
it evaluates inside validation settings on your tenant, adding `[Approved_By_Email] <> ME()` would let you
block someone approving under another person's name, which the text-column comparison cannot. I have not
confirmed that it works there, and nothing in this build depends on it. It is five minutes to test and worth
knowing either way — if it works, say so in an issue and this document gets better.

### The flow that maintains `Closure_Ready`

One yes/no column on `Inspection`, written by one flow and by nothing else. Triggered on changes to
`Nonconformance` and `CorrectiveAction`, it sets the flag true only when all of the following hold for the
non-conformance linked to that inspection:

- three causal levels present (already refused on the `Nonconformance` list, so this is a re-check rather
  than the control);
- at least one corrective action exists, with `Owner` and `Due` populated;
- that action carries an `Approved_By` (which the `CorrectiveAction` list has already refused if it matched
  the raiser or the owner).

Anything else, it sets the flag false.

**On a record that has already closed, it cannot do that in one step, and the flow has to be written for it.**
Clause 3 of formula 1 says a `Closed` record must carry `Closure_Ready = Yes`, so a bare write of `No` to a
closed inspection is refused by the very rule the flag exists to serve — the flow errors, the flag stays Yes,
and the record sits closed on evidence that has since been undone. Write both columns in one update instead:
set `Stage` back to `Dispositioned` **and** `Closure_Ready` to `No` in the same action, which is a legal item
and therefore saves. That is the flow-with-revert pattern of mechanism 3, and it is the honest behaviour —
the record leaves the closed state because it no longer qualifies for it.

Two things follow. The flow needs the list-owner connection for this, because closure has already made the
item read-only to everyone else. And a reverted closure is an event somebody should see, not a silent
correction: raise it, do not just log it.

**Log every transition of the flag, and every write the flow attempted and could not make.** A refused write
by the flow is not a nuisance — it is the platform telling you a record is now in a state your rules forbid,
and it is exactly the case that would otherwise pass unnoticed. The log is the evidence that the control
operates, and it is what you show when somebody asks how you know the cross-list check ran.

### Locking the closed record

Validation cannot express "these fields may not change now", because it cannot see the previous value. So at
closure, the flow calls **Stop sharing an item** and then **Grant access** with the **Visitors** role — read,
not Members, which is edit — to the site's members group. The record becomes read-only to everyone but site
owners.

This is the strongest single control available on this route and it is why closure is the right place to
spend the permission-scope budget. Watch the count, and archive on a schedule.

## 4 · Build order on Lists

Ten steps. A working, genuinely-gated incoming inspection in an evening if you have used a low-code forms
tool, a weekend if not.

| # | Step | Why it is in this position |
|---|---|---|
| 1 | Create the three lists on a SharePoint team site | Not personal lists — the settings below do not exist there |
| 2 | Create every column with the **display name** used in `docs/01-data-model.md`, underscores included | Validation formulas take display names; matching the model means the formulas paste unchanged |
| 3 | Add the shadow columns (`*_Email`, `NC_Reference_Text`, `Closure_Ready`) and hide them from forms | Every formula in §3 depends on them |
| 4 | Build flows **F1** and **F2** below, **before** you write any formula | The formulas require columns only these flows fill; write the formulas first and nothing can be saved |
| 5 | Paste **one** formula per list — the whole of formula 1 on `Inspection`, formula 2 on `Nonconformance`, formula 3 on `CorrectiveAction` | A list holds one formula. Adding gates one at a time overwrites the one before it |
| 6 | Test each list's formula from the **grid**, not the form, and test every clause separately | The grid is the path that proves the rule is on the item; one formula means one message, so only clause-by-clause testing tells you all three gates are live |
| 7 | Build flows **F3**, **F4** and **F5**, each with a trigger condition | The infinite-loop anti-pattern is real; constrain the trigger before you point it at anything |
| 8 | Add the closure lock (stop sharing, then grant **Visitors**) under a list-owner connection | Needs owner rights; a member connection fails at run time, not at design time. `Members` would re-grant edit |
| 9 | Set up the archive job before you need it | 5,000 unique scopes arrives sooner than you think |
| 10 | Re-run step 6 with a few thousand rows in the lists | Nothing above changes with volume, but anything you put in a canvas app does — see `docs/06-validation-and-test-plan.md` T16 |

### The flows this build needs

Every formula above depends on a column some flow writes, and until now those flows have been named in
passing rather than listed. Build all five before you trust any gate. None is more than a handful of actions,
and every one needs a trigger condition excluding the service identity, or it retriggers itself.

| # | Flow | Trigger | What it writes | Why a gate needs it |
|---|---|---|---|---|
| F1 | **Resolve people** | Item created or modified on all three lists | `Inspector_Email`, `Raised_By_Email`, `Owner_Email`, `Approved_By_Email` from the matching person column | Formula 1 clause 1 and formula 3 read these; nothing else writes them |
| F2 | **Link and copy down** | Item created or modified on `Nonconformance` and `CorrectiveAction` | `NC_Reference_Text` on the inspection from the linked non-conformance; `Raised_By_Email` from the non-conformance onto each of its corrective actions | Formula 1 clause 2 and formula 3's raiser comparison |
| F3 | **Maintain `Closure_Ready`** | Item created or modified on `Nonconformance` and `CorrectiveAction` | `Closure_Ready` on the inspection — and, on a record that has already closed, `Stage` back to `Dispositioned` in the same update | Formula 1 clause 3 |
| F4 | **Stamp transitions** | Item modified on `Inspection`, where `Stage` changed | `Inspected_On` and the other transition timestamps | Nothing refuses a blank timestamp — see the warning below |
| F5 | **Reconcile** | Scheduled, nightly | Nothing. Reports mismatches | Every column above is a copy, and a copy can drift |

**F4 and the timestamps: know what is enforced and what is merely done.** `docs/02-hard-gate-pattern.md` §4
says every evidentiary timestamp must be system-generated, and F4 does that. But **no validation formula can
require it**, and you should not try: the flow stamps `Inspected_On` after the transition saves, so a rule
demanding it at `Inspected` would refuse the very save that triggers the stamp. So the timestamp is
system-set and trustworthy, and its *presence* is not gated. A record that reached `Inspected` while F4 was
switched off carries a blank `Inspected_On` and nothing refuses it. F5 is what finds those; make "non-`Draft`
record with no `Inspected_On`" one of the things it reports.

**F5 is not optional and it is not a tidiness job.** It is the only thing standing behind four separate
claims: that the shadow email columns still match the person columns they came from, that
`NC_Reference_Text` still matches the lookup, that `Closure_Ready` still reflects the linked records, and
that no record slipped past a stamp. Have it write its findings somewhere durable and read them.

**Step 5 is the one that surprises people.** Validation settings hold a single formula for the whole list. If
you paste Gate 1, test it, then paste Gate 2, you have replaced Gate 1 — and because Gate 2 also refuses a
half-empty record, the test in step 6 still passes and you will believe you have both. Paste the whole of
formula 1 in one go.

**Step 6 is the one people skip and it is the whole point.** Open the list in grid view, type `Closed` into
`Stage` on a half-empty record, and press enter. If it saves, your rule is in the form and you have built the
thing this kit exists to warn about. If it refuses, the rule is on the item and you have a hard gate on a
licence you already own. Then do it again for each clause: a record with a result but no inspector, a
non-Accept disposition with no non-conformance reference, a closure with `Closure_Ready` unset. One passing
test proves one clause, not the formula.

---

## 5 · What this route still cannot do

The honest ceiling, in one place, so you can decide whether it is high enough.

| Limit | Consequence | Live with it, or move? |
|---|---|---|
| No column-level permissions | Cannot lock three fields while leaving others editable. All-or-nothing at item level. | Live with it. Lock at closure, accept mid-stage edits are possible. |
| Validation cannot read another list | Cross-record rules become a flow-maintained flag the formula reads, or a flow check with revert | Live with it. The refusal on the flag is real; the flag's honesty is the flow's job, so log it. |
| One formula and one message per list | Every gate on a list shares one refusal sentence | Live with it, and put per-gate messages in the app — `docs/10-canvas-app-gates.md` §3. |
| Multiple lines of text unusable in formulas | Any "this narrative field is not blank" gate needs a companion column or a flow check | Live with it. Prefer a single line of text where 255 characters will do. |
| Validation cannot see the previous value | No "field is frozen once its stage passed" before closure | Live with it, unless mid-stage tampering is your actual risk. |
| Person columns unusable in formulas | Separation of duties rests on shadow text columns a grid user could edit | Live with it. Log changes to those columns and review them. |
| Shadow columns are written after the save | A stage change and the person column it depends on need two saves | Live with it, or build the app and do both in one `Patch` — §2. |
| 5,000 unique permission scopes recommended | Closure locking has a shelf life without archiving | Live with it, with an archive job — and do the arithmetic for the check table, not the inspection list. |
| 6,000 Power Platform requests per user per 24 hours on the seeded licence | This design is flow-per-record. Every action inside every flow counts, and a background flow always draws on its **owner's** allowance, not the acting user's | Live with it, but count first — see below. |
| No transition-order enforcement | A record with every field filled could jump `Draft` → `Closed` | Add a flow check if order matters; usually it does not. |

**A word about the last row, because it is the one that bites a working shop rather than a pilot.** Microsoft
documents 6,000 Power Platform requests per user per 24 hours for Microsoft 365 licences — currently 10,000
during a transition period, which is not a number to build against. Every action inside a flow counts, not
every run, and retries and pagination count too. The part that catches people: automated and scheduled flows
"always use the limits of the **owner** of the process, regardless of why the process started or which
accounts are used for connections within the process". So all five flows above draw on one person's daily
allowance, however many inspectors are working. There is also a five-minute ceiling of 100,000 requests
independent of licence, and a flow throttled continuously for **14 days** is turned off altogether.

Count before you build at volume. A single check record that triggers F1, F4 and a closure lock is not one
request, it is a dozen or more once you include the gets, the conditions and the updates. The high-volume
example in this kit — `examples/production-hard-gate-checklist.md`, where every check record is locked on
creation — is the one to work out first, because the failure is silent in the worst way: locks stop being
applied and flags stop being maintained while records carry on closing, and nothing in the record says so.
Two mitigations, and use both: own the flows with a dedicated service account rather than a person, and put
"flows ran, and ran to completion" on the dashboard beside the records they guard.

**The one that would move me to Dataverse** is not on that list as a limit — it is a symptom. If you find
yourself needing to prove that a record *was not altered between stages*, rather than that it was complete at
each stage, you have outgrown this route. That is a per-column, per-record, stage-conditional lock, and it is
the thing Lists genuinely cannot do at any level of cleverness.

Everything else here is a design constraint you can work within.

## 6 · The sentence you can honestly give an auditor

Derive it from what you actually built, not from this document.

**If you completed steps 1–9 and the grid tests in step 6 refused:**

> "Records are validated on the lists themselves rather than in the form, so the conditions hold whichever
> client writes the record — we tested that directly, from the grid. An inspection cannot reach a stage
> without the evidence that stage requires. A causal analysis cannot be saved at less than three distinct
> levels. An action cannot be marked approved unless the approver is recorded and is neither the person who
> raised the non-conformance nor the person who owns the action. Conditions that span records are evaluated by an
> automated process which maintains a flag on the record, and the record cannot close while that flag is
> unset; the flag's history is logged. Closed records are made read-only at closure. There is no
> column-level permission on this platform, so between stages a user with grid access can alter fields their
> earlier stage required; we have not claimed otherwise, and closure is the point at which the record becomes
> fixed."

That is a longer sentence than "the system enforces it", and every clause in it is true. The long true one
survives the follow-up question. Run `docs/06-validation-and-test-plan.md` and keep the results, so the
first clause is a fact you can show rather than a claim you make.

## What this document does not do

It does not make Lists into Dataverse, it does not survive a requirement for validated software, and it does
not remove the need to read `docs/03-implementation-guide.md` on where gates belong. The formulas above are
written for the incoming-inspection example and will need adapting to your own column names and stages —
mechanically, but attentively. Validation settings refuse to save a formula naming a column they cannot
resolve, so a mistyped column name shows up immediately rather than silently — but a formula that is *valid*
and tests the wrong column saves happily and tells you nothing, so read each clause back against
`docs/01-data-model.md` before you trust it.

Nothing here has been tested on your tenant. Tenants differ, entitlements change, and this document is a year
old the moment it is published. Build it in a sandbox and prove each formula yourself.
