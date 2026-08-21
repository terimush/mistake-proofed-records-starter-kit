# Gates in the canvas app

Microsoft Lists as the data source, a canvas app as the way in. This is the right architecture, and it has
one trap that goes straight at the central claim of this kit.

---

## 1 · The trap, stated first

`docs/02-hard-gate-pattern.md` §2 ranks three places you can enforce a gate, and names the weakest as
"hide or disable the button". **A gate in a canvas app is that.** The app is one write path among several —
the list's own grid view, a second app somebody builds next year, a flow, an import, the API — and a rule
that lives in one of those paths is a suggestion to everybody using the others.

So the honest default is: **canvas-app gates are advisory**, exactly as the kit says.

**But that is a statement about the doors, not about the app.** It holds only while the other write paths
are open — and there is an arrangement that would close them, by giving users read access to the lists and
routing every write through a flow they cannot bypass. **It rests on an assumption nobody has confirmed and
the documentation does not settle**, so it is not part of the build. It is in the appendix at the end of this
document, as an option to test rather than a design to adopt.

Everything between here and that appendix assumes the ordinary case: users can write to the lists, the app is
one write path among several, and the gates that matter live below it.

## 2 · Which gate belongs in which layer

Four layers, in descending order of strength. **Put every gate as low as it will go.** A gate in the app that
could have been a validation formula is a gate you will lose the first time somebody builds a second app.

| Layer | Enforces | Put here | Do not put here |
|---|---|---|---|
| **Permissions** | Who can write at all | Who holds Contribute on each list; the closure lock at the end of the record's life | Anything conditional — permissions cannot see a record's contents |
| **List validation** | Conditions over one record's own columns | Required fields per stage; causal depth and distinctness; approver ≠ raiser via shadow columns, on the list where the approval happens; a flow-maintained flag or counter standing in for a cross-list condition | Anything reading a second list *directly*; anything needing a field's previous value; anything reading a lookup, a person column or a multiple-lines-of-text column |
| **Flow** | Cross-list conditions; elevated writes | "A corrective action exists with an owner"; the closure lock; system timestamps | Anything list validation can express — you would be moving a control upwards for no reason |
| **Canvas app** | The user's experience of the gate; composite checks before submission | Refusal messages; showing what is missing; guided stage sequence; pre-submission cross-list checks | The only copy of any rule that matters |

**Duplicate deliberately, and know which copy is the control.** The app should check what validation checks,
so the user is told what is missing before they submit rather than being handed a server error. That
duplication is good design. It becomes bad design the moment the app's copy is the only copy.

## 3 · What the app layer genuinely adds

Three things, and the first is better than it sounds.

**One list, one validation message.** SharePoint's list validation settings take a single formula and a
single message for the whole list. Whatever a user got wrong, they get that one sentence. This kit argues
repeatedly that a refusal naming what is missing saves a support call that "Validation failed" generates —
and the list layer structurally cannot do that. **The app is where per-gate refusal messages live**, and that
alone justifies building one.

**Refusal before submission instead of reversion afterwards.** The cross-list conditions that
`docs/09-microsoft-lists-build.md` hands to a flow-with-revert can be evaluated by the app *before* the write.
The user is told no, at the moment they act, with a reason. That is a materially better experience than a
record that silently reverts eight seconds later, and better experience is not cosmetic here — a gate people
understand is a gate that gets used rather than routed around.

**Capture at the moment of the work.** The app is where you make the record easy enough to create *while the
job is happening*, which is the entire point of `examples/production-hard-gate-checklist.md`. Barcode entry,
a two-field check screen, defaults from the previous check, a person picker that resolves properly.

**One warning about offline.** Canvas apps can cache and submit later, and this destroys the timestamp story
completely: `Checked_On` becomes the time of submission, so a day's checks all land in one burst — which is
precisely the retrospective-records failure the pattern exists to prevent, now automated. If you genuinely
need offline capture, store the device time *and* the server time in separate columns, and be honest that the
device clock is user-controllable and therefore not evidence.

## 4 · Delegation, or how a canvas gate becomes silently wrong

**This is the most important section in this document.** A gate that fails loudly is a nuisance. A gate that
passes wrongly is a liability, and delegation produces exactly that.

Canvas apps do not evaluate a query over your whole list. They pull a bounded page of rows and evaluate
locally when the query cannot be handed to the data source. **The default limit is 500 records, raisable to
2,000** in Settings → General → Data row limit. Microsoft states the problem plainly: "the query might return
incorrect results if the data source has more than 500 or 2,000 records."

**Against SharePoint specifically, several functions you would reach for first do not delegate.**
`CountRows`, `Sum`, `Search` and `In` are absent from the SharePoint delegable-operations table. `IsBlank` is
documented as **not delegable on text columns**. `Filter` and `=` do delegate across all types, and
`StartsWith` delegates on text.

