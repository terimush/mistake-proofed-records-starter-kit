# Building it on Microsoft Lists

The whole pattern, on the licence you already have. This is the route to start on, and it goes further than
most people building on Lists realise.

---

> ### ⚠ Before you create a single list
>
> **Personal lists — the ones under "My lists" in the Microsoft Lists app — have no Validation settings.**
> The setting this whole document is built on exists only on a list that lives on a SharePoint site. If you
> start in "My lists", you will build three lists and their columns, open List settings to paste formula 1,
> and find there is nothing to paste it into. The lists then have to be rebuilt on a site; there is no
> "move to site" that carries settings across, and the flows in §4 bind to the site list, not the personal
> one.
>
> So, before anything else: create (or get made an owner of) a **SharePoint team site**, and create the three
> lists **from that site**, not from the Lists app's home page. The first build of this kit from its own
> instructions (`WI-M365-01`, Part A) made this a pre-flight check for exactly this reason. If you have
> already entered data into a personal list, `docs/08-troubleshooting-and-faq.md` — *"I can't find Validation
> settings"* — says what to do with it.

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
       IF( [Result]="Accept", TRUE,
           [Closure_Ready]=TRUE ) )
)
```

**Clause 3 is anchored on `Result`, not on `NC_Reference_Text`, and that is deliberate.** The obvious way to
write it is *"either there is no non-conformance, or the closure flag is set"* — and that is the shape
`docs/01-data-model.md` forbids, because `NC_Reference_Text` is a machinery column and the clause would then
be satisfied by the flow's column being **absent**. Written as above, the escape route is `Result`, which the
user sets and which clause 2 already governs; the machinery column is required rather than excused.

**Read the shape, because every formula below reuses it.** Each clause is not "when advancing, require X". It
is **"if the record is at or past this stage, X must be true"** — a statement about which states are legal
rather than about which transitions are. Written that way, one formula defends the record against every route
in: the form, the grid, the import, the API. A user who sets `Stage` to `Inspected` in grid view without
filling the fields gets the same refusal as one using the form, because the item as submitted is not a legal
item.

Since each condition is expressed over the whole region at or past its stage, it blocks one form of the
reverse attack: **clearing** `Result` on a record that has already been inspected fails the same test, because
the cleared item is not a legal item at that stage.

⚠ **It does not block *changing* it, and you should not claim that it does.** A user with edit access can set
`Result` from `Reject` to `Accept` on a dispositioned record and the item stays legal at every clause — so the
record closes as an accepted receipt, and the closure evidence a rejection would have required is never asked
for. The same applies to lowering `Quantity_Checked` or replacing `Inspector_Email` after the fact. Validation
cannot see a field's previous value, so no formula on this list can catch it; the reason is the same one that
means Lists has no column-level permissions. `docs/06-validation-and-test-plan.md` T12 expects exactly this
and marks it allowed on Route A. The ceiling table in §1 now names it, `docs/03-implementation-guide.md`
step 7 has always listed it as cheat 2 and said plainly that it succeeds on Route A — this page is what was
out of step, not that one. It is one of the symptoms that means you have outgrown Lists.

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
causal analysis yet, or you may have entered three levels that are all different from one another, and there
is nothing in between.

```
=OR(
   AND( ISBLANK([Cause_Level_1]), ISBLANK([Cause_Level_2]), ISBLANK([Cause_Level_3]) ),
   AND( NOT(ISBLANK([Cause_Level_1])),
        NOT(ISBLANK([Cause_Level_2])),
        NOT(ISBLANK([Cause_Level_3])),
        [Cause_Level_2] <> [Cause_Level_1],
        [Cause_Level_3] <> [Cause_Level_2],
        [Cause_Level_3] <> [Cause_Level_1] ) )
```

**All three pairs are compared, and the third comparison is the one people leave out.** Checking only
each level against the one above it lets a circular chain through — *"Operator error / Training gap /
Operator error"* satisfies two adjacent comparisons and says nothing. Three comparisons for three levels; if
you extend to five levels in `examples/nonconformance-intake.md`, that is ten pairs, and at that point the
check belongs in the flow rather than in the formula.

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
> shadow column being *absent* — and never trust one that could be satisfied by a shadow column being
> *stale*.** The shadow column is absent for a few seconds after every single save, and
> absent for ever on any record whose corrective write the rule itself refused. Every clause must be
> anchored on something the *user* wrote, and must require the shadow column rather than excusing it.

```
=IF( [Approved] <> TRUE, TRUE,
     AND( NOT(ISBLANK([Approved_By_Email])),
          NOT(ISBLANK([Raised_By_Email])),
          NOT(ISBLANK([Owner_Email])),
          [Approved_By_Email] <> [Raised_By_Email],
          [Approved_By_Email] <> [Owner_Email] ) )
