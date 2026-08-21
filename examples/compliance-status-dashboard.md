# Worked example — compliance-status dashboard

The one artefact in this kit that enforces nothing. Fabricated throughout; no real supplier, part, customer
or organisation appears anywhere in this kit.

---

## Read this before you build it

Every other document here is about making records that hold up. This one is about looking at them, and it
carries a risk the others do not: **a dashboard makes whatever it displays feel true.** A green panel built
on decorative gates is worse than no panel at all, because it converts an unexamined assumption into a
number somebody will quote in a management review.

So the design rule for this dashboard is the opposite of the usual one. It is not built to show that things
are fine. Half of it exists to show you the ways your own records might be worthless, and those panels come
first, before anything anyone would want to put on a wall.

Do not build this until `docs/07-gate-self-test.md` gives you an honest answer about the system underneath
it.

## What it is

Six read-only panels over the tables in `docs/01-data-model.md`, the `Nonconformance` extension in
`examples/nonconformance-intake.md` (panels 2, 3 and 4 need its `Stage` and transition timestamps), and the
two optional tables in `examples/production-hard-gate-checklist.md` (panels 5 and 6). No new tables, no new
columns, no writes of any kind. Build it
as views, a report page, or a saved set of queries — whatever your platform gives you. The queries below are
written generically so they port.

**It must be read-only, and that is a structural requirement rather than a preference.** A dashboard with an
"update" button on it is a second write path into your tables, and a second write path is exactly what
`docs/02-hard-gate-pattern.md` §2 warns about. If someone can change a record from the dashboard, every gate
you built is now conditional on that page also implementing them.

---

## Panel 1 — Record creation over time

**The distribution check from `docs/02-hard-gate-pattern.md` §5, as a permanent panel rather than a
one-off.** This is first for a reason.

```
COUNT of records
GROUP BY date(Created_On)
FOR the last 90 days
SPLIT BY record type
```

Display as a simple column chart, one column per working day.

**Group by creation, not by gate passage.** `Inspected_On` and its equivalents are stamped when a gate passes,
which can be days after the record was created. Plotting those manufactures the very Friday-spike artefact
this panel exists to detect. If you also want the gate-passage view, make it a second series and label it as
such.

**What healthy looks like.** Columns across the working calendar, roughly tracking production volume, with
gaps on days you were shut.

**What tells you something is wrong.** A handful of tall columns and a lot of empty ones. A column the day
before an audit. A regular Friday spike. Any of those mean the records were created somewhere other than
where the work happened, and the rest of this dashboard is measuring that gap rather than your process.

This panel will be uncomfortable to look at for the first month and that is its job.

## Panel 2 — Dwell time

**The dwell check, likewise made permanent.**

```
FOR each record created in the last 90 days
COMPUTE  dwell = Modified_On − Created_On
DISPLAY  median, and count of records with dwell < 60 seconds
```

**Use created-to-last-modified, not created-to-closed**, matching `docs/02-hard-gate-pattern.md` §5 and
`docs/07-gate-self-test.md`. Measuring to closure silently drops every record that never closed, which is
exactly the population most likely to be unhealthy.

**The single number worth watching is the second one.** Records created and closed inside a minute were not
filled in while the work was being done. If that count is anything other than near zero, say so plainly in
the panel label rather than burying it in a percentile.

## Panel 3 — Open work, by age

The operational panel, and the first one anybody actually wants.

```
COUNT of Nonconformance
WHERE Stage <> Closed
GROUP BY Stage, and by age bucket (0–7 days, 8–30, 31–90, over 90)
```

Add the same for `Operation` records not at `SignedOff`, and for `Inspection` records not at `Closed`.

**The interesting bucket is `Contained` and over 30 days.** Those are non-conformances where the immediate
problem was handled and the analysis never happened — the failure mode
`examples/nonconformance-intake.md` Gate 2 exists to prevent, showing up as a queue rather than as a
missing record. The gate stops them closing; it cannot make anyone work them.

## Panel 4 — Corrective actions overdue

```
COUNT and LIST of CorrectiveAction
WHERE Completed_On is blank
  AND Due < today
GROUP BY Owner
ORDER BY Due ascending
```

**Group by owner and show it to owners.** An overdue list nobody sees by name is a list that grows.

**One honesty note.** This panel is only meaningful because
`examples/nonconformance-intake.md` Gate 3 refuses a due date in the past. Without that condition the panel
fills with actions that were overdue the moment they were created, everyone learns to ignore it, and you
have built a very good way of hiding the four that matter among forty that never did.

## Panel 5 — Separation of duties

```
COUNT of CorrectiveAction ca
JOIN Nonconformance nc ON ca.Nonconformance = nc.<primary key>
WHERE ca.Approved_By = nc.Raised_By
   OR ca.Approved_By = ca.Owner

COUNT of Operation op
WHERE op.SignedOff_By = op.Operator
   OR op.SignedOff_By IN (SELECT Checked_By FROM ProductionCheck WHERE Operation = op)
```

