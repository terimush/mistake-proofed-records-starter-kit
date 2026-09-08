# Mistake-Proofed Quality Records — one page

Take this with you. Everything on it works against a system you already run.

---

## What it is

A rule the software enforces rather than a policy people are asked to remember:

> **A record cannot advance to the next stage until the evidence that stage requires actually exists.**

No inspection result, no advance. No root-cause entry, no closure. The record *is* the process, so following
it and producing the evidence become one action rather than two competing ones.

```
Draft ──▶ Inspected ──▶ Dispositioned ──▶ Closed
     gate 1        gate 2            gate 3
```

**Where the gate lives decides whether it is real.** A greyed-out button is a suggestion — anyone who opens
the underlying table bypasses it. Validation on save is tied to the screen — another form or an import writes
straight past it. Only enforcement **on the data, at the transition** holds, whichever client wrote the
record. All three look identical from the form. The difference shows up in an audit.

## The three mistakes that make a gate decorative

- **Editable history.** If a record can change after it closes, it is a form and not evidence. Lock the fields
  a stage required once that stage passes. Corrections are new versioned records, not overwrites.
- **Self-approval.** A gate the same person can raise and approve is not a control. Compare approver against
  raiser and reject a match. One condition; more audit credibility than any amount of workflow decoration.
- **Optional depth.** "Root cause" as one free-text box gets filled with the symptom restated. Require three
  linked causal levels, each referencing the one above. The annoyance is the analysis.

## Four questions to test your own build

Try each against your own system. The answers are the honest description of what you have, and the one to
give an auditor rather than "the system enforces it".

1. **Can you set the status directly?** Grid view, datasheet, any other client. If the status column moves,
   your gate never ran.
2. **Can you edit a closed record?** If the fields a passed stage required are still editable, you have a form
   rather than evidence.
3. **Can you close a non-conformance with one causal level?** If yes, your causal-depth gate is decorative.
4. **Can you approve your own corrective action?** If yes, you have no approval control at all.

Questions 1 and 2 are usually platform permissions. Questions 3 and 4 live in your rules and must fail on any
platform — if either succeeds, the problem is your build.

## Two checks you can run tomorrow

Both are a single query against a system you already have. Neither is flattering.

- **The distribution check.** Pull created-on timestamps for every record and plot them by day. Real
  in-process use spreads across the working calendar. If most records appeared on a handful of days — or a
  hundred in one afternoon — the work happened somewhere else and the system is a filing cabinet.
- **The dwell check.** Measure elapsed time between created and last-modified on each record. Records created
  and finalised within seconds were not filled in while the work was being done.

## Where to get the kit

Free, MIT licensed, no sign-up, nothing to buy and nobody to call: `https://github.com/terimush/mistake-proofed-records-starter-kit`

Start with `START-HERE.md`. If you have 15 minutes, read `docs/02-hard-gate-pattern.md` and nothing else.

**Free help on contract obligations.** APEX Accelerators (U.S. Department of Defense funded, formerly PTACs,
a centre serving every state) and MEP centres counsel small businesses at no charge — better first calls than
any vendor.

**The boundary.** This is a records pattern. It is not a quality management system, not validated software,
and not certification. See `DISCLAIMER.md`.

**And the part nobody likes saying.** On the tier most businesses already have you can enforce the gate
conditions themselves — validation runs on the list, not in the form, so it holds whichever client wrote the
record. What you cannot do is lock one column against one user, so question 2 above will usually succeed
until the record is locked at closure. That is the platform, not your build. Anyone who tells you the free
route enforces nothing, or that it enforces everything, is selling something either way.