```

**Validation message:** *"An approved action needs an approver, and the approver cannot be the person who
raised the non-conformance or the person who owns the action. Record the approver, wait for their name to
resolve, then mark it approved."*

**Both operands of each comparison are required to be present, not just the approver's.** A comparison
against a blank column is unequal, so `[Approved_By_Email] <> [Raised_By_Email]` is satisfied whenever
`Raised_By_Email` happens to be empty — and it is permanently empty on any action created without a parent
non-conformance, because the flow that copies it down has nothing to copy from. That is the same trap one
column to the left. Requiring all three shadow columns closes it, and it is why the `Nonconformance` lookup on
`CorrectiveAction` must be a required column.

**A note on the yes/no comparison.** Both this formula and formula 1 compare a yes/no column to the bare
`TRUE`, not to the string `"Yes"`. Keep them consistent, and confirm the behaviour on your own tenant with a
single deliberate test before you trust either — a yes/no comparison that never evaluates equal turns this
whole formula into a no-op that saves without error and looks identical from the form. `T09` is the test that
catches it and it is the one people skip, because it needs two identities.

Now it fails closed. Marking an action approved before the shadow column has been written is refused;
marking it approved when the approver is the raiser or the owner is refused; and clearing the shadow column
on an already-approved action is refused too, because the condition covers the whole `Approved = Yes` region
rather than the moment of approval. This is the condition `docs/02-hard-gate-pattern.md` §3 calls the one
that does most for audit credibility, and on Lists it is a refusal rather than a detection.

**It costs the two-save sequence from §2, and here that is a feature.** Record the approver; wait a moment
for the name to resolve into the hidden column; then set `Approved` to Yes. If you set both at once you are
refused with the message above. A canvas app collapses the two into one action.

⚠ **The residual gap is wider than a race, and it is the most important paragraph on this page.** This
formula guards **the save that sets `Approved` to Yes, and no other save.** Validation evaluates the submitted
item merged over the stored values, so any later save that touches only the *person* columns is judged against
the shadow values as they stand — which by then may be stale.

The version that needs no malice at all: an action is properly approved by Carol while Bob owns it, and then
somebody re-assigns `Owner` to Carol, which is an ordinary correction. `Owner_Email` still reads `bob`, so the
comparison passes and the save succeeds. F1 then tries to write `Owner_Email = carol` — **and this formula
refuses that write**, because it would make the item illegal. The shadow column stays `bob` for ever. The
record now shows an owner who approved her own action, and every query written against the shadow columns
says it is properly separated. The same mechanism lets `Approved_By` be swapped back to the raiser after
approval, and lets `Approved_By` be *cleared* entirely on an approved action.

**Two things follow, and neither is optional.**

**Give F1 and F2 the instruction F3 already has.** When a flow's corrective write to a shadow column would be
refused by this formula, it must **write `Approved` back to No in the same update**, so the item is legal and
the record visibly un-approves itself. A refused correction that leaves the record apparently approved is the
failure; a record that drops out of `Approved` and appears on somebody's list is the recovery. Log every one.

**The reconciliation must compare the shadow column against the person column it came from, not against the
other shadow columns.** Compared shadow-to-shadow, every case above looks clean — that is precisely what makes
it silent. `Approved_By_Email` versus `Approved_By`, `Owner_Email` versus `Owner`, on every action where
`Approved` is Yes. Any disagreement is either this, or somebody editing hidden columns in the grid, and both
need a person to look.

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
- that action has **`Approved` set to Yes**, which is the point at which the `CorrectiveAction` list has
  actually refused a self-approval.

⚠ **Test `Approved`, not `Approved_By`.** An earlier version of this specification said "carries an
`Approved_By`, which the list has already refused if it matched the raiser or the owner." That was false, and
it opened the whole chain: formula 3 short-circuits while `Approved` is No, so a corrective action can carry
the raiser's own name in `Approved_By` without any refusal at all. If F3 accepts that as approval, the
inspection closes self-approved and nothing anywhere in the build says no. `Approved` is a yes/no defaulting
to No, and it is the only column on that list whose value means a human made a decision.

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

⚠ **Two things about this lock that will catch you, and neither is obvious at design time.**

**The connection needs to be a *site* owner, not just a list owner.** **Stop sharing an item** removes every
permission except site owners', so a service account that owns the list but not the site locks itself out of
the item — and F3 is required to write `Stage` and `Closure_Ready` back onto exactly those closed records when
a closure is reverted. It fails silently, at run time, against the records that most need it. Make the service
account a site owner on the sandbox first and confirm F3 can still revert a locked record before you do it on
a live site.

**A reverted closure leaves a record nobody can work on.** When F3 pushes a closed inspection back to
`Dispositioned`, the item's permissions are still broken and everyone still holds read. The record now sits at
a stage that requires user work, and the user cannot do it. **The revert has to re-grant edit as well as
change the stage** — Grant access with the site members group in the `Members` role, restoring what closure
took away — and the flow should log every revert, because a reverted closure is an event somebody should
see.

## 4 · Build order on Lists

Ten steps. Earlier editions said "an evening if you have used a low-code forms tool, a weekend if not". The
kit has since been built by hand from these instructions, with a stopwatch. The plan drawn from that estimate
allowed 4.36 hours; the build took 8.78. The table below is what it actually cost.

### What Gate 1 actually cost to build

All figures are stopwatch-timed and self-reported. The whole-build and formula figures are from one person's
first build of Gate 1 — three lists and their columns, three validation formulas, four Power Automate flows
(F1a, F1b, F1c, F2a) — in a personal Microsoft 365 tenant, following `WI-M365-01` Part B by hand with no
scripting or click-automation. The build was done once; the tenant it was done in is no longer accessible,
and the system in the kit's screenshots was rebuilt by the same process in a second tenant. One build is one
data point. It is still the only measured one this kit has, and it is a better basis for a plan than the
guess it replaces.

| Part of the build | Allowed for in the plan | Actual | Note |
|---|---|---|---|
| Three lists and their columns (§2) | — | not separately timed | Column setup is repetitive rather than hard: about forty columns across three lists, each with settings that matter (step 3). The first build clocked the formulas and the whole; 8.78 h less 211 min leaves **316 min** for lists, columns and the four flows together, and that is the most that can be said about them from that build |
| Three validation formulas (§3) | **31 min** | **211 min** | Roughly **40 %** of the whole build. See below |
| Flow F1a · `Inspector_Email` on `Inspection` | — | **34.7 min** (31.5 carried from earlier clocks + 3.2 on the sheet) | The other three F1/F2 flows are copies of this one with the list changed. Passed 4 of 5 checks; the one it failed was the loop check — see §7, *The column-name trap* |
| Flow F1b · `Raised_By`, `Raised_On`, `Raised_By_Email` on `Nonconformance` | — | **3.9 min** build, **12.0 min** build and test | 5 of 5 checks |
| Flow F1c · `Owner_Email`, `Approved_By_Email`, `Completed_On` on `CorrectiveAction` | — | **7.7 min** build and test | 6 of 6 checks |
| Flow F2a · `NC_Reference_Text` and the reverse lookup | — | **21.8 min** | 1 of 4 checks at the time of record; the flow has still never run — see *Trigger conditions and the F2 write* below |
| **Whole Gate 1 build** | **4.36 h** | **8.78 h** | Two working days, not one evening |

**The flow figures are from a different clock.** They come from a second, instrumented build of the flows
alone, on 5 September 2026, with the flow build sheet running its own stopwatch: the four flows plus the
`Flow_Log` failure-log list (F0, 3.0 min) came to **79.2 minutes** in total. They do not sum with the 8.78
hours, which is the whole of the first build (28 August to 1 September 2026), and in that first build the
lists-and-columns and the four flows were not timed separately. No allowance for those rows was recorded —
hence the dashes.

**The formulas are the finding.** The plan budgeted the three formulas at 31 minutes — the time it takes to
paste three blocks of text into three settings pages, which is what "paste one formula per list" sounds
like. They took 211 minutes. The pasting is thirty seconds; the rest is the platform refusing the formula
until every column name resolves exactly, then the formula saving and doing something other than what you
read it to do, then the clause-by-clause grid testing in step 6 that is the only way to find out. **A plan
that books formulas as data entry is wrong by about a working day.** Book them as the design work they are,
and put step 6 inside the formula budget rather than after it.

The remaining flows (F2b, F3a/b/c, F4, F5) and the closure lock and archive job were **not** in the timed
build and carry no figure here — they have not been built yet. Do not read 8.78 hours as the cost of
everything in this document; it is the cost of the first working gate set, which is the point at which you
can run the grid tests and know whether the route holds.

### The steps

| # | Step | Why it is in this position |
|---|---|---|
| 1 | Create the three lists on a SharePoint team site | Not personal lists — the settings below do not exist there |
| 2 | Create every column with the **display name** used in `docs/01-data-model.md`, underscores included | Validation formulas take display names; matching the model means the formulas paste unchanged |
| 3 | Set the column settings the formulas assume: **clear the default on every choice column** (`Result`, `Severity` — a default is not a decision, and a defaulted `Result` makes clause 1 decorative), leave `Draft` as the default on `Stage`, turn **fill-in choices off** on `Stage` so no value can sidestep the `="Closed"` test, switch **Enforce unique values** on for `Reference`, `NC_Reference` and `CA_Reference`, and make `Quantity_Received` and the `Nonconformance` lookup on `CorrectiveAction` **required** | Each of these is a hole in a formula rather than a preference. Formula 1 compares against `Quantity_Received`, and formula 3's raiser comparison is defeated by a corrective action with no parent |
| 3b | Add the shadow columns (`*_Email`, `NC_Reference_Text`, `Closure_Ready`, `Stage_Last_Processed`) and hide them from forms. **Leave `Stage` visible and editable** — see the note below the table | Every formula in §3 depends on the shadow columns; `Stage_Last_Processed` is what lets F4 recognise its own write, and `docs/06-validation-and-test-plan.md` T14 assumes it exists |
| 4 | Build flows **F1** and **F2** below, **before** you write any formula | The formulas require columns only these flows fill; write the formulas first and nothing can be saved |
| 5 | Paste **one** formula per list — the whole of formula 1 on `Inspection`, formula 2 on `Nonconformance`, formula 3 on `CorrectiveAction` | A list holds one formula. Adding gates one at a time overwrites the one before it |
| 6 | Test each list's formula from the **grid**, not the form, and test every clause separately | The grid is the path that proves the rule is on the item; one formula means one message, so only clause-by-clause testing tells you all three gates are live |
| 7 | Build flows **F3**, **F4** and **F5**, each with a trigger condition | The infinite-loop anti-pattern is real; constrain the trigger before you point it at anything |
| 8 | Add the closure lock (stop sharing, then grant **Visitors**) under a list-owner connection | Needs owner rights; a member connection fails at run time, not at design time. `Members` would re-grant edit |
| 9 | Set up the archive job before you need it — and carry the original `Created` and `Created By` into columns of your own | 5,000 unique scopes arrives sooner than you think. Archiving is a create-plus-delete, so the copy is stamped with the service account and today's date; `docs/05-rollout-runbook.md` §8 says those two columns are why the records were worth anything. Copy them into `Original_Created` and `Original_Created_By` before the delete, or the archive is typed data of no evidentiary value. The job needs owner rights to delete permission-locked items |
| 10 | Re-run step 6 with a few thousand rows in the lists | Nothing above changes with volume, but anything you put in a canvas app does — see `docs/06-validation-and-test-plan.md` T16 |

### The sequence that actually worked

The table above is the order this document recommends. The first build from these instructions followed
`WI-M365-01`, whose Parts B to F run in a slightly different order, and that order worked. Where the two
differ, the differences are noted here rather than silently folded in, because a reader following the table
and a reader following the work instruction should both end up with the same system.

| WI part | What it does | How it maps onto the steps above |
|---|---|---|
| A (pre-flight) | Tenant with SharePoint, billing off, three test identities, a **team site**, the flow owner made a **site owner** | Step 1, plus the warning at the top of this document. Every one of these is cheap to check first and expensive to discover at step 8 |
| B · Build sheet | One list at a time — `Inspection`, then `Nonconformance`, then `CorrectiveAction`. For each: every column including the hidden shadow columns, the column settings, **then that list's formula**. Then the four flows F1a, F1b, F1c, F2a | Steps 2, 3, 3b and 5 done per list; step 4 done *after* the formulas rather than before. This works because clause 1 of every formula excuses `Draft`, so the lists accept Draft records while the flows do not yet exist. What you must not do is attempt any non-Draft save before F1 and F2 are running |
| C · Predictions | Twelve written predictions of gate behaviour, each recorded **before** its test is run | Step 6, made honest. Write down what you expect the grid to refuse before you type into it |
| D · Results | Screenshot every refusal, note the time, record what refuted you | Step 6's evidence. A refusal you remember is an anecdote; a refusal with a timestamp is what `docs/06-validation-and-test-plan.md` asks you to keep |
| E · Workbook | Build log (the clock), predictions, cost record, outcome measures, scorecard | Nothing in the table above asks for this, and it should. The build-hours figure in this section exists only because the clock ran |
| F · Release | Not part of the build | — |

Two things the work instruction does that the step table did not say plainly enough. First, **it puts the
stopwatch on every task**, start and stop, before and after, and refuses scripting or click-automation
because the manual figure is the one worth having. Second, **it writes the prediction before the test.** Both
cost minutes and both are the difference between "we built it" and "we watched it hold".

⚠ **On Route A the user sets `Stage`, and that is the design rather than a compromise.** It is tempting to
hide the column and have a flow advance the record — `docs/03-implementation-guide.md` describes it that way
in general terms — but on Lists nothing in this build can advance a record, and hiding `Stage` would leave an
inspector with no way to move one except the grid, which is the path step 6 uses as the *attack*. The honesty
here comes from validation, not from concealment: the user names the stage they are claiming, and the server
refuses the claim if the evidence is not there. A canvas app improves the experience and the refusal messages;
it is not what makes the gate hold. If you build one, it sets `Stage` too — through `Patch`, in one operation
with the person column, which is what collapses the two-save sequence in §2.

⚠ **The first end-to-end test cannot pass until step 7.** `Closure_Ready` is written only by F3, and formula 1
clause 3 requires it for any record whose result is not `Accept`. So at step 6 the closure clause can only be
tested negatively — confirm it refuses — and `docs/06-validation-and-test-plan.md` T01, the clean walk from
`Draft` to `Closed`, has to wait until F3 exists. Run T01 as the first thing after step 7, not as the first
thing in the suite.

### The flows this build needs

Every formula above depends on a column some flow writes, and until now those flows have been named in
passing rather than listed. Build all five before you trust any gate. None is more than a handful of actions,
and every one needs a trigger condition excluding the service identity, or it retriggers itself.

⚠ **This is a table of five *jobs*, not five flows.** A SharePoint item trigger binds to **one list on one
site**, so any row below that names more than one list is that many separate flows sharing one definition.
Built out, F1 is three flows, F2 is two, F3 is two — nine in total once F4 and F5 are added, plus the archive
job in step 9. Copy the first one and change the list; the logic is identical. Budget for nine, not five,
when you read the cost table at the top of this section — only four of them were in the timed build.

| # | Job | Trigger — one flow per list named here | What it writes | Why a gate needs it |
|---|---|---|---|---|
| F1 | **Resolve people, and stamp what nothing else stamps** | Item created or modified on all three lists | `Inspector_Email`, `Raised_By_Email`, `Owner_Email`, `Approved_By_Email` from the matching person column. **Also**, on creation: `Raised_By` from the built-in `Created By` and `Raised_On` from `Created`; and `Completed_On` when `Complete` becomes Yes | Formula 1 clause 1 and formula 3 read the shadow columns; nothing else writes any of these. ⚠ If F1 does not stamp `Raised_By`, `Raised_By_Email` resolves from nothing, F2 copies nothing down, and formula 3 then refuses **every** approval on **every** action, permanently |
| F2 | **Link and copy down** | Item created or modified on `Nonconformance` and `CorrectiveAction` | `NC_Reference_Text` on the inspection from the linked non-conformance; the reverse `Inspection.Nonconformance` lookup; `Raised_By_Email` from the non-conformance onto each of its corrective actions | Formula 1 clause 2 and formula 3's raiser comparison |
| F3 | **Maintain `Closure_Ready`** | Item created or modified on `Nonconformance` and `CorrectiveAction` — **and on `Inspection`**, or an inspection with no non-conformance is never visited at all (see the warning below) | `Closure_Ready` on the inspection — and, on a record that has already closed, `Stage` back to `Dispositioned` in the same update | Formula 1 clause 3 |
| F4 | **Stamp transitions** | Item modified on `Inspection`, where `Stage` changed | `Inspected_On` and the other transition timestamps | Nothing refuses a blank timestamp — see the warning below |
| F5 | **Reconcile** | Scheduled, nightly | Nothing. Reports mismatches | Every column above is a copy, and a copy can drift |

⚠ **`Closure_Ready` is an ordinary yes/no column, and hiding it from the form does not protect it.** There is
no column-level permission on this platform — the same fact that runs through this whole page — so a user in
grid view can write the flag directly. On a rejected receipt that has no non-conformance at all, three cells
close the record: type any string into `NC_Reference_Text`, set `Closure_Ready` to Yes, set `Stage` to
`Closed`. Clause 2 only tests that the reference is *present*, not that it resolves. **And with no
non-conformance and no corrective action in existence, F3 as originally triggered would never run against
that inspection**, so nothing would ever put the flag back — which is why F3 must also trigger on
`Inspection`. Even then the write happens first and the correction second. Give F5 two specific checks: any
inspection at or past `Dispositioned` whose `NC_Reference_Text` does not resolve to a real non-conformance,
and any inspection whose `Closure_Ready` is Yes without a complete non-conformance chain behind it. **Say this
one out loud rather than letting it be found** — the refusal on the flag is genuine, and what the flag means
is only as good as the flow that owns it plus the reconciliation that checks it.

⚠ **Every one of these writes is itself subject to the list's validation, and can be refused.** F2 writes
`NC_Reference_Text` onto an `Inspection` row and `Raised_By_Email` onto a `CorrectiveAction` row; both writes
go through the same formula a user's save does, so both can fail — silently, leaving the machinery column
blank, which is the condition the anchoring rule exists to defend against. **Give every flow in this table the
instruction F3 gets: log every write it attempted and could not make, and make that log something a person
reads.** A machinery column that is blank because a flow was refused looks exactly like one that is blank
because the flow has not run yet, and only the log tells you which.

⚠ **F4 needs the same site-owner connection F3 does, for the same reason.** F4 stamps the `Inspection` row on
every stage change — including the change *into* `Closed`, which is the same modification the closure lock
reacts to. Whichever wins the race, F4's write to a now-locked item fails on permissions, silently, at run
time, and the closure timestamp and `Stage_Last_Processed` are never written. Run F4 under the same site-owner
connection, and have it log the write it could not make.

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

### Trigger conditions and the F2 write — the exact expressions

These are the four Gate 1 trigger conditions and the F1c/F2a wiring as they stand after the review of
7 September 2026 **and the test session of 8 September 2026, in which all four flows were exercised against
records for the first time.** Three of the four conditions published in earlier drafts of this document were
wrong, and each was wrong in a way that no checker reported. Read §7 before you copy anything here.

The 7 September review found that every condition in the first build addressed the shadow columns by their
encoded internal name (`body/Inspector_x005f_Email` and so on) while the connector's trigger body in that
tenant used the plain key. The encoded reference resolved to null, `coalesce()` made it an empty string, and the
self-trigger guard was true on every save, including the flow's own write. §7, *The column-name trap that
hid inside the first build*, has the account of what that did; this subsection has the corrected text so
that nobody has to rediscover it. **The rule is: never write a trigger condition against a column key you
have not read back from a real run's trigger output — and never leave one in place until you have watched it
fire.** Save the flow with no condition, edit one item, open that run, and copy the key exactly as the body
shows it. Then put the condition back and make one qualifying change: if no run appears, the condition is
wrong, however reasonable it looks. Which form you get — plain or `_x005f_` — depends
on how the column was created and on the connector, not on what the list settings page displays, so the
keys below are what worked in one tenant and are the thing to check first in yours.

F1a, on `Inspection` — run only when the person column and its shadow disagree:

```
@not(equals(coalesce(triggerOutputs()?['body/Inspector/Email'], ''), coalesce(triggerOutputs()?['body/Inspector_Email'], '')))
```

F1b, on `Nonconformance` — run **only on creation**:

```
@equals(triggerOutputs()?['body/Modified'], triggerOutputs()?['body/Created'])
```

**This replaces the obvious version, which does not work.** Earlier drafts published
`@empty(coalesce(triggerOutputs()?['body/Raised_By_Email'], ''))` — fire while the shadow column is still
blank. On 8 September 2026 that flow was switched on and a new non-conformance created with
`Raised_By_Email` empty. It did not fire. Not on the create, not on a later modify, not at all. Removing the
condition made it fire on the next poll and stamp all three columns correctly, so the fault was the guard
alone.

The trigger output of that run says why: **a column that is null on the item is absent from the trigger
payload altogether.** The body carried `ID`, `NC_Reference`, `Description`, `Modified`, `Created`, `Author`
and `Editor`, and no `Raised_By`, `Raised_On`, `Raised_By_Email`, `Inspection`, `Severity` or `Cause_Level_*`
— every column that happened to be empty. A guard whose truth depends on a column being empty is therefore
asking about a key that is not there in exactly the state it is meant to catch.

`Modified` and `Created` are always present, and they are equal only on an unmodified record, which is what
"stamp the raiser on creation" means. The guard is also loop-safe by construction: the flow's own write
changes `Modified`, so it cannot re-trigger. Tested 8 September 2026 — one save, one run, three correct
stamps, `Raised_On` equal to `Created`, and no run at all for a later edit.

F1c, on `CorrectiveAction` — run when either shadow disagrees with its person column, or the action is
complete and not yet stamped:

```
@or(not(equals(coalesce(triggerOutputs()?['body/Owner/Email'], ''), coalesce(triggerOutputs()?['body/Owner_Email'], ''))), not(equals(coalesce(triggerOutputs()?['body/Approved_By/Email'], ''), coalesce(triggerOutputs()?['body/Approved_By_Email'], ''))), and(equals(triggerOutputs()?['body/Complete'], true), empty(coalesce(triggerOutputs()?['body/Completed_On'], ''))))
```

F1c's **Update item** on `CorrectiveAction`, Id = `triggerBody()?['ID']`. Read the two shadow columns as a
pair before you save — this is where the worst defect in the build was found:

```
CA_Reference        = triggerBody()?['CA_Reference']                  (required field, copied back unchanged)
Nonconformance Id   = triggerBody()?['Nonconformance/Id']             (required field, copied back unchanged)
Owner_Email         = triggerBody()?['Owner/Email']
Approved_By_Email   = triggerBody()?['Approved_By/Email']
Completed_On        = @if(and(equals(triggerOutputs()?['body/Complete'], true),
                              empty(coalesce(triggerOutputs()?['body/Completed_On'], ''))),
                          utcNow(),
                          triggerOutputs()?['body/Completed_On'])
