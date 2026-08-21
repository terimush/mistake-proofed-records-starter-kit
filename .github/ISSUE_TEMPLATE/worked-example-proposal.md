---
name: Worked example proposal
about: Propose a new worked example applying the pattern to a different process
title: "[Example] "
labels: worked-example
assignees: ''
---

Propose it here before writing it, so you do not spend an evening on something already in progress. The
standard an example has to meet is in `CONTRIBUTING.md`.

## The process being modelled

What it is, and where it sits in a shop. One paragraph. Say why it is worth an example — what it exercises
that `examples/incoming-inspection-checklist.md` does not.

## The stages

The stage sequence, in order. For instance: `Draft` → `Inspected` → `Dispositioned` → `Closed`.

## The gates, as explicit conditions

One block per gate, written the way `examples/incoming-inspection-checklist.md` writes them — a `REQUIRE`
block naming every field, an `ON PASS` block naming what is stamped and what locks, and the refusal message
the user would see.

```
Gate 1 (Stage → Stage):
    REQUIRE  ...
    ON PASS  ...
```

A prose description of a gate is not enough to build from. If a gate is hard to write as a condition, that is
usually worth saying too.

## What evidence each gate requires

For each gate, the fields or linked records that must exist, and who supplies them.

## Which route did you build it on

- [ ] Route A — Microsoft Lists (validation enforced, no column-level permissions)
- [ ] Route B — Dataverse (enforceable)
- [ ] A different platform (say which)
- [ ] Not built yet

If any condition in your example cannot be expressed as a Lists validation formula, say which and why —
reading another list, a person column, or a field's previous value are the usual reasons. State the expected
result on both
routes.

## What it does not cover

Every document in this kit carries a boundary section. What does your example leave out, and where would
somebody need to adapt it?

## Confirmations

- [ ] All data in this example is fabricated. No real company, supplier, customer, product, part number or
      person appears in it.
- [ ] Nothing in it is derived from an employer's system, procedure, form or record, and I have the right to
      contribute it under the MIT licence.
- [ ] I have walked a record through it, including the failing path.
