# Contributing

This is a personal project maintained in spare time. That shapes everything below, so it is worth saying
first rather than last.

---

## What is genuinely useful

The most valuable thing you can send is **what is wrong with it**.

A step that would not survive your shop floor. A gate that would get routed around by the second shift. An
assumption about how inspection works that does not hold where you are. A platform constraint I have missed,
or one I have described incorrectly. A refusal message that would generate a support call rather than prevent
one.

A polite "interesting read" is the least useful possible response. A blunt "this falls apart the moment you
have more than one inspector per shift, and here is why" is worth considerably more. There is no need to
soften it, and no obligation attached to sending it.

## How to report a problem

Open an issue using `.github/ISSUE_TEMPLATE/problem-report.md`. It asks which document, what you were doing,
which route you are on, what happened, what you expected, and whether it reproduces on a clean build. Route
matters in almost every report — behaviour that is a bug on Route B is frequently the documented platform
limitation on Route A, and the two need different fixes.

If your objection is to the pattern rather than to a document, use
`.github/ISSUE_TEMPLATE/this-would-not-work-here.md` instead. That template exists for shop-floor objections
and it is the one I would most like to receive.

The repository is at `https://github.com/terimush/mistake-proofed-records-starter-kit`. If issues are not the right channel for
what you have, `https://github.com/terimush/mistake-proofed-records-starter-kit/issues`
reaches me.

## Proposing a worked example

There are four worked examples. More would help, because incoming inspection is the easy case and the
pattern is more interesting where the process is messier than any of the four.

Propose one with `.github/ISSUE_TEMPLATE/worked-example-proposal.md` before writing it, so you do not
duplicate work already in progress.

**The standard a worked example must meet:**

- **Fabricated data only.** Use the kit's conventions — `Supplier Nine`, `BRACKET-A1`, `RC-2041`, `Person A`.
  No real company, supplier, customer, product, person or part number, anywhere, including in screenshots.
- **Gates stated as explicit conditions.** Written the way `examples/incoming-inspection-checklist.md` writes
  them: a `REQUIRE` block naming every field, an `ON PASS` block naming what is stamped and what locks, and
  the refusal message a user would actually see. Prose descriptions of a gate are not enough to build from.
- **Expected result on both routes.** State what happens on Route A (Microsoft Lists) and what happens on
  Route B (Dataverse). If a condition in your example cannot be expressed as a Lists validation formula — it
  reads another list, or a lookup column, or a person column, or a **multiple lines of text** column, or a
  field's previous value — say so in the example rather than leaving a reader to discover it. Name the list
  each formula belongs on, too: a condition over columns that live on two different lists is not a validation
  formula anywhere, however it is written.
- **Honest about what it does not cover.** Every document in this kit carries a boundary section. Yours
  should say which parts of the process it leaves out and where it would need adapting.
- **Built and tested, not theorised.** Say which route you built it on and walk a record through it, including
  the failing path.

## Style rules a contribution must follow

- **British English.** Licence (the noun), organisation, recognise, prioritise, minimise, analyse, behaviour.
  "-ise", never "-ize".
- **Second person.** Address the reader as "you". Not "we", not "the user should".
- **Plain and unhyped.** No marketing language, no exclamation marks, no emoji, and no badges other than the
  DOI badge in `README.md`.
- **No vendor pitching.** Naming a platform to describe what it does is fine and necessary. Recommending a
  product, linking a partner, or writing anything that reads as a sales route is not. That applies to my own
  work as much as anyone's.
- **No claim of certification or compliance.** The kit helps you produce evidence. It does not make anyone
  compliant with anything, and no contribution may imply otherwise.
- **Do not describe a record as beyond alteration.** That claim overstates what any of these platforms give you.
  Records here are constrained by rules and permissions — which an administrator can still change, and which
  is a meaningfully weaker claim. Write what is actually true, and write it per route: a record cannot reach a
  stage without the evidence that stage requires, on every write path, because the condition is evaluated on
  the item rather than in the form; timestamps are system-generated; a closed record is locked against edit by
  ordinary users. On Route A, do **not** write that fields lock on gate passage or that earlier stages are
  uneditable — there are no column-level permissions on that tier and neither is true before closure.
- **Wrap prose at roughly 110 characters** and cross-reference other files by relative path in backticks.

## Licence and rights

Contributions are accepted under the MIT licence in `LICENSE`. By opening a pull request you confirm you have
the right to submit what you are submitting.

In practice that means one thing above all: **do not paste material owned by your employer.** Not a procedure,
not a form layout, not a screenshot of an internal system, not sample data with a real part number in it.
This kit is released in a personal capacity and carries no organisation's material, and it needs to stay that
way. If you are unsure whether something you built at work is yours to give, assume it is not.

## Expectations, honestly

There is one maintainer, no company, no team, no support desk and no funded roadmap. I work on this in spare
time.

Issues and pull requests may go unanswered for a long time, and some may never get a response. When that
happens it is a comment on this project's capacity, not on the worth of what you sent. If a fix matters to
you and I have not acted on it, fork the kit and ship it yourself — the licence exists so you do not have to
wait for me.
