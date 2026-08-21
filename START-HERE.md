# Start here

You have been handed a folder of markdown files. This page tells you what to read, in what order, and how
far in to go before deciding whether any of it is worth your time.

**What this is, in two sentences.** A *hard gate* is a rule the software enforces rather than a policy people
are asked to remember: a record cannot advance to the next stage until the evidence that stage requires
actually exists. This kit documents how to build that for inspection and non-conformance records, on a
low-code platform your business may already license.

**What it is not.** Not a product, not a quality management system, not certification, and not a consultancy
pitch. MIT licensed. There is nothing to buy and nobody to call.

---

## Three ways in, by how much time you have

### 15 minutes — read one document

Read **`docs/02-hard-gate-pattern.md`** and nothing else.

That file carries the whole idea: what a gate is, the three places you can enforce one and why two of them
fail an audit, the three mistakes that make a gate decorative, and two checks you can run against a system
you already have.

If §2, on where to enforce a gate, does not describe a problem you recognise, the rest of the kit will not be
useful to you, and you can stop there with a clear conscience.

### 30 minutes — run the two checks against a system you already run

You do not need to build anything to get value from this.

Take any existing quality record system — a SharePoint list, an Access database, a spreadsheet with a
created-date column, your ERP's NCR module. Run the two checks in `docs/02-hard-gate-pattern.md` §5:

1. **The distribution check.** Pull created-on timestamps for every record and plot them by day. Real
   in-process use spreads across the working calendar. If most records were created on a handful of days, or
   a hundred appeared in one afternoon, the work happened somewhere else and the system is a filing cabinet.
2. **The dwell check.** For each record, measure elapsed time between created and last-modified. Records
   created and finalised within seconds were not filled in while the work was being done.

Both are a single query. Neither is flattering. They will tell you something true about a system you already
depend on, whether or not you ever look at the rest of this kit.

### Half a day — build the pilot

Read in this order:

1. **`docs/03-implementation-guide.md` → "Choosing your platform"** — read this *first*, before anything
   else in the build. Short version: build it on Microsoft Lists, which is included in the licence you
   already have and enforces more than its reputation suggests. Dataverse is an upgrade with exactly one
   trigger, and that guide names it.
2. **`docs/01-data-model.md`** — three tables, about twenty minutes to recreate.
3. **`docs/03-implementation-guide.md`** — the build steps, then
   **`docs/09-microsoft-lists-build.md`** for the actual Lists validation formulas. If your front end is a
   canvas app, read **`docs/10-canvas-app-gates.md`** before you put a single rule in it.
4. **`examples/incoming-inspection-checklist.md`** — a complete worked example to build against.
5. **`examples/sample-data.csv`** — fabricated records so you can watch it work before entering anything
   real.

Build it in a sandbox. Then try to break your own gates — step 7 tells you exactly how, and tells you which
failures are your build and which are the platform.

### After the pilot works

The build is the short part. Two documents matter more than the rest once something exists:

- **`docs/06-validation-and-test-plan.md`** — the tests that tell you whether what you built does what you
  think it does. Each states its expected result per platform, and one of them is expected to be *permitted*
  rather than blocked. Knowing which is which is the point.
- **`docs/05-rollout-runbook.md`** — getting from a working pilot to how the shop actually works, which is
  where most of these attempts die. Read it before you show anyone.

`docs/08-troubleshooting-and-faq.md` is worth skimming before you build rather than after — the trigger loop,
the person column that resolves to a non-unique display name, and the time-zone question are all cheap to
avoid and expensive to unwind.

---

## If you already have a system and just want to know whether it is real

Go straight to **`docs/07-gate-self-test.md`**. Thirty minutes, four things you attempt rather than inspect,
and it works against any system — an ERP module, a SharePoint list, something a contractor built for you
years ago. It does not require you to adopt anything here.

## If you want to build one of the other processes

Three more worked examples, each self-contained:

- **`examples/nonconformance-intake.md`** — the non-conformance as a record in its own right. If you build
  only one thing from this kit, build this one; it is where retrospective records do the most damage.
- **`examples/production-hard-gate-checklist.md`** — in-process production checks. The hardest, because it
  asks something of people whose job is the machine rather than the paperwork.
- **`examples/compliance-status-dashboard.md`** — six read-only panels over whatever you have built. Do not
  build this one until the self-test gives you an honest answer, because a dashboard makes whatever it
  displays feel true.

## If you want to teach it

`training/` holds a half-day workshop outline, speaker notes for a 45-minute talk, and a one-page handout.
MIT licensed, so you can deliver any of it as your own without asking.

---

## If you are checking it against a standard

**`docs/04-compliance-mapping.md`** maps the pattern to ISO 9001:2015 clauses and is deliberately explicit
about what it does *not* cover — most of clauses 4, 5, 6, 7.1.5, 7.2, 9.2 and 9.3 are organisational
questions this does not touch.

The CMMC and NIST SP 800-171 section is deliberately narrow and conservative. Read it before assuming any
reach there; it says plainly that this is a quality-records pattern and not a security control set.

Two currency notes as of August 2026, since standards move:

- **ISO 9001 is being revised.** The 2026 edition is expected to publish in September 2026. Clause references
  in the mapping are to the 2015 edition and should be checked against the current text.
- **CMMC Phase 2 was suspended in July 2026.** Phase 1 self-assessment requirements remain in force. If your
  contracts carry CMMC obligations, confirm current status with your contracting officer or an APEX
  counsellor rather than any third-party summary, this one included.

---

## What would genuinely help

If you read any of this, the most useful thing you can send back is **what is wrong with it** — a step that
would not survive your shop floor, a gate that would get routed around, an assumption about how inspection
works that does not hold where you are, a platform constraint I have missed.

A polite "interesting read" is the least useful possible response. A blunt "this falls apart the moment you
have more than one inspector per shift, and here is why" is worth considerably more.

There is no obligation attached to any of it.
