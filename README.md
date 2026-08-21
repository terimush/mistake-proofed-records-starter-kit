# Hard-Gate Starter Kit

**A free, low-code pattern for enforceable quality records in small manufacturing.**

Build inspection and non-conformance records that *cannot* be completed out of order, left half-filled, or
back-dated — using software your business most likely already licenses. No new platform, no consultant, no
per-seat quality-system subscription.

---

## The problem this addresses

A small manufacturer supplying a regulated customer carries substantially the same documentation and
traceability obligations as a large one. ISO 9001 wants evidence that inspections happened, that
non-conformances were investigated to root cause, and that records are controlled. Defence customers add
requirements of their own on top.

Large firms answer this with dedicated quality-management platforms. Those platforms are priced for large
firms. The result is that a great many small shops run their quality system on paper, spreadsheets, and
memory — and then rebuild the evidence retrospectively when an audit is scheduled.

Retrospective records are the failure mode. They are hard to defend, they take days to assemble, and they
mean nobody was actually being guided by the process while the work was happening.

## The idea

A **hard gate** is a rule the software enforces rather than a policy people are asked to remember:

> A record cannot advance to the next stage until the evidence that stage requires actually exists.

No inspection result, no advance. No root-cause entry, no closure. The record *is* the process, so following
the process and producing the evidence are the same action rather than two competing ones.

The pattern is deliberately small. It is three tables, one status column, and a set of conditions on the
transitions between statuses. It runs on general-purpose low-code business platforms — the worked example
here targets the Microsoft Power Platform, but the pattern is not specific to that vendor and the
documentation is written so it can be rebuilt anywhere with a forms layer and a rules engine.

**One thing to know before you start.** You can build this on **Microsoft Lists**, on the subscription you
almost certainly already have, and the gates will be genuinely enforced — Lists evaluates validation on the
server when the item is saved, so a correctly written rule holds whichever client wrote the record, including
someone typing into the grid. That is the central claim of this kit, on a standard licence.
`docs/09-microsoft-lists-build.md` has the formulas.

What Lists cannot do is lock one column against one user, so a field can be altered between stages until the
record is locked at closure. Dataverse fixes that and costs a premium licence.
`docs/03-implementation-guide.md` names the single symptom that means you have outgrown Lists; until you hit
it, the free route is the right route.

## What is in this kit

**Start with `START-HERE.md`.** It tells you what to read, in what order, and how far in to go before
deciding whether any of this is worth your time.

### The pattern

| File | What it is |
|---|---|
| `docs/01-data-model.md` | Three tables, their columns, and how they relate. Recreate in about twenty minutes. |
| `docs/02-hard-gate-pattern.md` | The actual pattern: where gates live, how to enforce them so they cannot be bypassed, and the three mistakes that make a gate decorative. |
| `docs/03-implementation-guide.md` | Platform choice and its consequences, step-by-step build, licensing notes, and an honest time estimate. |
| `docs/04-compliance-mapping.md` | Which ISO 9001:2015 clauses and which CMMC / NIST SP 800-171 practices this helps you evidence — **and which it does not**. |

### Getting it into use, and proving it works

| File | What it is |
|---|---|
| `docs/05-rollout-runbook.md` | Getting from a built pilot to how the shop actually works: the first process, the parallel run and its time limit, and what to do when a gate gets routed around. |
| `docs/06-validation-and-test-plan.md` | A test suite for finding out whether what you built does what you think it does, with the expected result stated separately for each platform. |
| `docs/07-gate-self-test.md` | A printable thirty-minute check on a system you already run, even if it did not come from this kit. |
| `docs/08-troubleshooting-and-faq.md` | The failures people hit while building this, and the questions a sceptical quality manager or auditor actually asks. |
| `docs/09-microsoft-lists-build.md` | **The build, on the licence you already have.** What Lists genuinely enforces, the validation formulas gate by gate, the closure lock, and the ceiling in one table. |
| `docs/10-canvas-app-gates.md` | Why a gate in the app is the weakest of the three, which gate belongs in which layer, what the app layer genuinely adds that the list cannot, the delegation limit that makes a counting gate silently wrong past 500 rows, and a ranked set of workarounds — separated from the popular ones that only hide the warning. |