A lookup column stores the related row's identifier, not its human-readable reference, so join on whatever
your platform's lookup actually holds — the item ID on SharePoint — rather than on `NC_Reference`. If you have
built the Route A shadow columns, `ca.Raised_By_Email = ca.Approved_By_Email` is the same test on one row and
is cheaper.

**Both numbers should be zero, and the panel should say so on its face** — label it "should be zero", not
"self-approvals". A number with an expectation printed next to it gets questioned when it changes; a bare
number does not.

If either is non-zero, one of three things is true: the gate is not enforcing what you think, someone has
administrative rights and used them, or the records predate the gate. All three are worth knowing and only
the third is benign.

**And run one more query beside it, because a zero here can be produced by the gate failing rather than
working.** Count approved corrective actions whose `Approved_By_Email` is blank. That column is written by a
flow, so a persistent blank on an approved action means the flow's write was refused — which is what happens
when someone self-approved and the rule caught it too late to matter. Those rows are invisible to the
comparison above, because the column it compares was never populated. This number should also be zero, and it
is the one that tells you whether the first zero means anything.

## Panel 6 — Coverage against requirement

The panel that answers "are we doing the checks we said we would".

```
FOR each Operation at Stage in {Verified, SignedOff} in the last 90 days
COMPUTE  actual   = count of In-process ProductionCheck
         required = Required_Checks
DISPLAY  count where actual < required
```

On a correctly built system this should sit at zero, because Gate 2 in
`examples/production-hard-gate-checklist.md` refuses the transition. **That is precisely why it is worth
displaying.** A number that ought to stay at zero is the cheapest possible monitor on whether your gate is
still running — if it moves, something changed: a rule was disabled, a new form was added, an import path
appeared, or somebody has been editing the underlying table.

**Two Route A caveats, so you read the number correctly.** Where the coverage condition is flow-checked
rather than refused, a record can sit wrong for the seconds between the bad write and the revert, so a panel
refreshed at that moment shows a count that is real but transient — timestamp the refresh. And the panel
recomputes `actual` honestly while `required` is derived from operator-entered `Quantity_Made`, so it catches
an under-counting gate but not an understated quantity. It is a monitor on the gate, not on the work.

Treat a persistent non-zero value here as an incident, not as a metric.

---

## What a green dashboard does not mean

Worth printing next to the screen.

| A green panel says | It does not say |
|---|---|
| The records that exist are complete and internally consistent | That records exist for all the work that was done |
| Nobody closed a record without the evidence the gate required | That the evidence is any good |
| No corrective action is past its due date | That any of them worked |
| Causal analysis has three levels | That the third level is a real system cause rather than a restatement |
| Separation of duties held | That the approver read anything |

**The gap between those columns is the whole limit of what a records pattern can do.** Panels 1 and 2 are
the only two that speak to the left-hand column being about real work at all, which is why they are at the
top of the page rather than at the bottom where dashboards usually put their caveats.

## What to change for your shop

**Add a supplier or work-centre breakdown** to panels 3 and 4 if you have enough volume for it to mean
anything. Below roughly thirty non-conformances a quarter it will be noise.

**Add repeat-cause detection**, which is the highest-value addition to this page: group closed
non-conformances by `Cause_Level_3` and count. A third-level cause appearing four times means the corrective
action did not work, whatever the effectiveness check said. It is not a gate and it cannot be one — it is a
question for a person.

**Resist adding a monthly count of non-conformances raised as a performance measure.** If raising a record
makes a number look bad, the number goes down and nothing else changes. Count them by all means; do not
target them.

**Resist a percentage-complete gauge.** Records that exist are complete by construction here — the gates
guarantee it — so the gauge reads 100 per cent permanently and teaches everyone that the dashboard is
decorative.

## Refresh, access and export

**Refresh.** Daily is enough for panels 3 to 6. Panels 1 and 2 are historical and can refresh weekly. Live
refresh on every open is a good way to throttle your tenant for no benefit.

**Access.** Read-only to everyone whose work appears on it, including the people whose names are on panels 4
and 5. A dashboard about people, visible only to their managers, will be experienced as surveillance and
will change behaviour in the direction of fewer records.

**Export.** Give yourself a dated export of the underlying query results, not just the picture. When someone
asks in eleven months what the dashboard said in August, a screenshot is a screenshot and a dated export is
evidence.

## What this example does not do

It is not a quality management review, it does not compute cost of quality, it has no trend analysis or
statistical process control, and it makes no attempt at forecasting. It has no gates and enforces nothing —
it reads. Every claim it appears to make depends entirely on the system underneath it, which is why the
first thing to build is not this page but the honest answer in `docs/07-gate-self-test.md`.
