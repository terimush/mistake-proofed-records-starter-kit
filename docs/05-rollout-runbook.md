# Rollout runbook

`docs/03-implementation-guide.md` tells you how to build the software. This tells you how to get from "I read
this" to "this is how we work now" — the part that usually fails.

It assumes you have already made the Route A / Route B choice in `docs/03-implementation-guide.md`. That
choice changes what you can honestly say during rollout, so the sections below say which route they assume.

---

## 1 · Choosing the first process and the first team

Pick one process. Not two, and not one you also intend to redesign at the same time. Incoming inspection is
usually the right first choice, for three reasons.

**Bounded.** A receipt arrives, someone checks it, it is accepted or it is not. Compare in-process
inspection, where the unit of the record is genuinely contested — per operation, per batch, per shift — and
you will argue about the data model for a fortnight before anyone enters anything.

**Frequent.** You need volume for the habit to form. A process that fires three times a month never becomes
automatic, and every entry is a fresh act of remembering. Incoming inspection usually fires daily.

**Low stakes if you get the stages wrong.** You will get them wrong on the first pass; everyone does. Here
the cost is a stage that annoys people for a week. On a final-release process it is a shipment held for a
reason nobody understands.

**Choose the first team by disposition, not seniority.** You want the inspectors who will tell you the gate
is wrong, not the ones who work around it politely and say nothing. One shift, two or three people. A rollout
to everyone at once removes your ability to learn anything from a complaint.

## 2 · The phases, and how long each actually takes

| Phase | Elapsed time | What done looks like | What makes you stop and go back |
|---|---|---|---|
| **Sandbox build** | About two working days for the first working gate set (8.78 h measured, stopwatch-timed, one build), plus an afternoon adapting stages | Three tables, three gates, and the build order in `docs/03-implementation-guide.md` — ten steps on Lists, against a test environment | Causal-depth or self-approval gates can be bypassed — a build fault, not the platform. Back to `docs/02-hard-gate-pattern.md` §2 |
| **Dry run on fabricated data** | Half a day | `examples/sample-data.csv` walked through every stage, every cheat in build step 7 attempted and logged | You cannot state in one sentence which cheats your build blocks |
| **Parallel run** | Two to four weeks, hard limit | Both methods running, a written comparison, one real non-conformance carried to closure | Either side is being filled in retrospectively to keep up. See §3 |
| **Cutover** | One day, announced a week ahead | The old method stops on a date everyone knows. Old records exported and retained, not deleted | Nobody can say who owns a gate refusal at 6am on a Saturday |
| **First month** | Four weeks from cutover | Distribution and dwell checks run, keep-or-stop decision made on the evidence | The checks show retrospective entry — a process or training problem, not a software one |

Do not shorten the sandbox phase to "I will just be careful on the live list". On either route, a rule that
reverts an invalid stage change can retrigger itself, exhaust your request limits and get suspended;
`docs/03-implementation-guide.md` covers the fixes.

## 3 · The parallel run, and its hard time limit

This is where most rollouts either earn their keep or quietly die. Treat it as an experiment with a stated
end date, not as a transition period.

**How long.** Two weeks minimum, four weeks maximum. Two because you need one full cycle including a real
non-conformance carried to closure — a fortnight of clean accepts tells you nothing about the gates that
matter. Four because of the trade-off below.

**What you are comparing.** Not "is the new one nicer". Four specific things:

- **Record count.** Did the same number of receipts reach both? A shortfall in the new system means people
  are skipping it, and you need to know why before you cut over.
- **Refusals.** Every time a gate refused, was the refusal correct? Keep a list. A correct but unwelcome
  refusal is a training item. A wrong one is a design fault you fix during the run rather than after.
- **Entry timing.** Run the distribution check from `docs/02-hard-gate-pattern.md` §5 against the parallel-run
  data. If the new records are already clustering into Friday afternoons, cutting over will not fix that.
- **Time per record.** Ask, do not guess. If the new method takes materially longer, either a gate demands
  evidence the stage does not need or the form is badly laid out.

**The honest trade-off.** A parallel run doubles the data-entry burden on the same people for no visible
benefit to them. That is unsustainable by design, and everybody involved knows it. People abandon one side,
and which side is not your choice: abandon the new one and you conclude wrongly that it failed; abandon the
old one and you have cut over by accident, without a decision.

So set the end date before you start, say it out loud, and hold it. Beyond four weeks you are not running an
experiment, you are taxing your inspectors.

## 4 · Who needs to be told what

Four audiences, four different messages. One email to all of them is the commonest rollout mistake after
skipping the sandbox.

**The inspectors doing the entry.** What changes about their day, that a refusal is the system working
rather than the system being broken, and exactly who to tell when a gate is wrong. Do not sell them a quality
philosophy. Tell them plainly: this is how you stop being asked, six months from now, for records of a
receipt you cannot remember.

**The supervisor whose numbers change.** Their non-conformance count will go up, because issues once handled
verbally now leave a record — not because quality got worse. Say so before the first monthly report. A
supervisor ambushed by their own numbers becomes the rollout's most effective opponent, reasonably.

**Whoever will be asked for records in an audit.** Usually the quality manager. They need the sentence from
build step 7 in `docs/03-implementation-guide.md` — the written list of which cheats the build actually
blocks. On Route A that sentence says the *write* to `Stage` cannot be prevented, but the resulting item is
refused by list validation, so the record does not land in a stage it has no evidence for; what remains open
is altering a field after its stage has passed, until closure. In writing before an auditor asks, not during.