Now put that against this kit's own gates.

**The coverage gate in `examples/production-hard-gate-checklist.md` counts in-process checks.** Written the
obvious way in a canvas app:

```
CountRows(Filter(ProductionChecks, Check_Type = "In-process"))
```

On a check list that has grown past 500 rows, that counts within the first 500 and returns a number that is
too small — or, once you filter to one operation, plausibly correct and arrived at by luck. Either way the
gate is now deciding on a partial view of the data, and **a coverage gate that silently under-counts is worse
than no coverage gate**, because it produces a confident pass on an operation that was never checked.

**The fix is mechanical.** Delegate the filter, then count the small result:

```
Set(varChecks,
    Filter(ProductionChecks, Operation_Reference = varOperation.Operation_Reference));
Set(varCount, CountRows(varChecks))
```

`=` on a text column delegates, so the filter runs on the server and returns only that operation's checks —
a handful of rows. Counting a handful locally is safe.

**`IsBlank` needs more care than the other substitutions, and the obvious fix is wrong.** Microsoft's
substitute is `Field = Blank()`, not `Field = ""`, and it warns in the same breath that the two are not
interchangeable: `Filter(…, CustomerId = Blank())` "will delegate to SharePoint. These formulas aren't
equivalent because the second formula won't treat the empty string ("") as empty." Worse for a gate, the
same note says the approach "works for the 'equals' operator ("=") but not the operator for 'not equals'
("<>")" — and a gate asserting a field is *present* is written with `<>`. So `Field <> Blank()` does not
delegate at all and silently falls back to the bounded page.

The conclusion is not a better formula. It is that **a presence check does not belong in the app**: put it in
list validation, where `NOT(ISBLANK([Field]))` is evaluated by the server over the actual record, and use the
app copy only to tell the user what is missing before they submit.

That is the quick fix. **§5 is the full ladder**, including the technique that removes the query altogether
and the list of popular "workarounds" that only silence the warning.

**Nobody is warned at run time.** The delegation warning is a yellow triangle in the formula editor at design
time. The app does not tell the user, the record does not record it, and the gate does not fail — it just
quietly decides on incomplete data.

**Which means your test results are not valid until you test at volume.** Every test in
`docs/06-validation-and-test-plan.md` passes on a list of twelve fabricated rows, and would pass identically
on an app whose gates silently break at row 501. Load a sandbox list to three thousand rows and run the suite
again — see T16. This is the single most likely way a build that passed its tests will be wrong in
production, and it will be wrong in the direction of letting things through.

## 5 · Working around delegation — and telling a fix from a trick

Search for "Power Apps delegation workaround" and most of what you find makes the yellow triangle disappear
without making the answer right. **In a quality-records system that is the worse outcome**, because it
converts a defect you could see into one you cannot. Sort every technique into one of two piles before you
use it.

### The pile that does not work

Each of these removes the warning and leaves the wrongness exactly where it was.

| Technique | What it actually does |
|---|---|
| Raise Data row limit to 2,000 | Moves the cliff from 500 to 2,000. Your check table will pass 2,000 in a year, and the app gets slower on the way. |
| `ClearCollect` the list, then filter the collection | The warning goes because collections are in memory — and the collection holds the first 500 rows. You have hidden the truncation, not removed it. |
| Wrap it in `With()`, or loop with `ForAll` over a non-delegable filter | Same. The bound was already applied before your logic ran. |
| Turn the delegation warning off in the editor | Literally hiding it. Somebody will do this at 5pm on a Friday. |
| Split into several lists to stay under the limit | Legitimate *only* if the split reflects something real, like a year boundary. Splitting to dodge a limit gives you the same problem plus a join you cannot delegate either. |

### The pile that works, in the order to try it

**1 · Keep the working set small.** The best answer is not cleverness, it is design. If the active list never
passes a couple of thousand rows, delegation never bites. Archive closed records to an archive list on a
schedule — which `docs/09-microsoft-lists-build.md` §1 already requires you to do for the permission-scope
ceiling. **Two problems, one job.** Build it early and most of this section stops mattering.

**2 · Make the query genuinely delegable.** Against SharePoint that means `Filter`, `LookUp`, `Sort`, `=`,
the relational operators on numbers and dates, and `StartsWith` on text. Then:

- `Search(...)` → `StartsWith(...)`, which delegates on text.
- `IsBlank(TextColumn)` → `TextColumn = Blank()` — **not** `= ""`, and read the warning in §4 before you use
  either. The negation does not delegate at all, so move presence checks down to list validation.
- `In` → an explicit `=` against a variable, or `StartsWith`.
- `Not(...)`, and the `!` operator, **do not delegate against SharePoint**. This catches people rewriting a
  validation rule into the app, because validation rules are full of `NOT(ISBLANK(...))`. Express the
  negation as a comparison, or leave the check on the list.
