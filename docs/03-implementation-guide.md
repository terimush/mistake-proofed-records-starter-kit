# Implementation guide

Honest estimate: **two working days**, not the evening earlier editions promised. The kit has now been built
by hand from its own instructions, with a stopwatch: the first working gate set — three lists, three
formulas, four flows — took 8.78 hours against a 4.36-hour plan (stopwatch-timed, self-reported, one build).
The per-part figures are in `docs/09-microsoft-lists-build.md` §4, *"What Gate 1 actually cost to build"*;
plan from that table rather than from this sentence. One item from it belongs here because it changes how
you plan: **the three validation formulas are not data entry.** They were budgeted at 31 minutes as pasting
and took 211, about 40 % of the build, because a formula that saves is not thereby correct and has to be
proved clause by clause from the grid. Book them as design work. Budget a further afternoon to adapt the
stages to how your shop actually works, which matters more than the build.

**Start on Lists.** It is included in most Microsoft 365 business subscriptions and it enforces more than its
reputation suggests — `docs/09-microsoft-lists-build.md` is the complete build. Read "Choosing your platform"
below first anyway, because what you can honestly claim afterwards depends on where you put the rules, and
that difference is not visible from the form.

---

## Choosing your platform

The pattern needs three things: a table store, a forms layer, and somewhere to run a rule when a record
changes. Two common Microsoft routes exist. **Start on Route A.** It is included in the licence you already
have and it will take you further than its reputation suggests.

### Route A — Microsoft Lists + Power Apps + Power Automate · start here

Included in most Microsoft 365 business subscriptions. Check your own tier before assuming; entitlements
differ and change.

**What you get, and this is the part usually reported wrongly:** Lists evaluates **list validation formulas
on the server when the item is saved**. Not in the form — on the item. A gate written as a validation formula
therefore holds whichever client wrote the record: the form, the grid, an import, a flow, the API. That is
the central claim of this kit, on a standard subscription.

Two of the three conditions this kit argues do the most work — **enforced causal depth** and **separation of
duties** — are expressible as validation formulas and are genuinely enforced here, **provided each one is
written on the list that owns the columns it tests.** A formula sees one row of one list, so causal depth
belongs on `Nonconformance` and the approver comparison belongs on `CorrectiveAction`; written as a single
condition on the inspection they are unbuildable on any list, because the columns live in three places.
`docs/09-microsoft-lists-build.md` §3 gives the exact formulas, says which list each belongs on, and covers
the shadow-column technique the person columns require.

**What you do not get:** column-level permissions. The smallest unit of permission is the *item*, not the
column. Microsoft documents permissions for sites, lists, folders and items, and nothing below the item.

Four consequences, and they are narrower than they sound:

- **You cannot lock three fields while leaving others editable.** Locking is all-or-nothing per item. A flow
  running under a list-owner connection *can* make a whole item read-only at closure, which is a real
  per-record lock — with a documented ceiling of 5,000 recommended unique permission scopes per list, so plan
  the archive job.
- **You cannot express a rule that reads another list.** Validation formulas operate on their own row only.
  Cross-record conditions are checked by a flow that reverts an invalid transition — detection with automatic
  correction, which is a weaker claim than a refusal and should be described as one.
- **You cannot see a field's previous value**, so "frozen once its stage passed" is not available before
  closure.
- **A list gets one validation formula and one message**, and a formula cannot read a lookup, a person column
  or a multiple-lines-of-text column. So every gate on a list is one `AND(...)` with one shared refusal
  sentence, narrative fields need a companion column or a flow check, and person comparisons run on shadow
  text columns. None of it is fatal; all of it shapes the build.

**Verdict:** the right place to start, and further than a pilot. Build it, run the grid test in step 7, and
describe what it actually blocks.

### Route B — Dataverse + Power Apps + Power Automate

Requires Power Apps Premium or Power Automate Premium licensing. This is a **paid step up** and is not
included in a standard Microsoft 365 subscription.

**What you get:** server-side business rules and validation that run whatever client wrote the record, plus
column-level security through Column Security Profiles — you enable security on the column, then grant
Create / Read / Update to specific users or teams through a profile.

**Understand what column security actually is before you plan around it**, because it is routinely assumed to
do two things it does not do.

- **A profile is table-wide, not per record.** It binds a user or team to a column across every row.
  Microsoft's own description is "access to column data *for all records*". You cannot use a profile to lock
  `Result` on the records that have passed Gate 1 while leaving it writable on the ones still in `Draft` —
  revoking Update revokes it everywhere. Stage-conditional locking is a job for a server-side rule that
  rejects updates to gated columns when `Stage` is past the gate, or for per-record field sharing. It is not
  a job for a profile.
- **Several column types cannot be secured at all.** Lookup columns are among the exclusions, and a
  person/user column is a lookup. That means `Inspector`, `Owner`, `Raised_By`, `Approved_By`, `SignedOff_By`
  and `Checked_By` — the columns the separation-of-duties gates depend on — are not securable on any tier.
  They are protected by your rules or not at all.

