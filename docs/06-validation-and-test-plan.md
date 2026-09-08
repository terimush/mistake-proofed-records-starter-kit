# Validation and test plan

A way to find out whether the thing you built does what you think it does — before you point it at real work,
and as evidence afterwards that you checked.

---

## 1 · What this is, and what it is not

This is a **functional test plan for a records pattern**: deliberate attempts to break your own build, an
expected result for each on both platform routes, and a log you keep.

It is **not software validation in the regulated sense**. Not computer system validation, not GAMP
qualification, and nothing to do with 21 CFR Part 11. Running every test here does not produce a validated
system and is not evidence that your software is fit for a regulated use. Obligations of that kind need a
qualification process this document does not attempt — validation plan, defined requirements, IQ/OQ/PQ or an
equivalent, change control. Nor is any of it certification or compliance; `docs/04-compliance-mapping.md` sets
that boundary. What it gives you is the difference between "I built it" and "I can show it works, and I can
tell you where it does not."

## 2 · Before you test

**Use a sandbox, not your live tenant.** Several tests write bad data on purpose, one tries to delete a closed
record, and one provokes a trigger loop.

**Fabricated data only.** Use the sample CSVs in `examples/`, or invent your own in the same style —
`Supplier Nine`, `BRACKET-A1`, `RC-2041`. Test records outlive the test.

**Two test identities, minimum — and T10 needs a specific pair.** T09 and T10 cannot be run with one account,
which is why people skip them. T10 additionally needs two accounts whose *display names are identical* while
their underlying identifiers differ, which is a directory change many readers cannot make in a corporate
tenant. Arrange it before you start, or accept that T10 is untestable in your environment and log it as such
rather than as a pass. It is worth the effort:
and self-approval is the control that does most for credibility.

**Know which route you are on, and where your rules live.** Route A is Microsoft Lists with Power Apps and
Power Automate; Route B is Dataverse.

Three tests do not follow the general "refused on both routes" pattern, and knowing which is which is most of
the value of the suite:

- **T12** (edit a field after its stage passed) and **T13** (type over a system-set date) are expected to be
  **permitted on Route A** and refused on Route B. Lists has no column-level permission. Log them
  `ALLOWED-PL`; they are not build defects.
- **T10** (two identities sharing a display name) is expected to be **permitted on both routes**. A refusal
  there means your comparison is on names rather than identities.

Everything else should be refused on either route, **provided your gate conditions live in list validation
rather than in a form, and on the list that owns the columns they test**. If a test that should be refused
succeeds on Route A, that is usually a build defect rather than a platform limit; see
`docs/09-microsoft-lists-build.md` §3 before you record it as one.

**And if any part of your enforcement lives in a canvas app, T16 is not optional.** The suite passes on
twelve fabricated rows whether or not your gates survive a real list.

## 3 · How to run the suite

Work through the tests in order; several depend on the state left by the one before. Run them as an ordinary
user, never as an administrator — testing enforcement from an account that bypasses it tells you nothing.
Record what happened rather than what you expected, and capture a screenshot at each step.

**On Route A, test every clause separately and do not stop at the first refusal.** A SharePoint list holds one
validation formula and one message, so all of a list's gates are clauses of a single `AND(...)` and every
failure returns the same sentence. A record that is refused tells you *some* clause fired, not that all of
them are present — which matters because the commonest Route A build error is pasting gate formulas one at a
time and silently replacing the previous one. T02 to T08 are written one condition at a time for exactly this
reason. Run all of them.

Log each attempt as `BLOCKED` (refused — a pass wherever the expected result says refused), `ALLOWED-PL`
(succeeded, and your route's expected result says it would) or `ALLOWED-BUILD` (succeeded where it should have
been refused; fix and re-run).

## 4 · The test suite

The conditions under test are those in `examples/incoming-inspection-checklist.md`. Gate 1 in short form:

```
REQUIRE  Quantity_Checked  is not blank
     AND Quantity_Checked  <= Quantity_Received
     AND Result            in {Accept, Reject, Use-as-is, Rework}
     AND Inspector         is not blank

ON PASS  set Inspected_On = now()
         lock Quantity_Checked, Result, Inspector
```

### T01 · A clean record walks the whole path

