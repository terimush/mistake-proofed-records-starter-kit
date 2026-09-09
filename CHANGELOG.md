# Changelog

All notable changes to this kit are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versioning follows
[Semantic Versioning](https://semver.org/spec/v2.0.0.html) as interpreted for a documentation kit in
`VERSIONING.md`.

---

## [Unreleased]

### Fixed

- **Three of the four published Gate 1 trigger conditions were wrong and are corrected.** The four P1 flows
  were exercised against records for the first time on 8 September 2026, and the session found that two of
  them never fired at all while a third looped. `docs/09-microsoft-lists-build.md` §4 now carries the tested
  expressions: F1b becomes `@equals(body/Modified, body/Created)` (fire on creation only, loop-safe by
  construction) in place of a guard that waited for a shadow column to be empty; F2a becomes
  `@not(empty(body/Inspection))` in place of `body/Inspection/Id`. F1a and F1c's conditions were already
  correct and are unchanged.
- **F1c wrote the owner's e-mail into the approver's shadow column.** The flow was built as a copy of F1a
  and inherited `Owner/Email` as the source for both `Owner_Email` and `Approved_By_Email`, so it put a
  wrong value into one of the three columns formula 3 adjudicates — and, because that column could then
  never agree with an empty `Approved_By`, it also looped. `docs/09-microsoft-lists-build.md` §4 now
  publishes F1c's full Update item mapping with the correct source.
- **An encoded column name survived the 7 September correction pass.** F1c's `Completed_On` expression still
  named `Completed_x005f_On`, so the "already stamped?" test was always true and the flow re-stamped the
  completion time on every run. Corrected, and the rule added: apply an encoding fix across the whole
  definition, then re-test.
- The claim that F2a's corrected wiring was untested is removed; F2a has now been tested end to end.

### Added

- **`docs/06-validation-and-test-plan.md` gains T18 and T19.** T18 proves a trigger condition actually fires
  — the opposite failure to T17's loop, and the one that leaves a flow On, clean and silent. T19 proves a
  stamp is written once and then preserved, a fault invisible on the first save. Both carry rows in the
  results record.
- **`docs/09-microsoft-lists-build.md` §7 gains *What the first test session taught*:** that a null column is
  absent from the trigger payload rather than present-and-empty; that silent non-firing is a failure mode
  which looks like success; that a flow made by *Save as* inherits the original's field mappings, which is
  both why F1b took 3.9 minutes against F1a's 34.7 and how the mapping defect propagated; and that lists
  must be matched by GUID, not by name.
- **`docs/02-hard-gate-pattern.md` §5 gains the refusal check**, with the list's own refusal message captured
  from the reference build — the first time the separation-of-duties gate has been observed refusing a save
  — together with the caution that the observed refusal was over-determined, and that a gate reading shadow
  columns can be corrupted by the flows that maintain them.
- **`docs/08-troubleshooting-and-faq.md` gains two entries:** *My flow is switched on, the checker is clean,
  and it never runs*, and *The completion timestamp keeps moving*.

### Changed


- **Renamed.** The kit is now *Mistake-Proofed Quality Records* (repository `mistake-proofed-records-starter-kit`). "Hard gate" remains the name of the pattern inside the documents; only the title people meet first has changed.

- **The kit has been built from its own instructions**, by hand and with a stopwatch, in a personal
  Microsoft 365 tenant, following `WI-M365-01`; that tenant is no longer accessible and the system in the
  screenshots was rebuilt by the same process in a second one. `START-HERE.md` now says so, in one paragraph,
  and `docs/09-microsoft-lists-build.md` §7 records what the build taught.
- **The time estimate was replaced with a measured figure.** "An evening, or a weekend" in
  `docs/03-implementation-guide.md` and `START-HERE.md` becomes about two working days: 8.78 hours actual
  against a 4.36-hour plan for three lists, three formulas and four flows (stopwatch-timed, self-reported,
  one build). `docs/09-microsoft-lists-build.md` §4 carries the per-part table; figures not recorded were
  left as bracketed placeholders rather than guessed.
- **The placeholder build figures are filled in.** The four flow rows of the cost table in
  `docs/09-microsoft-lists-build.md` §4 now carry the figures from a second, instrumented build of the flows
  alone on 5 September 2026 (F1a 34.7 min, F1b 12.0 min build and test, F1c 7.7 min, F2a 21.8 min; 79.2 min
  with the failure-log list), stated as not summing with the 8.78 h of the first build. The lists-and-columns
  row says "not separately timed", with the 316 minutes the first build leaves for lists, columns and flows
  together; the unbuilt flows are said to be unbuilt. `docs/05-rollout-runbook.md` and
  `training/practitioner-talk.md` lose the last two "an evening" estimates.
- **A review of the running flows on 7 September 2026 found two defects, both corrected.** Every trigger
  condition in F1a, F1b and F1c, and the condition inside F2a, addressed the shadow columns by the encoded
  internal name (`body/Inspector_x005f_Email`) where the connector's trigger body used the plain key; the
  reference resolved to null, the self-trigger guard was true on every save, and F1a ran itself every thirty
  seconds for two days. F2a's reverse-lookup write was also wired to the parent's existing value rather than
  the trigger's `ID`, so it could never set the lookup; its condition carried an empty row and the flow had
  never been renamed. All conditions are rewritten with plain keys and the F2a wiring is corrected; the exact
  expressions are now in `docs/09-microsoft-lists-build.md` §4, *Trigger conditions and the F2 write*, with
  the read-back rule, and `docs/03-implementation-guide.md`'s loop warning carries the rule in two sentences.
  F2a was still untested at that point; it was tested end to end on 8 September, in the test session
  recorded in the Fixed block above.
- **`docs/06-validation-and-test-plan.md` gains T17, the loop test** — one edit, exactly one run, five
  minutes of silence, and a version number that did not climb — which each F-flow must pass before it is
  left on.
  `docs/08-troubleshooting-and-faq.md` gains *"My flow runs every thirty seconds and every run succeeds"*
  and *"The reverse lookup on the inspection never fills"*.
- **Formulas are budgeted as design work, not data entry.** The three validation formulas took 211 minutes
  against 31 allowed — about 40 % of the build. `docs/03-implementation-guide.md` says so once;
  `docs/09-microsoft-lists-build.md` §4 says why.
- **"My lists" versus site lists is now the first warning in the build.** Validation settings do not exist
  on personal lists in the Microsoft Lists app. A boxed warning opens `docs/09-microsoft-lists-build.md`,
  `START-HERE.md` gives it one sentence, and `docs/08-troubleshooting-and-faq.md` gains *"I can't find
  Validation settings"*, including what to do with data already entered.
- **The build order in `docs/09-microsoft-lists-build.md` §4 is annotated with the sequence that actually
  worked** — `WI-M365-01` Parts A to F mapped onto the ten steps, including the formulas-with-each-list
  order the work instruction used, why it is safe, and the prediction-before-test and stopwatch practices
  the step table had not asked for.

### Added

- **`GLOSSARY.md`** — every term the kit uses and does not stop to explain, grouped by when you meet it and
  written for someone who runs a quality system rather than someone who builds software. A cold first reader
  could not define *canvas app*, *delegation*, *shadow column*, *the grid*, *on the server* or *tenant* after
  thirty minutes with the kit, and every one of them is load-bearing.
- **`examples/sample-standalone-corrective-actions.csv`** — the corrective actions belonging to the
  standalone non-conformance example, with the `Verified_On` column that example adds. They were previously
  mixed into the core file, where their parents do not exist.

### Fixed

- **Formula 1's closure clause was satisfied by a machinery column being absent** — the exact shape
  `docs/01-data-model.md` calls the most expensive mistake in the kit. Re-anchored on `Result`, which the user
  sets. A dispositioned rejection could previously be closed in one grid save by clearing
  `NC_Reference_Text`.
- **Formula 2 did not enforce what its prose claimed.** It compared each causal level only against the one
  above it, so *"Operator error / Training gap / Operator error"* saved cleanly. All three pairs are now
  compared.
- **Formula 3 excused blank shadow columns**, because a comparison against an empty column is unequal. Both
  operands of both comparisons are now required to be present, and the `Nonconformance` lookup on
  `CorrectiveAction` is a required column so the raiser's address can always be copied down.
- **Flow F3 tested the wrong column.** Its specification said a corrective action "carries an `Approved_By`,
  which the list has already refused if it matched the raiser" — false, because formula 3 short-circuits
  while `Approved` is No, which is its default. A self-approval could reach closure with no refusal anywhere
  in the chain. F3 now tests `Approved`.
- **The yes/no comparison was written two ways** — `=TRUE` in formula 1, `<>"Yes"` in formula 3. Unified on
  the bare boolean, with an instruction to confirm it on your own tenant before trusting either.
- **"Blocks the reverse attack" was an overclaim.** Clearing `Result` is refused; *changing* it is not, and
  the record then closes as an accepted receipt. `docs/03-implementation-guide.md` step 7 always said so;
  `docs/09-microsoft-lists-build.md` now agrees, and the ceiling table names the consequence.
- **Nothing in the Lists build advanced a record**, while three other documents described a transition
  mechanism that does not exist on that route. Resolved in favour of what the build actually does: on Route A
  the user sets `Stage` and validation refuses illegal states. `docs/01-data-model.md`,
  `docs/03-implementation-guide.md` and `docs/09-microsoft-lists-build.md` now say the same thing, and
  `Stage` stays on the form.
- **The thirty-minute path was unrunnable by the reader it describes.** It required a created-date column and
  two SQL queries. It now branches on whether you have such a system, tells the reader who does not that they
  already have their answer, gives a spreadsheet method for the reader who does, and ends in one decision and
  one action.
- **`docs/08-troubleshooting-and-faq.md`** described a closed-with-nothing-filled-in record as "the platform,
  not a bug". Formula 1 refuses exactly that record; the symptom now points at the three build faults that
  actually cause it.
- **Three different answers to "what do I build first"** across `README.md`, `START-HERE.md` and
  `docs/03-implementation-guide.md`. One answer now, with the reasoning: incoming inspection first because it
  is the easiest to learn on, the non-conformance intake second because that is where the value is.
- **`docs/06-validation-and-test-plan.md`** — T06's second half expected a refusal no formula can make on
  Route A; T07's precondition described a record formula 2 refuses outright; several tests expected per-field
  messages on a route that has one message per list; T01 could not pass before flow F3 existed. All corrected
  per route.
- **The flow inventory read as five flows.** A SharePoint trigger binds to one list, so it is nine, plus the
  archive job. Every flow's write is itself subject to validation and can be refused silently, so every flow
  now carries the logging instruction only F3 had.
- **The closure lock had two unstated failure modes** — it needs a *site* owner connection, not a list owner,
  or the flow locks itself out of the records it must revert; and a reverted closure left a record nobody
  could edit, because nothing re-granted the permissions closure removed.
- **The archive job destroyed the evidence it exists to preserve.** Archiving is a create-plus-delete, so the
  copy carries the service account and today's date. The original `Created` and `Created By` must be carried
  into columns of your own first.
- **Column settings the formulas depend on were never stated** — choice defaults cleared, fill-in choices off
  on `Stage`, `Enforce unique values` on the three reference columns, `Quantity_Received` required. Each was
  a hole in a formula rather than a preference.
- **Four columns marked "system-set" had no writer named**, and the direction of the
  `Inspection`↔`Nonconformance` link was unstated — linking from the wrong side leaves a record that silently
  cannot advance. `Complete` was added to `CorrectiveAction` so `Completed_On` has an event to hang off.
- **Sample data** — `examples/sample-corrective-actions.csv` carried an extension column and three rows whose
  parents live in another example's file; `examples/sample-data.csv` set `Closure_Ready` true on accepted
  receipts that flow F3 would set false.

## [1.0.0] — 2026-08-20

Repository import. Not tagged and not released; first release is 1.1.0. Kept as the baseline the 1.1 changes are measured against.

### Added

- **`README.md`** — the problem, the idea, what is in the kit, and the boundaries, including where Microsoft
  Lists enforcement genuinely stops.
- **`START-HERE.md`** — three ways into the kit depending on whether you have 15 minutes, 30 minutes or half
  a day, plus the currency notes on ISO 9001 and CMMC status.
- **`docs/01-data-model.md`** — the three tables (`Inspection`, `Nonconformance`, `CorrectiveAction`), their
  columns with generic types, the relationships between them, and the two notes on person columns and
  system-set dates.
- **`docs/02-hard-gate-pattern.md`** — the pattern itself: what a gate is, the three places you can enforce
  one and why two of them fail an audit, the three mistakes that make a gate decorative, two design
  principles, and the distribution and dwell checks.
- **`docs/03-implementation-guide.md`** — the platform choice and its consequences, the infinite trigger loop
  warning, eight build steps, the single symptom that means you have outgrown Microsoft Lists, guidance on
  adapting the stages, and an honest time estimate.
- **`docs/04-compliance-mapping.md`** — a mapping to ISO 9001:2015 clauses and a deliberately narrow and
  conservative treatment of CMMC and NIST SP 800-171, each with an explicit statement of what is not covered.
- **`docs/05-rollout-runbook.md`** — getting from a built pilot to how the shop actually works: choosing the
  first process and team, the phases, the parallel run and its time limit, what to do when a gate gets routed
  around, and how to stop without losing the records.
- **`docs/06-validation-and-test-plan.md`** — a test suite for finding out whether what you built does what
  you think it does, the results record it produces, and the description of your build you can honestly give
  an auditor.
- **`docs/07-gate-self-test.md`** — a printable thirty-minute checklist for deciding whether the gates in a
  system you already run are real, usable on a build that did not come from this kit.
- **`docs/08-troubleshooting-and-faq.md`** — the failures people hit while building this, written as symptom,
  cause and fix, and the questions a sceptical quality manager or auditor actually asks.
- **`docs/09-microsoft-lists-build.md`** — the complete build on Microsoft Lists: the four enforcement
  mechanisms and their relative strength, the shadow-column technique that person, lookup and
  multiple-lines-of-text columns require, a table of which condition belongs on which list, the three
  validation formulas that follow from it — one per list, because a list holds one formula — the
  flow-maintained flag that carries a cross-list condition, the closure lock and its permission-scope
  ceiling, a ten-step build order, and the ceiling stated in one table.
- **`docs/10-canvas-app-gates.md`** — the app layer: why a canvas gate is advisory, a four-layer table of
  which gate belongs where, what the app genuinely adds that the list cannot (per-gate refusal messages,
  refusal before submission, capture at the moment of the work), the SharePoint delegation limit that makes a
  counting gate silently wrong past 500 rows, and a ranked ladder of workarounds — explicitly separated from
  the popular techniques that remove the warning while leaving the wrong answer in place. An appendix sets
  out the read-only-list arrangement that would make an app gate enforceable, clearly marked as untested and
  outside the build.
- **`examples/incoming-inspection-checklist.md`** — a complete worked example with all three gates written as
  explicit conditions, refusal messages, and a failing record walked through every stage.
- **`examples/nonconformance-intake.md`** — a non-conformance record that stands alone rather than hanging off
  an inspection, with five stages, causal-depth conditions that defeat the usual ways of faking depth, and two
  separate approver separations.
- **`examples/production-hard-gate-checklist.md`** — in-process production checks, the two optional tables
  they need, coverage arithmetic against a locked check frequency, and the condition that makes retrospective
  entry visibly impossible.
- **`examples/compliance-status-dashboard.md`** — six read-only panels, the two integrity panels placed first,
  and an explicit statement of what a green dashboard does not mean.
- **`examples/sample-data.csv`**, **`examples/sample-nonconformances.csv`**,
  **`examples/sample-corrective-actions.csv`**, **`examples/sample-standalone-nonconformances.csv`**,
  **`examples/sample-operations.csv`**, **`examples/sample-production-checks.csv`** — fabricated records for
  every table, including records deliberately left mid-stage so the gates can be seen refusing.
- **`DISCLAIMER.md`** — no warranty, no compliance claim, and the specific position on what a Route A build
  can and cannot be described as.
- **`CONTRIBUTING.md`** — what is useful to send back, the standard a worked example must meet, style rules,
  and the licence position on contributions.
- **`VERSIONING.md`** — what MAJOR, MINOR and PATCH mean for a documentation kit, how releases are archived
  and cited, and which properties of the pattern are stable commitments.
- **`training/workshop-outline.md`** — a half-day hands-on workshop outline with a timed plan, the live
  demonstration sequence, and facilitator notes.
- **`training/practitioner-talk.md`** — speaker notes for a 45-minute practitioner talk with 15 minutes of
  questions, including the questions you will be asked and the honest answers.
- **`training/one-page-handout.md`** — a single printable page covering hard gates, the three mistakes, the
  self-test questions and the two analytical checks.
- **`.github/ISSUE_TEMPLATE/`** — templates for problem reports, worked-example proposals, and shop-floor
  objections.
- **`.github/pull_request_template.md`** — the contribution checklist.
- **`RELEASE-CHECKLIST.md`** — the maintainer's pre-flight, including every placeholder token and the two
  steps that are irreversible once a DOI is minted.
- **`CITATION.cff`**, **`.zenodo.json`**, **`LICENSE`** — citation metadata, archive metadata, and the MIT
  licence.

### Notes

- **Versioning starts here.** Earlier draft numbering was used while the material was being written and was
  never published. No release before 1.0.0 exists publicly, and any reference to an earlier number is to an
  internal draft rather than to a released version. The public history begins with this entry.
- **All sample data is fabricated.** No real company, supplier, customer, product or person appears anywhere
  in the kit.
- **Standards references are dated.** ISO 9001 clause references are to the 2015 edition. CMMC and NIST
  SP 800-171 material reflects the position stated in `START-HERE.md`. Check both against current sources
  before relying on them.
- **This release documents a pattern, not software.** There is no executable code to install, and nothing
  here is validated for your use. See `DISCLAIMER.md`.
- **Microsoft Lists is the starting platform, and the guidance on it changed late in drafting.** Earlier
  drafts described Lists gates as advisory on the grounds that the platform has no column-level permissions.
  That understated it: list validation formulas are evaluated by the server on save, so a correctly written
  gate condition holds whichever client wrote the record. The documents were corrected before release, and
  `docs/09-microsoft-lists-build.md` states both what Lists enforces and where it genuinely stops.
- **An adversarial review before first publication changed several things, and they are worth knowing about
  if you read a pre-release copy.** A SharePoint list holds one validation formula and one message, so the
  gates for a list are now given as a single formula rather than one per gate. A formula sees one row of one
  list, so each condition is now placed on the list that owns its columns — causal depth on `Nonconformance`,
  the approver comparison on `CorrectiveAction` — and cross-list conditions are carried by a flow-maintained
  flag the formula reads. Validation formulas use display names, not internal names, and cannot read
  multiple-lines-of-text columns at all. The shadow email columns are written after the save, so a stage
  change needs two saves or an app. The in-process check counter counts in-process checks only. The
  read-only-list arrangement moved from the body of `docs/10-canvas-app-gates.md` to an appendix, because the
  assumption underneath it is unconfirmed and the body of the kit should not depend on it. Sample data and
  metadata were corrected to match.

[Unreleased]: https://github.com/terimush/mistake-proofed-records-starter-kit/compare/5481729f1949a05d89e8af28d09a6adc7f0f929d...HEAD
[1.0.0]: https://github.com/terimush/mistake-proofed-records-starter-kit/commit/5481729f1949a05d89e8af28d09a6adc7f0f929d
