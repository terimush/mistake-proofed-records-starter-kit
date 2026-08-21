---
name: Problem report
about: Something in a document is wrong, unclear, out of date, or does not work as described
title: "[Problem] "
labels: problem
assignees: ''
---

Fill in what you can. A partial report is better than none — leave anything you do not know.

## Which document

Which file, and which section or step. A relative path helps — for example
`docs/03-implementation-guide.md`, build step 4.

## What you were doing

The task you were on when you hit it.

## Which route are you on

- [ ] Route A — SharePoint lists + Power Apps + Power Automate
- [ ] Route B — Dataverse + Power Apps + Power Automate
- [ ] Neither — a different platform (say which)
- [ ] Not building — reading only

Route matters in most reports. Behaviour that is a defect on Route B is often the documented platform
limitation on Route A, and the two need different fixes. See `docs/03-implementation-guide.md`, "Choosing your
platform".

## What happened

What the document said, or what the build did.

## What you expected

What you thought would happen instead.

## Can you reproduce it on a clean build

- [ ] Yes — reproduces on a fresh sandbox build following the documented steps
- [ ] No — only happens on my existing build
- [ ] Have not tried

If yes, the shortest sequence that reproduces it is the most useful thing in this report.

## Anything else

Version of the kit you are on, if you know it. Platform tier, if it is relevant.

**Do not paste anything from an employer's system** — no internal procedures, no screenshots of live
records, no real part numbers, supplier names or customer names. Recreate the problem with fabricated data.