**Proves.** The positive path works, and nothing built to refuse bad data also refuses good data.

⚠ **Run this one after the flows exist, not first.** On Route A, `Closure_Ready` is written only by flow F3,
and formula 1's closure clause requires it for any record whose result is not `Accept`. So this test cannot
complete until step 7 of `docs/09-microsoft-lists-build.md` §4 is done. It is numbered first because it is
the test you care about most, not because it is the one to run first.

**Steps.** In empty sandbox tables, create `RC-2041`, 500 units of `BRACKET-A1` from `Supplier Nine`. Quantity
checked 50, `Result = Reject`, inspector set; advance. Raise `NC-0312` with description, severity and
containment; advance. Three causal levels, `CA-0155` with owner and due date, approved by the *second*
identity; advance.

**Expected — both routes.** Reaches `Closed` with a system-set `Inspected_On`.

**Expected — Route A, one extra step you have to know about.** Set the person column and *then* the stage, in
two saves. The shadow email columns the formulas compare are written by a flow after the item is saved, so a
save that sets `Inspector` and `Stage = Inspected` together is refused — the column the rule reads is still
blank at that moment. Same at closure with the approver. This is a property of the platform, not a defect;
`docs/09-microsoft-lists-build.md` §2 explains it, and a canvas app hides it by doing both in one `Patch`. If
your record will not advance and every visible field is filled, this is why.

**If it differs.** Usually a blank-versus-empty-string comparison, a lookup resolving to an identifier your
condition tests as text, or the two-save sequence above. Fix it first; every later test assumes it passed.

### T02–T04 · Gate 1 with a required field blank

**Preconditions.** A fresh `Draft` record with the other two Gate 1 fields completed. One run per test.

**Steps.** Attempt to advance to `Inspected`.

**Expected — Route B.** Refused every time, with a message naming the missing field.

**Expected — Route A.** Refused every time, with **the list's single message**, which names all three gates
rather than the field you missed. A list holds one formula and one message; §3 says so and the tests are
written for both routes. Judge this test on the refusal, not on the wording.

**If it advances anyway.** **T02, `Quantity_Checked`:** the condition tests the field's presence rather than
its value, or the advance control is not wired to the rule. **T03, `Result`:** choice columns carry defaults,
and a default is not a decision. **T04, `Inspector`:** check the column type — a text column taking a typed
name passes while giving you nothing that survives scrutiny (`docs/01-data-model.md`).

### T05 · Quantity checked greater than quantity received

**Steps.** With quantity received 500, enter quantity checked 600 and advance. Repeat with 0. Then repeat
with the field left genuinely empty.

**Expected — both routes.** 600 refused. **Zero is *not* refused by the formula as written** — `NOT(ISBLANK(0))`
is satisfied and `0 <= Quantity_Received` holds — so a record can reach `Inspected` with nothing inspected.
Decide deliberately whether zero is legitimate in your process; if it is not, add `[Quantity_Checked] > 0` to
clause 1 and set the column's minimum value to 1. Decide it now rather than discovering it, and
find out, rather than assume, whether your platform can tell an empty number column from a typed zero. The
last of the three attempts is the one that answers it: if an empty `Quantity_Checked` is *permitted*, your
`is not blank` condition is not doing anything on number columns and you need a different test — a minimum
value, or a companion yes/no the form sets.

**If it differs.** Numeric comparison against a text column succeeds silently. So does a blankness test on a
number column that the platform treats as zero.

### T06 · Non-accept result with no linked non-conformance

**Preconditions.** Record at `Inspected`, `Result = Reject`, nothing linked.

**Steps.** Advance to `Dispositioned`. Repeat with a non-conformance linked but `Containment` blank.

**Expected — Route B.** Refused both times.

**Expected — Route A.** Refused the first time. **The second time it succeeds, and that is the platform.**
`Containment` is multiple lines of text, which validation cannot read at all, so no formula on the
`Nonconformance` list can require it. If you want that gate on Route A you have to add it yourself — either
make `Containment` a single line of text and add the condition to formula 2, or have a flow check it and
maintain a yes/no the formula reads. `docs/09-microsoft-lists-build.md` §1 sets out both. Until you do,
write down that this one is open.

