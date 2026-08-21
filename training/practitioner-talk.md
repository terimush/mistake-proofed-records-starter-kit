# Section talk — speaker notes

A 45-minute practitioner talk with 15 minutes of questions, for a professional-society section meeting. The
audience is quality practitioners: quality managers, quality engineers, auditors, a few consultants, and
usually two or three people whose job title says something else entirely.

These are running-order notes carrying the argument, not a slide dump. Build whatever slides you like around
them. The talk works with six slides and it works with none.

**Two things this talk must not become.** It must not become a vendor pitch — no product recommendation, no
partner link, no route to a sale. And it must not become a personal advertisement. The strongest version of
this talk is the one that admits, in front of the room, what the pattern cannot do. An audience of quality
practitioners can smell a sales deck from the second slide, and once they have, nothing after it lands.

---

## Running order

### 0:00 – 0:07 · The problem

Open with the asymmetry, because everyone in the room has lived it.

A small manufacturer supplying a regulated customer carries substantially the same documentation and
traceability obligations as a large one. The standard does not scale its requirements to headcount. Defence
customers add their own on top. Large firms answer this with dedicated quality-management platforms priced
for large firms, and the small shop answers it with paper, spreadsheets and memory.

Then name the actual failure mode, which is not what people expect. It is not missing records. Most shops can
produce records. It is **retrospective records** — assembled in the fortnight before an audit, reconstructed
from memory and dispatch notes, technically present and impossible to defend. They take days to build, they
are hard to stand behind, and their real cost is that nobody was being guided by the process while the work
was happening.

Ask the room: who here has rebuilt records for an audit? Wait for the hands. They will go up, and the talk
now belongs to the audience rather than to you.

### 0:07 – 0:12 · The idea

One sentence, and put it on a slide on its own:

> A record cannot advance to the next stage until the evidence that stage requires actually exists.

No inspection result, no advance. No root-cause entry, no closure. The record *is* the process, so following
the process and producing the evidence stop being two competing activities and become one action.

Then make it concrete fast: three tables, one status column, and a set of conditions on the transitions
between statuses. Three gates. Draft to inspected, inspected to dispositioned, dispositioned to closed. The
whole pattern fits on a napkin, and that is deliberate — the version with eleven gates is the version that
gets routed around.

### 0:12 – 0:22 · Why most implementations are decorative

This is the heart of the talk. It is the section that makes practitioners reconsider a system they already
run, and it is the reason to give the talk at all.

There are three places you can enforce a gate, and they are not equivalent.

**Hide or disable the button.** The advance control is greyed out until the fields are filled. This is the
version most people build. It is a suggestion. Anyone who can open the underlying table can set the status
directly and the gate never runs.

**Validate on save.** The form refuses to commit a record that does not satisfy its gate. Better, and still
tied to the screen. A second form, an import, a mobile client or an integration writes to the same table and
the gate is simply not there.

**Enforce on the data, at the transition.** The rule lives with the record. Any path attempting an invalid
stage change is rejected, whichever client attempted it, and the status column is not directly editable by
ordinary users at all — it changes only as a *result* of a gate passing.

Land the point: **the difference between these three is invisible from the form.** All three look identical to
the person using the system, and to the person demonstrating it in a management review. It becomes visible in
exactly one place, which is an audit, at the worst possible moment.

### 0:22 – 0:29 · The three mistakes

**Editable history.** If a record can be changed after it closes, it is a form and not evidence. Lock the
fields a stage required once that stage passes. Later stages stay open; earlier ones do not. A genuine
correction is a new versioned record with a reason, not an overwrite.

**Self-approval.** A gate requiring approval that the same person can satisfy is not a control. Compare
approver against raiser and reject a match. One condition. It does more for audit credibility than any
amount of workflow decoration, and it is the single change most rooms go home and make.

**Optional depth.** "Root cause" as one free-text box gets filled with the symptom restated. If you want
causal analysis, require structured depth — three linked levels, each referencing the one above. It is mildly
annoying to complete, and the annoyance is the analysis.

### 0:29 – 0:38 · The demonstration

Live if the wifi is trustworthy, screenshots if it is not. Rehearse either version; a demonstration that
stalls costs more than it earns.

