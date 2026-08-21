# Troubleshooting and FAQ

Two halves. The failures you will hit while building this, and the questions a sceptical quality manager or
an auditor will ask — answered honestly rather than favourably.

Route A means SharePoint lists + Power Apps + Power Automate; Route B means Dataverse. Read "Choosing your
platform" in `docs/03-implementation-guide.md` first — several answers turn on which one you are running.

---

## 1 · Troubleshooting

### The flow triggers itself in a loop

**Symptom.** One record edit produces dozens or hundreds of flow runs, seconds apart. Everything slows down
and the platform eventually starts refusing requests.

**Cause.** The rule that reverts an invalid stage change writes to the record that triggered it. That write
satisfies the trigger, which runs the rule, which writes again — Microsoft's documented infinite trigger loop
anti-pattern, which `docs/03-implementation-guide.md` warns about. Power Automate flags it when you save the
flow, and the warning is correct.

**Fix.** Turn the flow off and let the queued runs drain, then use both mechanisms. Put a **trigger
condition** on it so it fires only on the transitions you care about — typically when `Stage` has changed and
the modifying identity is not the service account your rules run under. Add a **Terminate** action that stops
the run when the flow detects it is reprocessing its own write. Do it in a sandbox first.

### Someone set `Stage` directly and the gate never ran

**Symptom.** A record sits in `Closed` with no result, no inspector and no timestamp. Nothing refused it.

**Cause.** On **Route A** this is the platform, not a bug. SharePoint's smallest unit of permission is the
item, not the column, so there is no supported way to make `Stage` read-only to someone who can still edit
records. Anyone in grid view, an import or another client can write it. Keeping the column off every form
stops the honest user and nobody else.

**Fix.** On **Route B**, enable column security on `Stage` and grant Update through a Column Security Profile
only to the service identity your rules run under. That closes it properly.

On **Route A** you cannot stop the write by permission — but you can make the resulting item illegal, which
is usually enough. Write the gate as a list validation formula stating what must be true of a record *at*
each stage rather than what must happen during a transition, and the server refuses the item whichever client
submitted it, grid included. `docs/09-microsoft-lists-build.md` §3 has the formula shape. What remains
genuinely open on Route A is altering a field after its stage has passed, so write that one down rather than
explaining it away, and lock the record at closure.

### Timestamps are wrong, or in the wrong time zone

**Symptom.** `Inspected_On` shows a time hours ahead of the inspection, or a record appears to have been
inspected tomorrow.

**Cause.** Almost always this is correct data displayed badly. The platform stores date/time values in UTC
and renders them in the viewer's time zone — reliably so on SharePoint, and on Dataverse depending on the
column's date/time behaviour, which is fixed when the column is created and awkward to change afterwards.
Where the conversion applies, a record stamped at 14:00 local shows as 18:00 or 19:00 in an
export. That looks wrong and is not.

**Fix.** Set the time zone in your app, view and report layer rather than converting the stored value, and
label exported columns as UTC so nobody re-converts them. Do not respond by adding a user-typed local date. A
system-set UTC value rendered in local time records when something happened; a user-typed local date records
what someone remembered or preferred, which defeats the pattern. `docs/01-data-model.md` is explicit that
`Due` is the only date a user types, because it is a plan rather than evidence. If a rule really is stamping
a converted value, fix the rule and raise the correction as a new record rather than editing the old ones.

### Two people share a display name, so the self-approval check can be defeated

**Symptom.** Gate 3 passes when approver and raiser are the same human, or refuses when they are two
different people sharing a name.

**Cause.** The comparison reads display name rather than directory identity, and display names are not
unique: two employees called Person A compare equal.

**Fix.** Compare the directory identifier the platform resolves behind the person column, not the text it
renders. If the comparison reads a string a user could type, you have a text column where
`docs/01-data-model.md` calls for a person column. Fix the column type first, then the comparison.

### The gate refuses with an unhelpful message and people stop trusting it

