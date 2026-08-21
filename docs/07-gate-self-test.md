# Gate self-test

Thirty minutes against a system you already run, built from this kit or not. Four things you attempt and two
queries you run. At the end you will know whether your gates are real, and exactly which sentence you are
entitled to say about them.

---

## Before you start

Do this on a record you created for the purpose, in a sandbox if you have one. Three of the four attempts
write data, and one tries to close a record that should not close.

Attempt C needs **two user identities**. Without a second one you cannot test self-approval, which is where
most systems quietly fail. Borrow a colleague for five minutes.

Sign in as an ordinary user. Testing enforcement from an administrator account tells you nothing about the
people the gates exist for.

---

## 1 · The four attempts

These are things you *try to do*, not things you inspect. A gate you can only see is not a gate.

### A · Set the status directly

**Attempt.** Open the underlying table — datasheet or grid view, quick edit, an export-and-reimport, any
client other than the form you normally use. Set the status of a blank record to your final value, whatever
"finished" is called in your system. Save.

**A real gate refuses this.** The status only ever changes as the result of a rule that checked the evidence.

**If it saved.** Your enforcement lives on the screen rather than in the data, and anyone with edit access can
finish a record that contains nothing. This is usually fixable without changing platform: state the rule as a
condition the *record* must satisfy at each stage and put it in the data layer's own validation, rather than
as a check the form performs before submitting. On Microsoft Lists that is list validation settings, and
`docs/09-microsoft-lists-build.md` §3 shows the formula shape.

### B · Edit a field after the stage that required it has passed

**Attempt.** Take a record that has moved past the stage where the inspection result was recorded. Change that
result to something else, first through the form, then through the grid view.

**A real gate refuses this.** Fields lock when the stage that required them passes; later stages stay open.

**If it saved.** The record is a form, not evidence: anything in it can be adjusted later to match whatever
story is needed, which is exactly what an auditor is asking about when they ask how records are controlled.

This is the one of the four that some platforms genuinely cannot fix. Validation rules generally cannot see a
field's previous value, and without column-level permissions there is nothing to revoke. Where that is your
situation, the available answer is to lock the whole record once it closes and to be accurate about the
window before that.

### C · Approve your own action

**Attempt.** Find a corrective action you raised yourself. Put your own name in the approval field and close
it.

**A real gate refuses this**, by comparing the approver against the raiser and rejecting a match.

**If it saved.** You do not have an approval control, you have an approval field. This one is worth fixing
first because it is usually one condition to write and it is the failure an auditor recognises fastest.

**Then check how it compares.** If two people in your organisation share a display name, the comparison must
be on the directory identifier rather than the name. A name comparison fails both ways: it blocks two
different people with the same name, and it permits genuine self-approval by anyone holding two accounts.

### D · Type a date that carries evidentiary weight

**Attempt.** Find the date that says when the inspection or the check actually happened. Try to type a
different value into it. Then set your workstation clock forward a week and create a record.

**A real system refuses both.** Dates that carry evidentiary weight are stamped by the platform when the
transition occurs. The only date a user should type is a planned due date, which is an intention rather than
a record.

**If either worked.** Your chronology is whatever people entered. Every claim you make about work being
recorded as it happened rests on it, so this is not a small finding.

---

## 2 · The two queries

From `docs/02-hard-gate-pattern.md` §5. Both run against records you already hold, and neither is flattering.

**The distribution check.** When were your records actually created?

```
SELECT   date(Created_On) AS day, count(*)
FROM     <your records table>
GROUP BY day
ORDER BY day
```

*A bad result looks like* a flat calendar with spikes: a hundred records created on one afternoon, or every
record in a quarter created in the week before an audit. That means the work happened somewhere else and the
system is a filing cabinet. Real in-process use spreads across the working calendar.

**The dwell check.** How long was each record open?

```
SELECT   Reference, Created_On, Modified_On,
         (Modified_On - Created_On) AS dwell
FROM     <your records table>
ORDER BY dwell ASC
```

*A bad result looks like* a large block of records with a dwell of seconds. Those were typed up afterwards
from something else — a notebook, a spreadsheet, somebody's memory. A healthy system shows records that stay
open for as long as the job takes.

---

## 3 · Scoring, without false comfort

Count the attempts that were **refused**. Partial credit is not a passing grade for the claim most people want
to make, so take the sentence that matches your score and do not upgrade it.

**Four refused, both queries healthy.** *"The stages are enforced by the software. A user cannot set the
status directly, change a field after the stage that required it, approve their own action, or type a date the
record depends on. Here are the test records."*

**Three refused.** *"The software enforces [the three]. It does not prevent [the fourth]; that is a known
limitation, and it is managed by [restricting edit access / reviewing the change history]."* Name the gap
yourself. It is a much better conversation than having it found.

**One or two refused.** *"There is a form with a status field and a written procedure. The software does not
enforce the sequence."* That is an honest description of most quality systems, and it is not what anyone
should be told the system does.

> **Before you take that score at face value, check which two.** On a platform with no column-level
> permissions — SharePoint lists on a standard Microsoft 365 subscription, among others — attempts **B** and
> **D** are *expected* to succeed no matter how well the system is built, because there is nothing to revoke.
> A correct build on that tier therefore scores two: A and C refused, B and D permitted. That is not the
> sentence above; it is this one:
>
> *"The software refuses a status change without the evidence that stage requires, on any client, and it
> refuses self-approval. It cannot prevent a field being altered after its stage has passed, or a system date
> being overwritten, because this platform tier has no column-level permissions — that is tested and logged
> as a known limitation, and the record is locked once it closes."*
>
> **A and C are the two that must be refused on any platform.** They live in your rules rather than in
> permissions. If either of those succeeded, the score above applies to you and the platform is not the
> reason.

**None refused.** *"These are records of what people typed, at whatever point they got round to typing it."*

No score here is certification of anything, and none of it is software validation in the regulated sense —
see `docs/06-validation-and-test-plan.md` §1, which is the longer version of this page.

---

## 4 · What this page does not tell you

**Whether your process is right.** Every attempt above asks whether the software holds, not whether you are
inspecting the right things or whether your causal analysis is any good.

**Whether your records are unalterable.** They are not, and none can be. A sufficiently privileged
administrator can change or remove anything in any platform of this kind. The claim available to you is
narrower: ordinary users cannot bypass the gates, and changes leave a trace.

**Whether the trace survives.** Whether a deletion or an override is logged, and for how long, is a platform
and licensing question. Find out before you rely on the answer.

---

## 5 · Tick-box summary

Print this page, run the checks, keep the sheet.

System: ______________________  Route / platform: ______________________

Tested by: ______________________  Date: ______________  Second identity: ______________

| # | What you attempted | Refused | Succeeded |
|---|---|---|---|
| A | Set the status directly, outside the form | ☐ | ☐ |
| B | Changed a field after its stage had passed | ☐ | ☐ |
| C | Approved an action you raised yourself | ☐ | ☐ |
| D | Typed over a system-set date | ☐ | ☐ |

| # | What you verified | Correct | Not correct |
|---|---|---|---|
| C2 | Comparison is on directory identity, not display name | ☐ | ☐ |
| D2 | Client clock change did not move the stamped time | ☐ | ☐ |

| # | Query | Healthy | Concerning |
|---|---|---|---|
| 1 | Created-on distribution across the calendar | ☐ | ☐ |
| 2 | Dwell between created and last modified | ☐ | ☐ |

Refused: ____ of 4. The sentence you may honestly use: ________________________________________

Evidence held (screenshots, exports, query output): ________________________________________