**If it differs.** A gate checking that a link exists, but not what is behind it, passes empty records.

### T07 · Two causal levels instead of three

**Proves.** Enforced causal depth. Must hold on both routes, because it lives in your rules rather than in
permissions.

**Preconditions.** Record at `Dispositioned`, and a non-conformance you are trying to give
`Cause_Level_1` and `_2` only.

**Steps.** Attempt to **save the non-conformance** with two causal levels. Then attempt to save it with three
levels where the first and third are the same word.

**Expected — both routes.** Refused both times. Note *where* the refusal happens on Route A: formula 2 is a
coherence rule, so the partial analysis cannot be saved **at all** — you never get as far as advancing the
inspection. That is stricter than this test originally assumed, and it is the intended behaviour. The
same-word case is what the third comparison in formula 2 exists to catch.

**If it differs.** Your gate is decorative. Return to `docs/02-hard-gate-pattern.md` §2 before anything else.

### T08 · Corrective action with no owner, and with no due date

**Preconditions.** Record at `Dispositioned`, three causal levels present, a corrective action with `Owner`
cleared.

**Steps.** Advance. Restore the owner, clear `Due`, advance again.

**Expected — Route B.** Refused both times: *"A corrective action needs an owner and a due date."*

**Expected — Route A.** Refused both times — **but not by the `CorrectiveAction` list, and not immediately.**
Nothing in formula 3 reads `Owner` or `Due`; those two conditions live in flow F3, which withdraws
`Closure_Ready`, after which formula 1 refuses the *inspection's* close with the **`Inspection`** list's
message. So the refusal is asynchronous and it arrives from a flow. ⚠ **Wait for F3 to complete before you
advance**, or you will close the record inside the window and log a build defect that does not exist. Record
this one as flow-enforced rather than validation-enforced; the distinction is the whole point of §5's log.

**If it differs.** An action with no owner is a wish, and one with no due date is a wish with better grammar.

### T09 · Self-approval

**Proves.** The control that does most for audit credibility. **Requires two identities.**

**Preconditions.** `NC-0312` raised by identity 1, `CA-0155` linked with owner and due date; signed in as
identity 1.

**Steps.** Set `Approved_By` to yourself, wait for the shadow column to resolve, then set `Approved` to Yes.
Then sign in as identity 2 and do the same.

**Expected — both routes.** Refused for the raiser and permitted for identity 2.

⚠ **Pin the owner in the preconditions: identity 1 owns the action, identity 1 raised the non-conformance,
identity 2 approves.** Formula 3 requires the approver to differ from the **owner** as well as the raiser, so
the obvious two-identity setup — identity 2 as both owner and approver — is refused *correctly* and will be
logged as a build defect that is not one. `examples/sample-corrective-actions.csv` row `CA-0155` is the
arrangement that works.

⚠ **Run the lazy way on a *fresh* action, not on the one you have just approved.** On an already-approved
action the lazy save succeeds on a correctly-built system, because validation reads the stored shadow value —
see the residual-gap paragraph in `docs/09-microsoft-lists-build.md` §3. That is a real weakness and it is a
different one; testing it here will tell you your build is broken when it is not.

**And run it the lazy way as well, because that is the way that finds the defect.** Set `Approved_By` to
yourself and `Approved` to Yes in the *same* save, without pausing. That must also be refused. If it
succeeds, your rule is keyed on the approver's shadow email being present rather than on `Approved`, the
shadow column is blank at the moment of the save, and you have a gate that passes every self-approval made in
one action while refusing the ones made in two. Open the record afterwards: if `Approved_By_Email` is still
blank on an approved action, that is the signature of this fault.

**If it differs.** This must hold on Route A too: the comparison is not running, `Raised_By` is not a resolved
identity, or the rule is anchored on the wrong column — see `docs/09-microsoft-lists-build.md` §3 formula 3.

### T10 · Two identities with the same display name

**Proves.** That the comparison is on directory identifiers rather than names.

**Preconditions.** Two accounts both displaying as `Person A`, with different underlying identifiers.

**Steps.** Raise a non-conformance as the first `Person A`, approve as the second, advance.

**Expected — both routes.** **Permitted** — two different people, and a correct comparison lets it through.

