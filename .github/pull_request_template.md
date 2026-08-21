## What changed, and why

One or two sentences. If it relates to an issue, link it.

## Checklist

- [ ] **All data is fabricated.** No real company, supplier, customer, product, part number or person appears
      anywhere in this change. The kit's conventions are `Supplier Nine`, `BRACKET-A1`, `RC-2041`, `Person A`.
- [ ] **Nothing is derived from an employer's systems, procedures, forms or records**, and I have the right to
      contribute this under the MIT licence in `LICENSE`.
- [ ] **British English** — licence (noun), organisation, recognise, analyse, behaviour. "-ise", never "-ize".
- [ ] **No claim of certification or compliance.** Nothing in this change states or implies that the kit, or
      anything built from it, makes anyone compliant with any standard or contract.
- [ ] **The Route A / Route B distinction is stated wherever it is relevant** — on Route A, gate conditions
      are enforced by list validation on every write path, but there are no column-level permissions, so a
      field can be altered after its stage has passed until closure. Route B is where per-column, per-record,
      stage-conditional locking becomes available. See `docs/03-implementation-guide.md`.
- [ ] **Cross-references resolve.** Every relative path in backticks points at a file that exists, and every
      section reference points at a section that exists.
- [ ] **`CHANGELOG.md` updated** under `## [Unreleased]`.
- [ ] Style read: second person "you", no marketing language, no exclamation marks, no emoji, prose wrapped at
      roughly 110 characters, and the vocabulary rules in `CONTRIBUTING.md` followed.

## Anything a reviewer should check

Assumptions you made, platform behaviour you could not verify, or parts you are unsure about.