1. Advance an empty record. Refused, with a message naming what is missing.
2. Fill it properly and advance. Point at the timestamp: set by the platform, not typed by anyone.
3. Close with one causal level. Refused.
4. Approve your own corrective action. Refused, because the approver matches the raiser.
5. **Then try to break it.** Open the underlying list directly, outside the form, and set the stage to
   closed on a half-empty record. If you built the gate in the form, it works and the room sees the whole
   problem. If you built it as list validation, the server refuses it — and that is the more useful
   demonstration, because it shows the fix rather than only the failure.

Step 5 is the point of the whole demonstration. Do not skip it to protect the material. A room that has
watched somebody go around a form, and then watched the same attempt bounce off the list itself, understands
where a rule has to live in a way no slide achieves.

### 0:38 – 0:43 · The honest limitations

Say all of this out loud. It is the part that earns the room.

The pattern runs on general-purpose low-code platforms. Say both halves of what the included tier does,
because rooms have heard the pessimistic half from vendors and it is wrong.

**What the included tier does do.** On SharePoint lists in a standard Microsoft 365 subscription, a gate
condition written as a *list validation formula* is evaluated by the server when the item is saved. Not in
the form — on the item. So it holds whichever client wrote the record, including somebody typing into the
grid. That is the demonstration you just gave in step 5, and it is the reason this talk is worth giving.

**What it does not do.** There are no column-level permissions on that tier. So a field its earlier stage
required can still be altered afterwards, until the record is locked at closure — and locking at closure is
available, through a flow that stops sharing the item. Conditions that have to read a second list are
checked and reverted rather than refused. Those are the limits, and they are narrower than "the gates are
advisory".

**Where the paid step comes in.** Dataverse and a premium licence buy per-column, per-record,
stage-conditional locking — the ability to prove a record *was not altered between stages*, rather than that
it was complete at each stage. Say the licensing step exists and say exactly what it buys. A talk that omits
it is selling something; a talk that overstates it is selling something else.

And the boundary: this is not a quality management system, it is not validated software, and it is not
certification. It is a records pattern. Certification depends on leadership, competence, internal audit,
management review and a great deal else that this does not touch.

### 0:43 – 0:45 · Where to get it

MIT licensed, free, no sign-up, nothing to buy and nobody to call. It is at `https://github.com/terimush/hard-gate-starter-kit`.

If your contracts carry obligations you are unsure about, **APEX Accelerators** — U.S. Department of Defense
funded, formerly PTACs, with a centre serving every state — and **MEP centres** provide free counselling.
Better first calls than any vendor, and better than this talk.

Close on the two checks, because they are the thing anyone can do tomorrow without building anything: pull
your created-on timestamps and plot them by day, and measure the elapsed time between created and
last-modified on every record. Neither is flattering. Both are one query.

---

## Questions you will be asked, and the honest answers

**"Is this validated software?"** No. It is documentation. There is no code to install. Anything you run is
something you built, on a platform you licensed, and validating it for your use is your responsibility. Do
not soften this.

**"How is this different from our ERP?"** If your ERP already enforces the transitions and stamps the
timestamps, it is not different and you should use the ERP. This exists for shops whose ERP does not do it,
or who cannot afford the module that would. Then offer the two checks — an ERP module that nominally enforces
gates and shows a hundred records created in one afternoon is worth knowing about.

**"Can we do this in Excel?"** You can build the tables. You cannot enforce the transitions, because a
spreadsheet has no server-side rule that runs whatever client wrote the row, and its dates are typed rather
than stamped. A spreadsheet records what someone says happened. That is the whole gap.

**"Who supports it?"** Nobody. It is a personal project maintained in spare time by one person. There is no
company, no support desk, no service level and no roadmap. Issues may go unanswered for a long time. It is
MIT licensed precisely so that you never have to wait for me.

**"Does this make us compliant?"** No. It helps you evidence a specific set of clauses — control of external
providers, control of nonconforming outputs, corrective action, documented information — and the mapping
document is explicit about the much longer list it does not reach. Nothing built from this is evidence of
certification against anything.

**"What does it cost?"** The documents are free. The build costs the time — an evening if you have used a
low-code forms tool before, a weekend if you have not, plus an afternoon adapting the stages. Nothing else,
on the licence most shops already hold. Per-column locking costs a premium licence, which you should price
for your own tenant rather than take from me. Do not quote a figure from the stage; licensing changes, and a
wrong number said with confidence is worse than no number.