**If it differs.** A refusal means you are comparing display names. That fails in the other direction too: a
name comparison also *permits* real self-approval whenever one person holds two accounts.

### T11 · Direct write to the table

**Proves.** Whether enforcement lives with the data or with the screen. This result decides what you may say
about the whole system.

**Steps.** As an ordinary user with edit access, open the underlying table directly — datasheet or grid view,
or any client other than your form. Set `Stage` to `Closed` on a **half-empty** `Draft` record — one with no
result and no inspector — and save. The word *half-empty* matters: a `Draft` record that happens to be
complete closes legally, because there is no transition-order enforcement on this route
(`docs/09-microsoft-lists-build.md` §5),
and a tester who uses a complete record will condemn a correct build.

**Expected — Route B.** Refused. Column security grants Update on `Stage` only to the identity your rules run
under, so the write does not land.

**Expected — Route A.** **Refused, if you built the gates as list validation formulas.** This is the test
that surprises people. Lists has no column-level permission, so nothing stops the *write* to `Stage` — but
the server evaluates list validation on save, and a formula written as "a record at this stage must have
these fields" makes the submitted item illegal. The write is rejected whichever client sent it. See
`docs/09-microsoft-lists-build.md` §3 for the formula shape.

**If it differs.** On Route A, a successful write almost always means your gate is in the form rather than in
list validation settings — which is the failure this whole kit exists to warn about, and it is worth stopping
to fix before you go further. It can also mean the formula on that list is not the one you think it is:
validation settings hold a single formula, so a gate pasted after another one replaced it. Open the settings
page and read what is actually there. On Route B, a successful write means column security is not enabled on
`Stage`, or your account bypasses the profile.

### T12 · Editing a locked field after its gate has passed

**Proves.** That earlier stages lock on passage.

**Preconditions.** A record at `Inspected`, so Gate 1 has passed and its fields should be locked.

**Steps.** Change `Result` from `Reject` to `Accept`, first through the form, then through the grid view.

**Expected — Route B.** Refused by both paths.

**Expected — Route A.** Refused through the form if you removed the field on advance. **Succeeds through the
grid view**, the same limitation as T11. Log it `ALLOWED-PL`.

**If it differs.** If the form itself permits the edit on either route, build step 4 was not completed.

### T13 · Timestamp integrity

**Steps.** Three attempts on a record that has just passed Gate 1. Look for `Inspected_On` as an editable
control on any form — it should not appear. Try to type into it through the grid view. Then set your
workstation clock forward a week, advance a fresh record, and compare the stamped time against real time.

**Expected — Route B.** Not editable by an ordinary user, and the value reflects server time whatever the
client clock says.

**Expected — Route A.** Editable through the grid view — `ALLOWED-PL` again. The clock result should still
be clean, because the value is stamped where the rule runs. Check how your platform stores time zones before
reading an offset as tampering.

**If it differs.** A timestamp that follows the client clock is supplied by the app rather than the platform.
Fix it; a hand-influenced date destroys the property that makes the chronology worth anything.

### T14 · The transition rule does not retrigger itself

**Proves.** That you handled the infinite-trigger-loop anti-pattern rather than dismissing the warning.

**Steps.** Perform one invalid transition that makes the rule write back to the record, then count the runs
in the run history.

**Expected — both routes.** One triggering run, plus at most one further run that terminates on detecting its
own write. A trigger condition of this shape produces that:

```
RUN ONLY WHEN  Stage has changed
          AND  the modifying identity is not the service identity the rules run under
```

**The second half of that is expressible directly; the first half is not, on SharePoint.** A trigger condition
is an expression over the item as it now stands, and the item does not carry its own previous value — the
same limit that stops validation seeing it. So compare the modifying identity in the trigger condition, and
detect "`Stage` has changed" inside the flow: keep a `Stage_Last_Processed` column that only the flow writes,
terminate when it already matches `Stage`, and set it before you write anything else. That gives you the same
two-runs-and-stop behaviour this test looks for, and it is why the test counts runs rather than inspecting the
condition.

**If it differs.** A run history that keeps growing after you stop touching the record is the loop, and it
will throttle your tenant if it reaches a real list. Fix it before any other test.

