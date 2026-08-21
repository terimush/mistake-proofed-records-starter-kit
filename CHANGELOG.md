# Changelog

All notable changes to this kit are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versioning follows
[Semantic Versioning](https://semver.org/spec/v2.0.0.html) as interpreted for a documentation kit in
`VERSIONING.md`.

---

## [Unreleased]

Nothing yet.

## [1.0.0] — {{RELEASE_DATE}}

First public release.

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

[Unreleased]: https://github.com/terimush/hard-gate-starter-kit/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/terimush/hard-gate-starter-kit/releases/tag/v1.0.0
