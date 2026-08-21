# Release checklist

Everything that must be true before a version is tagged. It exists because two of the steps below are
irreversible: a DOI, once minted, is permanent, and so is whatever metadata was attached to it.

This file is for the maintainer. Nothing in it is needed to use the kit.

---

## 1 · Placeholders

The repository ships with `{{...}}` tokens rather than plausible-looking wrong values, deliberately — a
placeholder that is obviously a placeholder gets fixed, and a fake ORCID full of zeros gets published.

**The kit is not schema-valid while these are unresolved.** `CITATION.cff` will not render GitHub's "Cite
this repository" box until `orcid` is a real identifier or the line is deleted, and `date-released` must be
a real date. Nothing may be tagged, and no archive may be built, while a `{{` remains anywhere.

Resolve every one:

| Token | Where | What it becomes |
|---|---|---|
| `{{REPO_URL}}` | `CITATION.cff`, `CHANGELOG.md`, `CONTRIBUTING.md`, `VERSIONING.md`, `training/practitioner-talk.md`, `training/one-page-handout.md`, `training/workshop-outline.md` | The public repository URL |
| `{{RELEASE_DATE}}` | `CITATION.cff`, `CHANGELOG.md` | The date you actually tag, ISO format |
| `{{ORCID}}` | `CITATION.cff`, `.zenodo.json` | A registered ORCID iD — **or delete both lines entirely** |
| `{{CONTACT}}` | `CONTRIBUTING.md` | A contact route you are willing to publish |

**Four tokens, and all four are knowable before you tag.** That is deliberate: a DOI does not exist until the
deposit, the deposit happens when you tag, and a token that cannot be resolved before tagging would ship
inside the archive as a broken link. So nothing in the kit needs a DOI in order to be correct. Add the badge
to `README.md` after the first deposit, per §4; it ships from the next release onwards, and v1.0.0 is
complete without it.

Confirm with a single command before tagging:

```
grep -rn "{{" . --include="*.md" --include="*.cff" --include="*.json"
```

The only file that should still match is this one.

## 2 · Before the first release only

- **Register an ORCID** if you intend to have one. It is free and takes about five minutes, it is permanent,
  and it ties this kit to anything else you publish under one verifiable identity. Do it *before* the deposit
  — retrofitting an author identifier onto an existing DOI is more trouble than it is worth.
- **Enable the Zenodo integration for the repository before you cut the release.** Zenodo only archives
  releases created *after* the switch is turned on. A release tagged first and archived later is a release
  with no DOI, and the fix is to tag another one.
- **Decide the deposit type deliberately.** `.zenodo.json` sets `publication` / `technicalnote` rather than
  `software`, because this kit contains no executable code and `DISCLAIMER.md` opens by saying so. Depositing
  as software puts the word on the permanent landing page and in every downstream citation.

## 3 · Every release

- `CHANGELOG.md` has an entry for the version, and `[Unreleased]` is empty or accurate.
- `version` matches in `CITATION.cff`, `.zenodo.json`, the changelog heading and the git tag. Four places.
- **`.zenodo.json`'s `description` and `CITATION.cff`'s `abstract` say the same thing about what each
  platform route enforces.** Read them side by side, out loud. The Zenodo description is the only text in
  this repository that cannot be corrected after release, and it is the one nobody opens.
- The version increment follows `VERSIONING.md` — MAJOR for anything that would invalidate a build someone
  has already made.
- Every cross-reference resolves. Files get renamed and references do not follow.
- **Every test ID cited outside `docs/06-validation-and-test-plan.md` names the test it means.** They drift
  when tests are renumbered.
- No occurrence of "immutable", or of any claim that a record cannot be altered.
- **No claim that Route A locks a field on gate passage, or that Route A gates are advisory.** Both are
  wrong, in opposite directions, and both have appeared in drafts. `docs/09-microsoft-lists-build.md` §5 is
  the reference.
- **Every validation formula in the kit names the list it belongs on**, and no formula references columns
  from more than one list.
- No real company, supplier, customer, product, person or part number anywhere, including in the sample data.
- The sample CSVs are internally consistent: no record sitting at a stage its own data could not have passed,
  and no system-set timestamp in the future.

## 4 · After tagging

- Record the release timestamp, the version DOI and the concept DOI.
- Add the DOI badge to `README.md` — this is the one badge the style rules permit.
- Clone the public repository into a clean folder and follow `START-HERE.md`'s thirty-minute path exactly as
  a stranger would. Fix whatever only works because you know what you meant. It takes half an hour and it is
  the difference between a published artefact and a usable one.