- Person columns: only email and display name delegate. Compare on the email, which is the shadow column you
  already have.
- `ID` is shown as a number in Power Apps but is text underneath on SharePoint, so only `=` delegates on it —
  the relational operators do not. Sorting is fine; comparing with `>` or `<` is not.

**3 · Never put a function on the column side.** This is the most common way a working query silently stops
delegating, and it survives code review because it looks harmless:

```
Filter(ProductionChecks, Upper(Operation_Reference) = varOp)     // NOT delegable
Filter(ProductionChecks, Operation_Reference = Upper(varOp))     // delegable
```

Transform the variable before the query, never the column inside it. Same for `Text()`, `Trim()`, `Left()`
and date arithmetic — do it to the value you are comparing against, and hand the data source a bare column.

**4 · Narrow on the server, aggregate on the small result.**

```
Set(varChecks, Filter(ProductionChecks, Operation_Reference = varOp.Operation_Reference));
Set(varCount, CountRows(varChecks))
```

The filter delegates, so the server returns only that operation's checks — a handful of rows — and counting a
handful locally is exact. **This is only safe when you can bound the result.** One operation has at most a few
dozen checks, so it is safe here. If a filter could legitimately return more than the row limit, you are back
where you started and should go to step 5.

**5 · Precompute the aggregate onto the record — the one worth the most.**

Keep a counter column on the parent: `Operation.InProcess_Check_Count`, incremented by the flow each time an
**in-process** check is created — not each time *a check* is created. The distinction is the whole value of
the column: a counter that also counts the first-off and the last-off reads three when one in-process check
was performed, and a coverage gate reading it passes an operation that was never checked. Name it for what it
counts and increment it on `Check_Type = "In-process"` only.

Decrement it when an in-process check is deleted, too. Nothing else will, and a counter that only goes up
turns a deleted check into permanent credit.

The gate then reads one number that lives on its own row. No query, no delegation, nothing to bound.

**And it does something better than fixing the app.** A count that lives on the record is a value list
validation can see, so the coverage gate moves out of the canvas app and into
`docs/09-microsoft-lists-build.md`'s validation layer — server-enforced, on every write path. That is this
document's own rule paying off: a gate you could not put low enough became one you could, by changing the
data rather than the query.

The cost is a denormalised number that can drift. Manage it deliberately:

- The flow is the **only** writer of the counter. Never the app, never a person.
- A scheduled job recomputes the counts and flags mismatches — counting the same rows the gate means, or the
  reconciliation agrees with a counter that is wrong in the same way.
- **Treat a mismatch as a signal rather than an error.** It means something wrote outside the flow, which is
  exactly what you want to hear about.

**6 · Do the aggregate in a flow instead.** Where a count genuinely has to span a large set, `Get items` with
an OData filter is the better place — but it has its own version of the same trap, so read this before you
rely on it. The default is **100 items**, raisable via Top Count to the **5,000** list view threshold, and
beyond that the action fails outright. Worse, Microsoft documents that on a list over 5,000 items a filter
query "may observe that no records are returned if there are no items matching the filter query in the first
5000 items" unless pagination is enabled on the action. **A silent empty result is the same class of failure
as a silent undercount**, so enable pagination and test the flow at volume too, not just the app.

For a true server-side count without pulling rows, **Send an HTTP request to SharePoint** — part of the same
standard connector — can call the REST endpoint directly. It is more to maintain and it is the right answer
only when steps 1 to 5 have genuinely failed.

### How to know which pile you were in

Run T16 in `docs/06-validation-and-test-plan.md`: load the list to three thousand rows and compare the gate's
answers against the small-data run. **If the numbers match, you fixed it. If they do not, you hid it.** There
is no third outcome and no amount of reasoning about the formula substitutes for the test.

And if a gate cannot be made delegable by any of the six, **move it down a layer rather than accepting it**.
An unreliable gate in the app is worse than an honest flow-check-and-revert, because the app one looks like
enforcement.

## Appendix · The read-only-list arrangement — untested, and not part of the build

**The warning comes before the design, deliberately.** What follows would make a canvas-app gate genuinely
enforceable rather than advisory, by closing every write path except the app. Nothing else in this kit
depends on it. No worked example uses it. The auditor wording in `docs/09-microsoft-lists-build.md` §6 does
not assume it. It is written down because it is the obvious next thought for anyone who has read §1, and it
is better to set out what would have to be true than to leave a reader to rediscover it badly.