```

**`Approved_By_Email` comes from `Approved_By`, never from `Owner`.** F1c was built as a copy of F1a via
*Save as*, and the copy carried `triggerBody()?['Owner/Email']` into both shadow columns. The flow therefore
wrote the **owner's** address into the approver's column on every run, on records with no approver set at
all. That column is one of the three that formula 3 adjudicates, so the defect put a wrong value into the
field the separation-of-duties gate reads — and because `Approved_By_Email` could then never agree with an
empty `Approved_By`, the second clause of the trigger condition was permanently true and the flow ran itself
every thirty seconds. One wrong source column, both a data-integrity fault in the gate and a loop. Corrected
and re-tested 8 September 2026.

**`Completed_On` reads `body/Completed_On`, not `body/Completed_x005f_On`.** The 7 September sweep rewrote
this flow's trigger condition with plain names and missed this action expression, which is a different place
in the same definition. Left as it was, `empty(coalesce(null, ''))` is always true and the flow re-stamps
`utcNow()` on every run instead of preserving the first stamp — a fault that only shows itself on the
*second* edit. **An encoding fix has to be applied by searching the whole definition, then re-tested.**

F2a, on `Nonconformance` — run only when the record is linked to an inspection:

```
@not(empty(triggerOutputs()?['body/Inspection']))
```

**This corrects the version earlier drafts called "correct in the first build".** That version was
`@not(empty(triggerOutputs()?['body/Inspection/Id']))`, and on 8 September 2026 a non-conformance was created
and linked to an inspection with the flow switched on. Nothing ran, on the create or the modify. Removing the
condition made it fire at once and complete the link write correctly.

The same path works inside the actions: `Get parent inspection` resolved
`@triggerOutputs()?['body/Inspection/Id']` to the parent's Id in that very run. So **an expression that
resolves inside an action can still resolve to null in a trigger condition, and the designer will not tell
you.** Do not read a rule about path depth into this — F1a's condition reads `body/Inspector/Email`, two
segments into a person column, and evaluates correctly in the same tenant. The reason for the difference was
not established. What was established is the procedure: prove the condition fires.

F2a's **Condition** inside the flow, after a *Get item* on `Inspection` named `Get parent inspection`
(Id = `triggerOutputs()?['body/Inspection/Id']`), joined with **Or** — proceed to the write if either the
copied-down reference or the reverse lookup is stale:

```
outputs('Get_parent_inspection')?['body/NC_Reference_Text']   is not equal to   triggerBody()?['NC_Reference']
outputs('Get_parent_inspection')?['body/Nonconformance/Id']   is not equal to   triggerBody()?['ID']
```

F2a's **Update item** on `Inspection`, Id = `triggerOutputs()?['body/Inspection/Id']`:

```
Reference           = outputs('Get_parent_inspection')?['body/Reference']            (required field, copied back unchanged)
Quantity_Received   = outputs('Get_parent_inspection')?['body/Quantity_Received']    (required field, copied back unchanged)
NC_Reference_Text   = triggerBody()?['NC_Reference']
Nonconformance Id   = triggerBody()?['ID']
```

**`Nonconformance Id` on the inspection write comes from the trigger — the non-conformance's own `ID` — and
never from `Get parent inspection`.** The first build wired that chip to
`outputs('Get_parent_inspection')?['body/Nonconformance/Id']`, which is the parent's *existing* value, so the
flow wrote back whatever was already there and the reverse lookup could never be set. The same review found
an empty third row in the condition and a flow still carrying its default name; all three were corrected on
7 September 2026.

F2a was tested on 8 September 2026, once its trigger condition was corrected: a linked non-conformance
produced exactly one run, the parent inspection received `NC_Reference_Text` and the reverse lookup, an
unlinked non-conformance produced no run at all, and a later edit took the false branch and wrote nothing.
The four P1 flows have now each been exercised against records. Run T17, T18 and T19 in
`docs/06-validation-and-test-plan.md` against your own build before you leave any of them switched on — the
conditions above are what worked in one tenant, not a guarantee about yours.

---

## 5 · What this route still cannot do

The honest ceiling, in one place, so you can decide whether it is high enough.

| Limit | Consequence | Live with it, or move? |
|---|---|---|
| No column-level permissions | Cannot lock three fields while leaving others editable. All-or-nothing at item level. | Live with it. Lock at closure, accept mid-stage edits are possible. |
| Validation cannot read another list | Cross-record rules become a flow-maintained flag the formula reads, or a flow check with revert. **The flag itself is an ordinary column with no permission on it**, so a grid user can write it and the formula will believe them | Live with it, and do not overstate it. The refusal on the flag is real; the flag's *honesty* is the flow's job **plus** a reconciliation that a person reads. Log both. |
| One formula and one message per list | Every gate on a list shares one refusal sentence | Live with it, and put per-gate messages in the app — `docs/10-canvas-app-gates.md` §3. |
| Multiple lines of text unusable in formulas | Any "this narrative field is not blank" gate needs a companion column or a flow check | Live with it. Prefer a single line of text where 255 characters will do. |
| Validation cannot see the previous value | No "field is frozen once its stage passed" before closure. **Concretely: `Result` can be changed from `Reject` to `Accept` on a dispositioned record, and the record then closes as an accepted receipt without ever being asked for the closure evidence a rejection requires.** The same applies to lowering `Quantity_Checked` or replacing `Inspector_Email` | Live with it and say so, **or** move — this is the clearest single symptom of having outgrown Lists. It is cheat 2 in `docs/03-implementation-guide.md` step 7, which has always said it succeeds. |
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
> without the evidence that stage requires. A causal analysis cannot be saved partially filled, or with any
> two of its three levels the same — it can be saved with none, and what forces it to exist is the closure
> gate rather than that rule. An action cannot be **marked** approved at a moment when the recorded approver
> is the person who raised the non-conformance or the person who owns the action; it can be re-assigned
> afterwards, and a nightly reconciliation compares the recorded approver against the value the rule tested
> and reports any disagreement. Conditions that span records are evaluated by an automated process which
> maintains a flag on the record, and the record cannot close while that flag is unset; the flag's history is
> logged, and because this platform has no column-level permission a user with grid access can also write
> that flag — the reconciliation is what catches it. Closed records are made read-only at closure. There is no
> column-level permission on this platform, so between stages a user with grid access can alter fields their
> earlier stage required; we have not claimed otherwise, and closure is the point at which the record becomes
> fixed."

That is a longer sentence than "the system enforces it", and every clause in it is true. The long true one
survives the follow-up question. Run `docs/06-validation-and-test-plan.md` and keep the results, so the
first clause is a fact you can show rather than a claim you make.

## 7 · What the first build taught

This kit was written before it was built. It has now been built once from its own instructions, by hand, in
a personal Microsoft 365 tenant, with a stopwatch running; that tenant is gone, and the system in the
screenshots was rebuilt by the same process in a second one. Everything below is from that first build and
is self-reported. It is one person and one build, and it should be read that way.

**The estimate was half the real figure.** 4.36 hours allowed, 8.78 hours taken, for three lists, three
formulas and four flows. Nothing in the build was hard in the sense of needing a skill the instructions did
not give; it was slow in the sense that every step had a check behind it — does the column name match, does
the setting exist, did the save refuse — and the checks are where the time goes. The old "an evening"
estimate counted the typing and forgot the checking. The table in §4 replaces it.

**Formulas are design work, not data entry.** 211 minutes against 31 allowed, about 40 % of the build.
Three blocks of text that fit on one screen took longer than the forty columns they test. The reason is in
§1 and §3: a formula that names a column the list cannot resolve is refused at save, and a formula that
saves is not thereby correct — it has to be read back clause by clause against `docs/01-data-model.md` and
then proved from the grid. Budget a working day for the three of them and you will be about right; budget
half an hour and you will be about a day out.

**The first wall is not in this document, it is before it.** Validation settings do not exist on a personal
list. A reader who opens the Lists app and presses *New list* is in "My lists", will build everything in §2
without a warning, and will find at step 5 that there is nowhere to paste the formula. That is now the first
thing in this document, and it is the first check in the work instruction, because it costs nothing to get
right and a rebuild to get wrong.

**The order can flex; the dependency cannot.** The step table says flows before formulas; the work
instruction did formulas with each list and the flows afterwards, and it worked, because every formula
excuses `Draft`. What does not flex is that no non-Draft save can succeed until F1 and F2 are running. Pick
either order and respect that.

**Measure it or you will not have measured it.** The build-hours figure exists because a clock was started
before each task and stopped after it, with the automation switched off. The measurement workbook that
accompanies the work instruction has a row for each task and asks for the prediction before the test. Fill
it in the same sitting. A figure reconstructed afterwards is an estimate with a decimal point.

### The column-name trap that hid inside the first build

Every trigger condition in the first build addressed the shadow columns by their encoded internal name —
`body/Inspector_x005f_Email`, `body/Raised_x005f_By_x005f_Email`, and so on — because that is what
SharePoint calls a column created through the UI with an underscore in its name. In the tenant the flows
ran in, the SharePoint connector's trigger body used the **plain** key (`Inspector_Email`). The encoded
reference resolved to nothing, `coalesce` turned nothing into an empty string, and the self-trigger guard
was therefore true on every save — including the flow's own write. F1a ran itself every thirty seconds
for two days and pushed one test record to version 2,938 before the loop was noticed; F1b did the same
until the tenant's trigger quota cut it off. The flows passed every check except the loop check, which is
the one that was left unticked.

The rule that comes out of it: **never write a trigger condition against a column name you have not read
back from a real run.** Open one run of the trigger, read the key exactly as the body shows it, and use
that. Which form you get depends on how the column was created and on the connector, not on what the
list settings page displays.

### What the first test session taught — 8 September 2026

The four P1 flows were exercised against records for the first time on 8 September 2026. All four had
faults. None of the faults was visible in the designer, the flow checker, or the flow's Status.

**A null column is absent from the trigger payload, not present-and-empty.** Read back from a real run: a
non-conformance with most fields blank produced a body carrying only `ID`, `NC_Reference`, `Description`,
`Modified`, `Created`, `Author` and `Editor`. Everything null was simply missing. The consequence is a trap
with a particular shape: **a guard whose truth depends on a column being empty is asking about a key that is
absent in exactly the state it is meant to catch.** F1b's published condition had that shape and never fired.

**Silent non-firing is a failure mode, and it looks like success.** The loop of the first build was loud —
a version number climbing on its own. Its opposite is silent. F1b and F2a each sat On, with a clean checker,
a green Status and an empty run history, doing nothing, for as long as nobody thought to ask. **"No runs" is
a defect until proven otherwise, not a quiet system.** The check that catches it is one qualifying change and
one look at the run history.

**A flow made by *Save as* inherits the original's field mappings, and a wrong source column is invisible in
the card view.** F1c was copied from F1a and carried `Owner/Email` into *both* shadow columns, so it wrote
the owner's address into the approver's column — one of the three columns formula 3 adjudicates — on records
with no approver. That also made the second clause of its trigger condition permanently true, so the flow
looped. One wrong source, a corrupted gate input and a loop together.

This is worth holding next to the build figures in §4. Copying F1a is why F1b took 3.9 minutes against
F1a's 34.7, and copying F1a is also how this defect propagated. The speed is real and so is the cost; a
build sheet that reports one without the other is not telling you the truth about the method. **List every
written column beside its source expression and read them as a pair before you save.**

**An encoding fix must be applied across the whole definition.** The 7 September sweep corrected F1c's
trigger condition and missed a `Completed_x005f_On` in the same flow's Update item expression. Symptom-led
fixes leave siblings behind. Search the definition, fix every hit, then re-test.

**Match a list by its GUID, not by its name.** The site the build ran in held two sets of lists with
overlapping names — a list titled `Inspection` living at `/Lists/Inspection1`, and a nine-item decoy titled
`Inspection_01` at `/Lists/Inspection`. A first pass at the loop check read the wrong record and would have
reported the loop as fixed on evidence from a list no flow touches. Take the table GUID from the flow
definition and confirm the list from that.

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