### T15 · Deleting a closed record

**Steps.** As an ordinary user, attempt to delete a closed record. Then look for evidence that the attempt
happened: recycle bin, version history, platform audit log.

**Expected — both routes.** Deletion is governed by permission rather than by your gates, so the outcome
depends on what you granted. The second half matters more: establish whether the deletion is recorded
somewhere retrievable, and whether that logging is on by default or a separate configuration or licensing step
on your tier.

**If it differs.** If an ordinary user can delete a closed record and nothing records that it happened, the
retention story is weaker than the gate story, and gating does not compensate.

### T16 · The suite again, at volume

**Proves.** That your gates still decide correctly once the lists are bigger than a demonstration. This is
the test most likely to change your answer, and the one nobody runs.

**Why it exists.** If any part of your enforcement lives in a canvas app, the app does not evaluate queries
over the whole list — it pulls a bounded page and evaluates locally when a query cannot be handed to the data
source. The default is 500 rows. Against SharePoint, `CountRows`, `Sum`, `Search` and `In` do not delegate,
and `IsBlank` does not delegate on text columns, so any gate built from them decides on a partial view once
the list outgrows the limit. Nothing warns the user, nothing fails, and the gate quietly starts passing
records it should refuse. `docs/10-canvas-app-gates.md` §4 has the detail and the fix.

**Steps.** In the sandbox, load each list to at least 3,000 fabricated rows — the check table first, since it
grows fastest. Re-run T01 and every negative test. Pay particular attention to any gate that counts,
compares against a count, or tests a field for blankness.

**Expected — both routes.** Identical results to the small-data run. Any test that passes at twelve rows and
fails at three thousand — or worse, *passes wrongly* at three thousand — is a delegation defect, not a data
problem.

**If it differs.** Rewrite the query so the filter runs on the server and the aggregate runs on the small
result: filter with `=` on a text column, collect, then count the collection. Raising the data row limit to
2,000 buys you time and is not a fix.

**Log this test separately in the results record**, with the row count you tested at. "Verified at 3,000
rows" is a materially different statement from "verified", and only one of them survives the question.

### T17 · A flow that writes to its own list runs once and then stops

**Proves.** That the self-trigger guard on each F-flow actually excludes the flow's own write — not that a
condition is present, which T14 already covers for the transition rule, but that the condition's operands
resolve to something on your tenant. Run it for every flow that writes to the list it triggers on: on Route
A that is each of the F1 flows, F2, F3 and F4, one at a time, and it must pass before that flow is left
switched on.

**Why it exists.** A trigger condition that names a column key the connector does not use — the encoded
`_x005f_` form where the trigger body carries the plain key, or the reverse — does not fail. It resolves to
null, `coalesce()` turns null into an empty string, and a guard built on that is true on every save. Every
run then succeeds, which is why nothing in the run history looks wrong. A failing version of this test, run
after the fact on the kit's own first build, showed the flow had been running itself every thirty seconds
for two days and had carried one test record to **version 2,938**. `docs/09-microsoft-lists-build.md` §4
has the corrected conditions and §7 the account.

**Steps.** With the flow switched on and nothing else touching the list, edit one field on one record — the
field the flow reacts to — and note the record's version number. Open the flow's run history and count.
Wait five minutes without touching the record. Count again, and read the version number again.

**Expected — both routes.** Exactly one run for the edit. No further runs in the five minutes. The version
number has climbed by the runs you can account for — your edit and the flow's one corrective write — and
no more.

**If it differs.** A second run seconds after the first, or any run at all during the five minutes, is the
loop. Switch the flow off before anything else, let the queue drain, then open the trigger output of one
run and read the column keys exactly as the body shows them; rewrite the condition with those keys and run
this test again. Do not switch the flow back on until it passes. A version number that keeps climbing on a
record nobody is editing is the same finding seen from the list, and it is the check to add to F5.

## 5 · The results record

Fill this in as you go and keep it, headed with the environment name and the date.