### Worked examples

The three build examples each state their gates as explicit conditions, give the refusal messages, walk a
failing record through every stage, and say what changes between Lists and Dataverse.
The dashboard is the exception: it has no gates and enforces nothing, by design.

| File | What it is |
|---|---|
| `examples/incoming-inspection-checklist.md` | Incoming material inspection. The easiest first build. |
| `examples/nonconformance-intake.md` | A non-conformance record that will not close without causal analysis somebody actually did. **If you build only one thing, build this.** |
| `examples/production-hard-gate-checklist.md` | In-process production checks, with the condition that makes retrospective entry visibly impossible. The hardest of the three, because it touches people paid to make parts rather than to fill in records. |
| `examples/compliance-status-dashboard.md` | Six read-only panels. The only artefact here that enforces nothing — half of it exists to show you how your own records might be worthless. |
| `examples/*.csv` | Fabricated sample records for every table, including records left mid-stage so you can watch the gates refuse. |

### Teaching it

| File | What it is |
|---|---|
| `training/workshop-outline.md` | A half-day hands-on workshop, with the timed plan and facilitator notes. |
| `training/practitioner-talk.md` | Speaker notes for a 45-minute practitioner talk, including the questions you will be asked and the honest answers. |
| `training/one-page-handout.md` | One printable page. The thing left on a chair. |

Everything here is free to reuse, including for teaching it yourself. See `CONTRIBUTING.md` if you want to
send something back, and `DISCLAIMER.md` before you rely on any of it.

## Who this is for

Quality managers, engineers and owners at small and mid-sized manufacturers — particularly those in a
regulated supply chain who have been told they need better records and have no budget for a platform.

It assumes you understand your own process. It does not assume you can write code.

## What this is not

It is not a quality management system, and it is not certification. It will not make you ISO 9001 certified
or CMMC compliant. It is a pattern for building records that hold up, which is one part of a much larger
picture that also includes leadership, competence, internal audit and — for CMMC — a set of security
controls this kit does not touch. Read `docs/04-compliance-mapping.md` before assuming otherwise; it is
explicit about the boundary.

There is no warranty. You are responsible for validating anything you build for your own use. `DISCLAIMER.md`
says this properly and is worth the two minutes.

**One more boundary, stated here because it is the one people most want to blur.** On Lists there are no
column-level permissions. Gate conditions are still enforced — the server refuses an item that does not
satisfy its stage — but a field its earlier stage required can be altered afterwards, until the record is
locked at closure. A rule that needs to read a second list becomes a flag a flow maintains on the record
itself, which validation then refuses on: the refusal is real, but what it refuses on is the flow's answer,
so the flow's log is part of your evidence. And each list holds one formula and one message, so per-gate
refusal wording lives in the app rather than in the list. All of those are platform facts rather than build
errors, and `docs/09-microsoft-lists-build.md` states the full ceiling in one table. Run
`docs/06-validation-and-test-plan.md` and describe to an auditor what your build actually blocks, not what
the pattern is capable of.

## Getting help

Free, publicly funded assistance exists for exactly this. **APEX Accelerators** (formerly PTACs) are funded
by the U.S. Department of Defense and provide free counselling to small businesses on government contracting,
including compliance obligations; there is a centre serving every state. **Manufacturing Extension
Partnership (MEP)** centres offer similar help on the manufacturing side. If you are unsure what your
contracts actually require of you, those are better first calls than any vendor — including this document.

Requirements and timelines change. Confirm your current obligations with your contracting officer or an
APEX counsellor rather than relying on any third-party summary, this one included.

## Licence and contributions

MIT — see `LICENSE`. Use it commercially, modify it, ship it inside your own tools, no attribution required
beyond the licence text. If you build something better, or find something here that is wrong, an issue or a
pull request is welcome.

## About

Written and released by **Tererai Mushangwe**, a quality engineering practitioner in Ohio, as a contribution
to the small-manufacturer community. Released in a personal capacity. Nothing here is derived from, endorsed
by, or associated with any employer, and no proprietary material of any organisation is included.
