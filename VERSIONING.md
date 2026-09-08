# Versioning and change control

How this kit is versioned, archived and cited, and which parts of it you can build against without expecting
them to move.

Change control for a documentation kit is genuinely different from change control for software, and it is
worth stating rather than assuming. Nobody upgrades a document. What actually happens is that somebody built
something from version X two years ago and needs to know whether version Y invalidates it.

---

## What the numbers mean here

The kit uses semantic versioning — `MAJOR.MINOR.PATCH` — with the artefact being a pattern and a set of
documents rather than a library.

**MAJOR** — a change to the data model or to the gate definitions that would invalidate a build someone has
already made. Removing a column, changing what a gate requires, changing the stage sequence, or reversing a
statement about what a platform route can enforce. If a reader who built to the previous version would now be
running something the kit no longer describes, it is MAJOR. This is the only category that should ever cost a
reader work.

**MINOR** — new material that does not invalidate anything. A new worked example, a new document, a new
platform route, a substantial new section in an existing document. Someone on the previous version loses
nothing by staying there; they only miss out on the addition.

**PATCH** — corrections and clarifications. A wrong clause number, a broken cross-reference, a step that was
ambiguous, a refusal message reworded, a currency note updated because a standard moved. The pattern is
unchanged; the description of it got more accurate.

**The awkward case.** A correction to something that was wrong in a way people built on is a MAJOR release,
not a PATCH, even though the intent was only to fix an error. The test is what it does to an existing build,
not what motivated the change.

## Releases, archiving and citation

Each release is tagged in the repository at `https://github.com/terimush/mistake-proofed-records-starter-kit` and deposited in an
archive that mints a DOI.

**Each tagged release receives its own DOI.** That version DOI resolves permanently to exactly the files in
that release, which is what makes a citation checkable years later.

**The concept DOI always resolves to the latest version.** It is shown on the archive landing page for any
release of this repository, alongside the version DOI. Use the concept DOI when you mean the kit in general,
in a bibliography entry that should not go stale, or when linking readers to the current material; use the
version DOI when you mean the files somebody actually built from.

**When you cite the kit in work that depends on what it said, name the version you actually used**, and
prefer the version DOI over the concept DOI. "Built to the hard-gate pattern, v1.0.0" is a statement someone
can verify. "Built to the hard-gate pattern" is not, because the pattern may have had a MAJOR release since.
`CITATION.cff` carries the citation metadata and is updated at every release.

The changelog in `CHANGELOG.md` is the human-readable record of what moved between versions. Read it before
assuming a newer release is a drop-in replacement for the one you built against.

## Deprecation policy

Three properties are the pattern. Everything else in the kit is scaffolding around them, and everything else
is comparatively free to change.

- **Enforcement lives on the data, at the transition** — not on the screen, not on the button.
- **Earlier stages lock on passage** — once a gate passes, the fields that gate required stop being editable
  by ordinary users.
- **Evidentiary timestamps are system-generated** — never typed by a user.

Those are stable commitments. They will not be quietly weakened, reinterpreted, or relaxed in a PATCH note.
Anything that changes one of them is a MAJOR release, announced as such, and shipped with migration notes
saying what an existing build would need to change and how to tell whether yours is affected.

Deprecation of anything smaller — a column, a stage name, a document — follows the same shape at lower cost:
the replacement ships in a MINOR release alongside the old material, the old material is marked as
superseded, and it is removed in the next MAJOR release rather than in the release that superseded it. You
get at least one version in which both exist.

## What is deliberately not versioned

Platform behaviour is not under this kit's control and does not follow its version numbers. Licensing tiers,
column-security features, and flow trigger behaviour all change on the vendor's schedule. The kit describes
them as they were understood when written, and the correction arrives whenever someone reports the drift —
which is one of the more useful things you can send back. See `CONTRIBUTING.md`.

The same applies to standards. ISO 9001 clause references, CMMC status and NIST SP 800-171 practice families
move independently of this kit. `START-HERE.md` carries dated currency notes for that reason. Confirm your
own obligations with your contracting officer or an APEX counsellor rather than with a versioned document.

## Release cadence

There is not one. This is a personal project maintained in spare time, and releases happen when there is
something worth releasing. There is no roadmap, no scheduled version, and no commitment to a next release.