> ### The assumption, and why this is an appendix
>
> The whole arrangement rests on one claim: that a user holding only **Read** on a list can still run a flow
> that writes to it, because the flow executes under the owner's connection. **Nobody has confirmed that, and
> the documentation does not settle it either way.** Read both sides before you decide.
>
> **Against.** Microsoft's guidance for Microsoft Lists flows, under "Managing run-only users", says plainly:
> *"In order to run flows, users must have Edit Permissions on the list."* If that applies here, the
> arrangement is dead.
>
> **For, but narrowly.** The same page opens by scoping itself: *"Users' ability to run flows depends
> entirely on the trigger, specifically: For a selected item / For a selected file."* Those are the two
> manual list triggers. A flow called from a canvas app uses neither, so the Edit-permission sentence is
> written about a case that is not this one — which is an argument that it may not apply, not evidence that
> it does not.
>
> **And a quotation people misuse in this argument.** Microsoft's flow-sharing guidance says a run-only user
> "might not need any special role **in Dataverse** — they're pressing a button and the flow uses the owner's
> privilege". That sentence is about Dataverse security roles. It is not about SharePoint list permissions
> and it settles nothing here.
>
> **Settle it before you design around it.** Build a sandbox list, grant a test account Read and nothing
> more, and try to write through the app. Twenty minutes. If it fails, everything below is unavailable to you
> and your canvas gates are advisory, exactly as §1 says. If it works, open an issue — this appendix becomes
> a section, and the kit gets better.

Close every door but one, and the gate on that door stops being a suggestion.

**The shape.** Users get **Read** on the list, not Contribute. Writes happen through a Power Automate flow
that the canvas app calls. The flow is shared **run-only** and uses the **owner's embedded connection**
rather than "provided by run-only user", so the write executes with the flow owner's permission and not the
user's.

If the test above passes, the user then genuinely cannot write to the list. Not from the grid, not
from a second app, not from the API — they lack the permission, and no cleverness recovers it. The app
becomes the only write path, so the gate in the app is the only door.

**This is worth doing, and it costs four things you should decide about deliberately.**

**Attribution moves, and you lose the platform's own answer.** Every write now shows the flow owner in
`Created By` and `Modified By`. The system's audit fields stop telling you who did anything. You must carry
the real user in your own columns, written by the flow from the app's `User()` context — which is precisely
what the shadow email columns in `docs/09-microsoft-lists-build.md` §2 already exist for. Say this out loud
when you describe the system, because "Modified By shows one service account for every record" is the first
thing an auditor will notice and the answer needs to be ready.

**The flow owner becomes a single point of trust.** Whoever owns that connection can write anything, past any
gate, with no record that it was them rather than a user. Name that account, make it one nobody uses
interactively, and do not let it be the personal account of someone who might leave.

**The permission assumption, again, because it decides everything.** See the top of this appendix. If a
genuinely read-only account cannot run the flow, the arrangement collapses entirely and nothing recovers it —
you are back to advisory app gates over a list users can write to, which is what the body of this document
assumes anyway.

**Read-only means read-only.** Nobody can use Quick Edit for legitimate bulk work either. On a list with a
lot of routine data entry that is a real cost, and it is the usual reason this pattern gets abandoned three
weeks in.

**And the permanent caveat:** a site owner or tenant admin can still write directly. That is true on every
platform in this kit and it is why `CONTRIBUTING.md` forbids describing any record as beyond alteration.

### What you would tell an auditor, if you built it

Only if the test above passed *and* you built it. Layered, because the architecture is layered, and each
clause is separately checkable.

> "Writes to the quality records go through one application. Users hold read access to the underlying lists
> and cannot write to them directly — we tested that with a user account. The conditions the application
> enforces are also enforced on the lists themselves, so a record cannot reach a stage without the evidence
> that stage requires regardless of how it was written. Conditions that span records are checked before
> submission and again by an automated process that reverts and logs an invalid change. Because writes are
> made by a service identity, the platform's own created-by and modified-by fields show that identity; the
> acting user is recorded in dedicated columns by the application, and those are the fields to read."

The last sentence is the one people leave out, and it is the one that turns a surprise into a design
decision. Say it before you are asked.

**If you did not build this, that wording is not yours to use.** Use
`docs/09-microsoft-lists-build.md` §6, which describes the ordinary case and is what the rest of this kit is
written against.

## What this document does not do

It does not make a canvas app into a control by itself. On the build this kit describes, a canvas gate is
the weakest of the three enforcement points and you should describe it that way. The appendix sets out what
would have to be true for that to change, and why it is an appendix rather than a section.
It does not cover app performance, ALM, environments or solution packaging, none of which this kit has
anything useful to say about. And it assumes standard licensing: a canvas app over SharePoint with standard
connectors, no premium tier. Confirm your own entitlements rather than taking a document's word for it.

Nothing here has been tested on your tenant. Build it in a sandbox, with a genuinely read-only test account
and a list big enough to break delegation, and prove each claim yourself.