| Test | Date | Tester | Route | Result | Evidence |
|---|---|---|---|---|---|
| T01 | | | | | |
| T02 | | | | | |
| T03 | | | | | |
| T04 | | | | | |
| T05 | | | | | |
| T06 | | | | | |
| T07 | | | | | |
| T08 | | | | | |
| T09 | | | | | |
| T10 | | | | | |
| T11 | | | | | |
| T12 | | | | | |
| T13 | | | | | |
| T14 | | | | | |
| T15 | | | | | |
| T16 | | | | | | ← record the row count you tested at |
| T17 | | | | | | ← one row per flow tested |

"Evidence" must point at something retrievable: a screenshot file name, an exported record, a flow run
identifier. A tick in a box evidences only that someone had a pen.

## 6 · The description you give an auditor

Derive it mechanically from the log. Do not improve it.

Note that T10 is the one test whose passing result is **permitted**, not blocked — it proves the comparison is
on directory identity rather than display name, so a refusal there means your comparison is broken. Read the
rows below with that in mind.

**Route B only — everything blocked except T10, which was permitted, and T16 run at volume.** "Transitions
are enforced on the table by server-side rules and column permissions. A user cannot set the stage directly or
alter a completed stage — here are the records."

This row is unreachable on Route A by construction, because T12 and T13 are *expected* to be permitted there
and Route A has no column permissions. If you are on Route A and every test blocked, something is wrong with
the test rather than right with the build — you probably ran it as an administrator, or against the form
rather than the grid.

**If T16 was not run**, strike any claim about what the system does and replace it with what it did on the
data you tested. A suite passed at twelve rows describes a demonstration.

**Route A, done properly — everything blocked except T10, T12 and T13, and T16 run at volume.** This is the
best available result on the free tier and it is a good one. Use the wording in
`docs/09-microsoft-lists-build.md` §6, in full — nothing shorter, because the clause about mid-stage edits is
what makes the rest of it survive a follow-up question.

**T07 or T09 allowed.** "There is a form with a status field." Nothing stronger, and fix it before the
conversation rather than during it. On Route A, check first whether the formula you think is on the list is
actually there: validation settings hold one formula, and a gate pasted after another replaced it.

**"The system enforces it" is only true if the attempt in T11 was refused.** Where the direct write succeeded,
the gates are enforced against people using the form — a smaller claim, and this is its wording:

> "Stage advances are controlled by rules in the application and by a rule that runs whenever a record
> changes. On Route B the status column is not exposed on any form; on Route A the user sets it and the server
  refuses any item not legal at the stage claimed. Either way it is not permission-protected: this platform
> tier has no column-level permissions, so a user with edit access to the list can set the status directly.
> That is tested and logged as a known limitation, and managed by restricting edit access and reviewing the
> change history."

The alternative is an auditor finding it themselves, after you claimed otherwise.

## 7 · When to re-test

Re-run the suite — or at minimum T07, T09, T11 and T14, the four that most often break quietly:

- **After a platform update** touching your forms layer, rules engine or permission model.
- **After any change to a gate condition.** A condition edited to fix one refusal routinely loosens another.
- **When a new form, client or integration writes to the same tables.** This is the one people miss: a gate
  enforced at the transition survives a new client, one enforced at the form does not, and a mobile layout, a
  bulk import or an ERP interface is a new client.
- **When a role changes** such that a new group holds edit access.
- **Periodically anyway.** Annually is defensible.

Keep the old logs. A dated series answers "how do you know it still works" better than one perfect result.

## 8 · What this test plan does not do

**It does not validate software.** Not in the CSV, GAMP or Part 11 sense, and not in any sense that would be
accepted as qualification. See §1, and take it literally.

**It does not test your process design.** Every test asks whether the software behaves as specified, not
whether the stages match how your shop works. A system can pass every test and still record the wrong things
diligently.

**It does not cover security, or scale.** No penetration testing and no review of the permission model beyond
the columns named; administrators are out of scope throughout. Fifteen fabricated records tell you nothing
about fifty thousand.

**It does not make records unalterable, and nothing can.** A sufficiently privileged administrator can change
or delete anything in these platforms. The available claim is narrower and more defensible: ordinary users
cannot bypass the gates, and alterations leave a trace.

**It is not a substitute for advice.** If your contracts or your regulator impose qualification obligations,
speak to someone who does that work. `README.md` lists free sources that are better first calls than any
vendor, this one included.
