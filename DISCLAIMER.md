# Disclaimer

Read this before you build anything from this kit. It is short and it is written plainly, because a
disclaimer someone actually reads is worth more than one they skip.

---

## This is documentation, not validated software

The kit is a set of markdown documents describing a pattern. It contains no executable code, no installer
and no configuration you can deploy. Anything running in your business is something you built, on a platform
you licensed, configured by you. It has not been validated, qualified or tested against your process by
anyone.

There is no warranty of any kind. The licence text in `LICENSE` says this in the usual legal form; this page
says it in ordinary words. The documents may contain errors. Platform behaviour described here may have
changed since it was written. Standards referenced may have been revised. You are responsible for checking
all of it against current sources before you rely on it.

## Three separate things you should not assume

**No warranty on the documents.** They are offered as-is, as a contribution to the small-manufacturer
community. Nobody is liable to you if a step is wrong, out of date, or does not apply to your platform tier.
There is no support arrangement attached to them and none is implied.

**No claim about what you build from them.** Nothing built from this pattern is thereby compliant with ISO
9001, CMMC, NIST SP 800-171, a customer's flow-down clauses, or any other standard or contract. The kit does
not deliver certification and cannot. `docs/04-compliance-mapping.md` is deliberately explicit about the
boundary — it maps where the pattern helps you produce evidence, and lists at length what it does not touch.
Qualification, validation and regulatory compliance remain entirely yours.

**Route A has a specific ceiling, and it matters more than the rest.** On Route A (Microsoft Lists) there
are no column-level permissions. Gate conditions written as list validation *are* enforced by the server on
save, so a record cannot reach a stage without the evidence that stage requires — but a field can still be
altered after its stage has passed, until the record is locked at closure, and rules that must read a second
list are reverted after the fact rather than refused. `docs/09-microsoft-lists-build.md` states the full
ceiling in one table and `docs/03-implementation-guide.md` describes the platform choice. Describing a Route A
build as though nothing in a record could be altered would be inaccurate. Route B (Dataverse, a paid
licensing step) is where
enforcement on the data becomes true. If you are unsure which you have, run build step 7 and write down which
of the four cheats your build actually blocks. That sentence is the honest description of what you have.

## If you operate under a regulated obligation

If your contracts carry obligations you are not certain about — defence flow-downs, CMMC status, customer
quality clauses — do not resolve them from this kit or any other third-party summary. Free, publicly funded
help exists for exactly this.

**APEX Accelerators** are funded by the U.S. Department of Defense, were formerly known as PTACs, and provide
free counselling to small businesses on federal contracting obligations, with a centre serving every state.
**Manufacturing Extension Partnership (MEP) centres** do the equivalent on the manufacturing side. Either is
a better first call than a vendor, and better than this document. Your contracting officer is better still
for what a specific contract requires of you.

## Independence

This kit was written and released by Tererai Mushangwe in a personal capacity. It is not derived from,
endorsed by, or associated with any employer, and no proprietary material of any organisation is included in
it. All sample data is fabricated. No organisation stands behind this material, and you should not present it
as though one does.