**The manager who might be asked for a Route B licence.** State the difference as a consequence, not a
feature, and do not ask for money you do not need. Route A is included in what you already pay for and
enforces the gate conditions themselves; what it cannot do is stop a field being altered between stages
before closure. The question is therefore narrow: does anyone need to prove a record *was not changed*
mid-process, as opposed to proving it was complete at each stage? If not, there is nothing to buy.

## 5 · Training the people who will use it

Twenty minutes, at the machine, on a real receipt. Not a classroom, not a slide deck, not a procedure people
sign and never read. Cover three things and stop.

**What the stages mean in their words.** Not `Draft` → `Inspected` → `Dispositioned` → `Closed` as labels, but
"you have started it", "you have checked it", "you have decided what happens to it", "it is finished".

**What each gate will ask for.** Walk one accept and one reject. The reject matters more: that is where the
gates bite and where people otherwise improvise.

**What to do when it refuses and they think it is wrong.** Name a person. Give them a way to record the
receipt anyway — on paper, deliberately, flagged — rather than leaving them stuck with material on the dock.
A gate with no relief valve gets routed around within a week and you will not hear about it.

Train in pairs and let the second person do the entry. Someone who has completed one record unaided will use
the system. Someone who watched a demonstration will not.

## 6 · When a gate gets routed around

It will happen. Treat the first instance as information rather than as a discipline problem: the first
question is not who did it, but whether the gate deserved to survive. Three questions, in order.

**Was the evidence the gate demanded actually available at that point?** If a gate asks for a result before
the measurement equipment is free, the gate is in the wrong place. Move it. This is much the commonest cause,
and it is your fault rather than theirs.

**Does skipping the step cause real harm?** If not, delete the gate. `docs/02-hard-gate-pattern.md` §4 is
blunt — gates should be few and load-bearing, and friction that protects nothing gets routed around and takes
the credible gates down with it.

**Is the process itself undefined?** Sometimes the gate is right and the honest answer is that nobody ever
decided who dispositions a borderline receipt. Software cannot decide that for you.

**The specific danger sign.** Watch for a private spreadsheet — one inspector's own file, kept at the bench,
entered into the system in a block on Friday. That is the retrospective-records failure the pattern exists to
prevent, rebuilt inside the system meant to prevent it. It shows up as a cluster in the distribution check
and as records created and closed seconds apart in the dwell check.

When you find one, do not confiscate it. Ask what it does that the system does not. Usually it lets them
record something before they have all the evidence a gate demands, which means you need an earlier stage
rather than a stricter rule.

## 7 · Deciding whether to keep it, after the first month

Run the two checks in `docs/02-hard-gate-pattern.md` §5 against your first month of real data. They are the
evidence for this decision. Opinions collected in a meeting are not.

**The distribution check.** Plot created-on timestamps by day. In-process use spreads across the working
calendar. Clusters mean the work happened somewhere else.

**The dwell check.** Elapsed time between created and last-modified. Records created and finalised within
seconds were not filled in while the work was being done.

Then one qualitative check. Ask an inspector, not a manager, whether a gate has ever stopped them doing
something wrong. A concrete example is worth more than either query. No example after a month of real volume
means the gates sit where nothing was going wrong anyway, and you can remove one.

If the checks are healthy, keep it and consider a second process. If not, the problem is the process design
or the training, and adding gates makes it worse. Fix or stop; do not gate harder.

## 8 · Rollback — stopping without losing the records

Deciding to stop is a legitimate outcome. Doing it badly is not — the records you created are real quality
records, and may be the only evidence an inspection happened at all.

**Export before you switch anything off.** Every table, to CSV, including the system-generated created-on,
modified-on and modified-by columns. Those columns are why the records were worth anything; an export that
drops them leaves typed data of no evidentiary value.

**Keep the exports under your existing retention rule.** Whatever your quality system says about retaining
inspection records applies to these, unchanged. A pilot is not exempt.

**Set the tables to read-only rather than deleting them.** On Route B, remove Update through the column
security profiles. On Route A there is no column-level permission, so set the list permissions to read-only
for everyone and record who retains Full Control. Deleting is the one thing you must not do — see
`docs/08-troubleshooting-and-faq.md` on records deleted instead of closed.

**Write one paragraph saying what happened.** Dates in use, process covered, why you stopped, where the
exports live. If an auditor later finds a two-month island of structured records with nothing either side,
that paragraph answers the obvious question.

**Tell the people using it what replaces it.** A rollout that stops without an announcement leaves half the
shop entering records into a system nobody reads.

## 9 · What this runbook cannot do for you

**It cannot create management commitment.** If nobody above the quality function cares whether inspection
records exist, gates get routed around and the routing is tolerated. That is an organisational problem and a
rollout plan does not touch it.

**It cannot fix an unmanaged quality system.** If your process is genuinely undefined — nobody knows who
dispositions a borderline receipt, or what the acceptance criteria are — enforcing stages surfaces that
loudly, and you answer it with people rather than software. Useful, but not the same as solved.

**It cannot substitute for competence.** A gate checks that a field is filled, not that the inspection was
done correctly or the causal analysis is any good. Three levels of "why" entered badly still pass Gate 3.
Competence and calibration sit outside this — see `docs/04-compliance-mapping.md` on ISO 9001:2015 clauses
7.1.5 and 7.2, which this pattern does not address.

**It cannot give you support.** There is none. This kit is a document set released by one person in a
personal capacity — no company, no response time, no obligation. What you build, you maintain.

**And it is not certification.** Running this rollout well produces records you can defend. It does not make
you ISO 9001 certified and it does not make you CMMC compliant. `docs/04-compliance-mapping.md` is explicit
about where the boundary sits.