Neither point weakens the pattern. Both change *where* the enforcement lives: on Route B it lives
predominantly in server-side rules, with column security as a useful additional lock on the plain-typed
columns it can cover.

**Verdict:** the upgrade, not the entry point. What it adds over Route A is per-column, per-record,
stage-conditional locking and rules that can read across tables without a revert.

### Deciding

**Start on Route A.** Not as a pilot to be thrown away — as the build. Most small manufacturers will never
need more, and a working gated record on a licence you already own beats a better one you have not bought.

**There is exactly one symptom that means you have outgrown it**, and it is worth stating precisely because
"we should probably upgrade" is otherwise a conversation with no end. Move when you need to prove a record
*was not altered between stages*, rather than that it was complete at each stage. That is a per-column,
per-record, stage-conditional lock, and no amount of cleverness produces it on Lists.

Everything else — cross-list conditions, the permission-scope ceiling, person columns in formulas — has a
documented workaround in `docs/09-microsoft-lists-build.md`. Reach for the licence when you hit the one
thing that has none.

Anything with a table store, a forms layer, server-side rules and field-level permissions will work. The
pattern is not Microsoft-specific; only this guide's worked examples are.

### One warning before you build the transition rules

The rule that reverts an invalid stage change writes to the same record that triggered it. That is
Microsoft's documented **infinite trigger loop** anti-pattern — a flow triggered by an update that then
performs an update "can potentially trigger itself forever." Power Automate will warn you when you save it.

Fix it deliberately rather than ignoring the warning: put a **trigger condition** on the flow so it only
fires on the transitions you care about, or add a **Terminate** action once the flow detects it is
reprocessing its own write. Do this before the flow ever touches real data — a loop against a live list is
unpleasant to unwind, and can exhaust your request limits and get the flow suspended. Never write a trigger
condition against a column key you have not read back from a real run's trigger output — a condition that
names a key the connector does not use resolves to nothing, and a guard built on nothing is true on every
save, including the flow's own (`docs/09-microsoft-lists-build.md` §7 records the two days that cost).

**Reading the key back is only half of it: you must also watch the condition fire.** A key that is correct
for an action is not evidence about the trigger, and the same wrong guard fails in two opposite directions —
loudly, by running on every save, or silently, by never running at all. Three of this kit's four flows
shipped a condition that was wrong, and every one of them showed a clean flow checker and a green Status
while doing it. Two traps account for most of it: **a column that is null on the item is absent from the
trigger payload entirely**, so a guard that waits for a column to be empty is asking about a key that is not
there; and **an expression that resolves inside an action can still resolve to null in a trigger condition.**
So: put the condition in, make one change that should qualify, and confirm exactly one run appears. If none
does, the condition is wrong, however reasonable it looks. Then the loop check — one edit, exactly one run,
then silence for five minutes — must pass before any F-flow is left switched on.

**Do this in a sandbox first.** Build it, run fake records through it, try to break your own gates. Only then
point it at real work.

## Build order

**1 · Create the three tables** exactly as in `01-data-model.md`. Get the column types right, particularly
the person columns — a text field where a person column belongs will quietly undermine two of the three gates.

**2 · Lock the `Stage` column.** Ordinary users must not be able to set it. This step is what separates an
enforceable record from a form with a status field — and **how far you can take it depends entirely on the
platform choice above.**

- *Route B (Dataverse):* enable column security on `Stage` and grant Update through a Column Security
  Profile only to the service identity your rules run under. This is the real thing.
- *Route A (Lists):* there is no column-level permission, so you cannot stop the write — but you can make
  the *result* of the write illegal, which gets you most of the way. Write the gate as a list validation
  formula stating what must be true of a record **at** each stage, rather than what must happen during a
  transition. A user who types `Closed` into `Stage` in grid view is then submitting an item that fails
  validation, and the server refuses it. `docs/09-microsoft-lists-build.md` §3 shows the formula shape.

  ⚠ **Leave `Stage` on the form on Route A.** It is tempting to hide it as well, and it is wrong here:
  nothing in the Lists build advances a record, so hiding the column leaves an honest inspector with no way
  to move one except the grid — the path you are about to use as the *attack* in step 7. On this route the
  user names the stage they are claiming and the server refuses the claim if the evidence is not there. The
  honesty comes from the refusal, not from concealment.

**3 · Build the rules.** What this step produces differs by route, and the difference is not cosmetic.

- *Route B (Dataverse):* one rule per gate. Each evaluates the gate condition and either advances `Stage`
  and stamps the timestamp, or refuses and returns a message naming what is missing. Refusing with a useful
  message matters — "Cannot advance: inspection result and quantity checked are required" saves a support
  call that "Validation failed" generates.
- *Route A (Lists):* **nothing advances the record.** The user sets `Stage`, and the list's single validation
  formula refuses any item that is not legal at the stage it claims. The flows on this route are not
  transition rules; they maintain the machinery columns the formula reads, and stamp the timestamps
  afterwards. One formula and one message per list — so the per-field message above is a Route B luxury, and
  on Route A it lives in the canvas app or nowhere. `docs/09-microsoft-lists-build.md` §3 and §4 are the
  build, and there are nine flows in it rather than five, because a SharePoint trigger binds to one list.

