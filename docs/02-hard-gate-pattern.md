# The hard-gate pattern

This is the part worth understanding. Everything else in the kit is scaffolding around it.

---

## 1 · What a gate is

A record moves through **stages**. Between each stage sits a **gate**: a condition that must be true before
the record is permitted to move on.

```
Draft ──▶ Inspected ──▶ Dispositioned ──▶ Closed
     gate 1        gate 2            gate 3
```

A gate is written as a plain condition over the record's own fields:

```
Gate 1 (Draft → Inspected):
    Quantity_Checked  is not blank
AND Quantity_Checked  <= Quantity_Received
AND Result            is one of {Accept, Reject, Use-as-is, Rework}
AND Inspector         is not blank

Gate 2 (Inspected → Dispositioned):
    IF Result = Accept          → no further evidence required
    IF Result ≠ Accept          → Nonconformance record exists AND is linked

Gate 3 (Dispositioned → Closed):
    IF a Nonconformance is linked
        → it has at least 3 causal levels recorded
        AND a corrective action with an owner and a due date
        AND a different person in Approved_By than in Raised_By
```

That is the whole idea. The rest of this document is about making the gate real rather than decorative.

## 2 · Where to enforce it — and this is where most implementations fail

There are three places you can enforce a gate, and they are **not** equivalent.

**Weakest — hide or disable the button.** The user can't click "Advance" until the fields are filled. This
is the version most people build, and it is the version that fails an audit, because it is a *suggestion*.
Anyone who can open the underlying table can change the status directly and the gate never runs.

**Better — validate on save.** The form refuses to commit a record whose stage does not satisfy its gate.
Stronger, but still tied to the form. If a second form, an import, a mobile client, or an integration writes
to the same table, the gate is not there.

**Correct — enforce on the data, at the transition.** The rule lives with the record, not with the screen.
Any path that attempts an invalid stage change is rejected, whichever client attempted it. In practice this
means a low-code business rule, a validation rule on the table, or a flow that runs on change and reverts a
transition that fails its gate — and it means the status column is not directly editable by ordinary users
at all; it only changes as a *result* of a gate passing.

Build the weakest version too, but only as courtesy. A user should be told what's missing before they try.
The gate that counts is the one behind it.

**One exception in principle, and it is a permissions exception rather than an app one.** The weakest
version would become strong if the screen were genuinely the *only* way in — if ordinary users held read
access to the store and every write went through the application, running under an identity they do not have.
Then there would be no other path for the rule to be absent from. Note what would actually have changed: not
the gate, which is still a check in a form, but the set of doors. Close one door and reopen another — a
second app, an import, a permission grant made in a hurry — and every gate you built goes back to being a
suggestion, silently.

On the Microsoft stack that arrangement rests on an assumption nobody here has confirmed, so it sits in the
appendix to `docs/10-canvas-app-gates.md` as something to test rather than something to build. Do not plan
around it until you have proved it on your own tenant.

## 3 · The three mistakes that make a gate decorative

**Editable history.** If a record can be changed after it closes, it is a form and not evidence. Once a stage
passes, lock the fields that stage required. Later stages stay open; earlier ones do not. If a genuine
correction is needed, that is a new versioned record with a reason, not an overwrite.

**Self-approval.** A gate requiring approval that the same person can satisfy is not a control. Compare the
approver against the raiser and reject a match. This one line does more for audit credibility than any
amount of workflow decoration.

**Optional depth.** "Root cause" as a single free-text box gets filled with the symptom restated. If you want
causal analysis, require structured depth: three linked levels, each referencing the one above. It is
mildly annoying to complete, which is the point — the annoyance is the analysis.

## 4 · Two design principles

**Gates should be few and load-bearing.** Every gate is friction, and friction that does not protect
something real gets routed around — people keep a private spreadsheet and enter it all at the end, and you
are back to retrospective records with extra steps. Three gates that matter beat eleven that do not.

**Timestamps must be system-generated.** Never let a user type the date a thing happened. The value of an
enforceable record is that its chronology is trustworthy; a hand-typed date destroys exactly that. Capture
created-on, changed-on and who did it from the platform, and treat them as read-only.

## 5 · How to tell whether your gates are actually working

Two checks, both quick, both worth running on your own system before you trust it.

**The distribution check.** Pull the created-on timestamps for every record and plot them by day. Real
in-process use spreads across the working calendar. If most of your records were created on a handful of
days — or worse, if a hundred appeared in one afternoon — then the work happened somewhere else and the
system is being used as a filing cabinet after the fact. That is the exact failure the pattern exists to
prevent, and it is invisible unless you look.

**The dwell check.** For each record, measure the elapsed time between created and last-modified. Records
that are created and finalised within seconds were not filled in while the work was being done. A healthy
system shows records that stay open for as long as the job takes.

Both checks are a single query and neither is flattering. Run them anyway. A quality system nobody audits is
just a database with opinions.