**Symptom.** "Validation failed." Inspectors start calling the system broken, and records entered per day
quietly fall.

**Cause.** A refusal that does not name what is missing is indistinguishable from a fault, and people work
around a gate they find arbitrary far sooner than one they understand.

**Fix.** Every refusal names the missing evidence in the inspector's vocabulary — *"Cannot mark inspected —
record the quantity checked, the result, and who inspected it"* rather than a rule identifier. The messages
in `examples/incoming-inspection-checklist.md` are written to be copied. Build the courtesy layer too: grey
out the advance control and show what is still needed, so a refusal is rarely a surprise.

### Records advance but the locked fields are still editable

**Symptom.** A record reaches `Inspected` and the inspector can still change `Result`.

**Cause.** Either build step 4 was not really completed, or you are on Route A, where it cannot be.

**Fix.** On **Route B**, resist the obvious answer. A Column Security Profile cannot do this: it applies to
every row in the table, so revoking Update to lock a passed stage locks the same column on records still in
`Draft`, and it cannot secure a person or lookup column at all. Implement it as a server-side rule that
rejects any update to a gated column when `Stage` is past the gate that required it — conditional, per
record, and it covers the person columns. See `docs/03-implementation-guide.md` step 4. On **Route A** you
can only drop the fields from the form once the stage advances,
which an editor in grid view goes around. Editable history is the first of the three mistakes in
`docs/02-hard-gate-pattern.md` §3, and on Route A you carry it knowingly — write that down.

### Everything slows down once volume rises

**Symptom.** Flows queue, runs are delayed by minutes, and lists are slow to open.

**Cause.** Usually a flow retriggering more than it needs to, an unindexed view over a grown list, or tenant
request limits being approached.

**Fix.** Narrow the trigger conditions. Index the columns you filter and sort on, particularly `Stage` and
`Reference`. Return fewer rows per view — an inspector needs today's open receipts, not four years of closed
ones — and archive closed records on a schedule. If limits still bite, you have outgrown the pilot.

### Someone deleted a record instead of closing it

**Symptom.** A reference number people remember has no record behind it, or a sequence has a gap.

**Cause.** Delete permission was left with ordinary users, the default when the list was created.

**Fix.** Try to recover it, but check which route you are on before you assume you can. SharePoint has a
two-stage recycle bin that retains deleted items for a limited period. Dataverse does **not** keep deleted
rows by default — an administrator has to have enabled record retention beforehand, only records deleted
after it was switched on can be restored, the retention window is capped at **30 days**, and several table
types are excluded from it entirely. Confirm which of those is true for your environment now rather than at
the moment you need it. Where recovery is possible, act
immediately. Then remove delete permission from everyone who does not need it, and give people a supported
way to void a record — a `Cancelled` value on `Stage`, set through a rule, with a required reason. People
delete records because there is no legitimate way to say one was raised in error, and a system with no such
path gets corrected by deletion.

Note the boundary honestly. No records system is beyond alteration by a sufficiently privileged
administrator; someone with tenant-level rights can remove a record and its history on either route. What you
build constrains ordinary users and leaves a trail for everyone else. That is a real control, not an absolute
one, and should never be described as one.

---

## 2 · FAQ

**Can I do this in Excel?**
You can model it. You cannot enforce it. A spreadsheet can lay out the columns and refuse an entry with data
validation, but gives you no rule that runs on the data regardless of who wrote it, and no history ordinary
users cannot edit. Anyone can unprotect a sheet, retype a date, or email a copy that becomes the new master.
If that describes where you are today, run the two checks in `docs/02-hard-gate-pattern.md` §5 against it
first.

**Is this validated software?**
No. It is a documented pattern in markdown files. Nothing here has been validated for any purpose and no
validation evidence comes with it. If your sector requires computerised-system validation, that obligation
falls on what you build. See `DISCLAIMER.md` in the repository root.