**Set the trigger condition now, not later** — see the infinite-loop warning above. A rule that writes to the
record that triggered it will retrigger itself unless you constrain it.

**4 · Lock earlier stages on passage.** Once Gate 1 passes, the fields Gate 1 required become read-only.
Same at each subsequent gate. Later stages remain editable.

Same platform caveat as step 2, with one correction people get wrong. On Dataverse, do **not** try to
implement this with a Column Security Profile — a profile applies to every row in the table, so revoking
Update to lock a passed stage locks the same column on records still in `Draft`. Implement it as a
server-side rule: reject any update to a gated column when `Stage` is past the gate that required it. That is
conditional, it is per record, and it works on the person and lookup columns a profile cannot secure at all.
Column security is still worth applying to `Stage` itself, which nobody should ever write directly.

On SharePoint you can only remove the fields from the form once the stage advances, which an editor can work
around.

**5 · Build the form.** Show the user which stage they are in, what the current gate needs, and grey out the
advance control until it is satisfiable. This is courtesy, not enforcement — the rules in step 3 are the
enforcement, and they must work even if someone bypasses this form entirely.

If the form is a canvas app, read `docs/10-canvas-app-gates.md` before you put any rule in it. Two things
there change the design: the arrangement that makes an app-level gate genuinely enforceable rather than
advisory, and the delegation limit that can make a counting gate silently wrong once a list passes 500 rows.

**6 · Add the self-approval check.** Compare `Approved_By` against `Raised_By` and reject a match. One
condition, disproportionate value.

Put it on the record where the approval is entered — `CorrectiveAction` — rather than on the inspection whose
closure it is meant to govern. `Approved_By` and `Owner` are both on that row already; have the flow copy the
non-conformance's `Raised_By_Email` down onto it and both comparisons become local. The refusal then stops
the invalid approval being recorded at all, which is earlier and stronger than stopping the closure that
follows it.

**Anchor it on a column the user sets, not on the presence of the shadow column.** A rule shaped "if there is
an approver's email, it must differ from the raiser" fails open, because the email column is written by a
flow *after* the save and is therefore blank at the moment the rule runs.
`docs/09-microsoft-lists-build.md` §3 formula 3 sets out the trap and the shape that avoids it. This is worth
five minutes of your attention: it is the difference between a control and a decoration, and it looks
identical from the form.

**7 · Load the sample data as reference state, then walk a record you create yourself.** The rows in
`examples/sample-data.csv` are already sitting at the stages they reached, which is useful for seeing what a
populated system looks like and useless for testing a transition. Create a fresh record and walk it through
every stage. Then deliberately try to cheat, **from the grid rather than the form**, because the grid is the
path that proves where your rule lives:

1. Set `Stage` to `Closed` on a half-empty record.
2. Change `Result` on a record that has already been inspected.
3. Submit a non-conformance with one causal level.
4. Approve your own action.

**On Route B all four should fail. On Route A, 1, 3 and 4 should also fail** — if your gates are written as
list validation formulas per `docs/09-microsoft-lists-build.md`, the server refuses the item whichever client
submitted it. If any of those three succeeds, your rule is in the form and your gate is decorative; go back
to `02-hard-gate-pattern.md` §2.

**Number 2 is the one that legitimately succeeds on Route A**, because validation cannot see a field's
previous value and there is no column-level permission. That is the platform, not your build, and it is why
the closure lock in step 8 matters. Finding it here rather than in an audit is the point of running the test.

Write down which of the four your build actually blocks. That sentence is the honest description of what you
have, and it is the one to give an auditor rather than "the system enforces it."

**8 · On Route A, add the closure lock.** At the final gate, have a flow running under a list-owner
connection stop sharing the item and grant read access back. That is the strongest control available on this
route, and the only one that makes a record genuinely fixed. Watch the unique-permission-scope count and set
up the archive job — `docs/09-microsoft-lists-build.md` §1 has the numbers.

## Adapting it to your process

The stages here are generic. Yours may need a customer-notification stage, a quarantine location, a
disposition-approval step for high-value scrap, or a link to your ERP receipt.

Two rules when you extend it. **Add a gate only where skipping the step would actually cause harm** — every
gate is friction, and unjustified friction gets routed around, which loses you the whole benefit. And **never
add a user-editable date for when something happened**; if you find yourself wanting one, what you actually
need is a new system-stamped transition.

## Running it for real

Start with one process and one team. Incoming inspection is the best first choice — the same answer
`START-HERE.md` and `README.md` give, for the same reason: bounded, frequent
enough to build habit, and low-stakes if you get the stages wrong on the first pass.

Tell people plainly why the gates exist. A gate presented as bureaucracy gets resented; the same gate
presented as "this is how you stop getting asked for records you don't have, six months later" usually
doesn't.

After a month, run the two checks in `02-hard-gate-pattern.md` §5 — the distribution of created dates, and
the dwell time between created and last-modified. They will tell you honestly whether the system is being
used while the work happens or filled in afterwards. If it is the latter, the problem is the process design
or the training, not the software, and no amount of additional gating will fix it.