**What do I tell an auditor this system does?**
Roughly: *"Inspection and non-conformance records are created while the work is happening, cannot advance
without the evidence each stage requires, and carry system-generated timestamps and resolved user
identities."* Then give the written list from build step 7 of which cheats your build blocks. On **Route A**
two claims are not supported: that the stage column is protected, and that earlier stages lock. Do not make
them.

**Do I need Dataverse?**
Probably not, and the question is narrower than it looks. Route A enforces the gate conditions themselves:
list validation is evaluated by the server on save, so a record cannot reach a stage without the evidence
that stage requires, whichever client wrote it. That is a defensible thing to show an auditor, provided you
describe it accurately and can produce the test results from `docs/06-validation-and-test-plan.md`.

What Route A cannot do is stop a field being altered *between* stages, because there are no column-level
permissions on that tier. So the question is: does anyone need to prove a record was not changed mid-process,
as opposed to proving it was complete at each stage? If yes, that is a premium licence — Dataverse on the
Microsoft stack, or any platform with server-side rules and field-level permissions. If no, there is nothing
to buy.

**How is this different from our ERP's NCR module?**
Often it is not, and that is the useful answer. Check one thing: does the module enforce on the data, or only
hide buttons on a screen? If it genuinely refuses a status change without the required evidence, whatever
client attempted it, use it and ignore this kit. Use this pattern where there is no such module, where it is
priced beyond what you can justify, or where it is a form with a status field.

**Will this make us ISO 9001 certified?**
No. It helps you evidence a subset of clauses — 7.5, 8.4.2, 8.5, 8.6, 8.7 and 10.2, mostly. A certificate
also depends on context, leadership, planning, competence, calibration, internal audit and management review,
none of which this touches; `docs/04-compliance-mapping.md` sets out both sides of that boundary. The same
goes for CMMC — this is a quality-records pattern, not a security control set.

**How many gates should we have?**
Few and load-bearing. Three that matter beat eleven that do not. Friction that protects nothing gets routed
around, which costs more than that gate — it teaches people that gates in general are worked around. Add one
only where skipping the step would cause real harm.

**Can we back-date a record we forgot to enter?**
No, and this is the question that most tests whether the point has landed. The value of an enforceable record
is that its chronology is trustworthy. A record entered on Thursday for work done on Monday, stamped Monday,
is a false statement inside an evidence system, and one of them makes every other timestamp arguable.

Instead: create the record now, with today's system timestamp, and state in the notes what happened, when,
and that it was entered late. That is a truthful late record and it is defensible; an overwritten timestamp
is neither. The same holds for corrections — a genuine correction is a new versioned record carrying a
reason, never an edit over the top of the old one.

If you need this often, the finding is not about the software. Retrospective entry is what the distribution
and dwell checks detect, and the failure the pattern exists to prevent.

**Who supports this?**
Nobody. No company, no support desk, no service level, no roadmap with dates on it. It is a document set
released by one person in a personal capacity, in evenings, under no obligation to anyone. Issues and pull
requests are welcome and may be answered slowly or not at all. For free help on your obligations, `README.md`
points to APEX Accelerators and MEP centres, both publicly funded.

**Can I use this commercially, or inside a product I sell?**
Yes. MIT licence — see `LICENSE`. Use it commercially, modify it, ship it inside your own tooling, no
attribution beyond the licence text. No warranty; validating what you build is your responsibility.

---

## What this document does not do

It does not cover platform administration, tenant licensing, or errors specific to your configuration. The
failures above are inherent to the pattern; the rest belong to your vendor's documentation.

It cannot tell you whether your build is correct. Only build step 7 in `docs/03-implementation-guide.md` can,
by attempting each cheat and recording which succeeded.

And it is not advice on your obligations. Nothing here interprets a standard, a contract clause or a
flow-down requirement. Confirm those with your certification body, contracting officer or an APEX counsellor
rather than any third-party summary, this one included.
