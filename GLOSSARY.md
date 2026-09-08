# Glossary

Every term this kit uses that it does not stop to explain. Written for someone who runs a quality system, not
someone who builds software. If a page uses a word you do not recognise, it should be here; if it is not,
that is a fault in the kit and worth an issue.

Terms are grouped by what you are doing when you meet them, not alphabetically, because the words arrive in
roughly this order.

---

## The idea

**Hard gate.** A rule the software enforces rather than a policy people are asked to remember. A record cannot
advance to the next stage until the evidence that stage requires actually exists. The whole kit is about where
to put these so they cannot be walked around.

**Decorative gate.** A rule that looks like a gate and is not, because it only exists in the screen people
normally use. Anyone who reaches the data another way — a bulk edit, an import, a different app — never meets
it. `docs/02-hard-gate-pattern.md` §2 is about telling the two apart.

**Low-code.** Software you configure rather than program: you fill in settings and drag steps around instead
of writing source code. It does not mean no work, and it does not mean no thinking about logic. It means you
will not be writing or compiling code, and you will not need a developer to change a rule later.

---

## The platform

**Tenant.** Your organisation's own Microsoft 365 world — your users, your files, your settings, walled off
from every other customer's. "On your own tenant" means "using your company's real subscription", as opposed
to reading about it.

**Microsoft Lists, and how it relates to SharePoint.** Microsoft Lists is the app you open. The lists
themselves are SharePoint lists, and every setting this kit asks you to change lives in the list's settings
inside SharePoint. Two names, one thing. Build the lists on a **team site**, not as personal lists — the
settings the kit needs do not exist on personal lists.

**Team site.** A shared SharePoint site belonging to a group of people, as opposed to your personal storage.
Someone with administrator rights may need to create one for you; that is usually the only thing in this kit
you cannot do yourself.

**Power Apps / canvas app.** Power Apps is Microsoft's tool for building screens over your data. A **canvas
app** is one kind — you lay the screen out yourself, like a slide. In this kit "the app" always means a canvas
app, and it is the screen an inspector types into. It is optional: the gates hold without it, and what it adds
is a better experience and a clearer refusal message. `docs/10-canvas-app-gates.md` is entirely about it.

**Front end.** The screen a person actually uses. Could be the built-in list form, could be a canvas app.

**Power Automate / flow.** Power Automate is the tool for building automations. A **flow** is one automation:
*when this happens, do these things.* This kit uses flows to fill in columns a validation formula cannot reach
by itself, to stamp timestamps, and to lock a record at closure.

**Trigger.** The event that starts a flow — "when an item is created or modified on this list". One trigger
binds to **one list**, which is why a job described as running on three lists is really three flows.

**Trigger condition.** An extra test on the trigger, so the flow only runs on the changes you care about.
Essential here: a flow that writes to the record that started it will start itself again unless you exclude
its own writes.

⚠ Note that "trigger" is also used in ordinary English in this kit — "the one trigger for moving to
Dataverse" means *the one reason*, not a flow trigger. Context tells you which.

**Route A / Route B.** The kit's two platform choices. **Route A** is Microsoft Lists plus Power Apps plus
Power Automate, on the licence you probably already have. **Route B** is Dataverse, which does more and costs
a premium licence. `docs/03-implementation-guide.md` chooses between them; almost everyone should start on
Route A.

**Sandbox.** A separate practice copy — a second site, or a second set of lists — where you can build and
break things without touching anything real. Costs nothing extra on Route A. Build here first, always.

**Service identity / service account.** A user account that exists for flows to run as, rather than for a
person to log in with. Flows run "as" somebody, and it should not be you personally — when you leave, or your
password changes, every flow owned by your account stops.

---

## Building the rules

**Column, and its display name.** A field on a list. Its **display name** is the label you see. Validation
formulas refer to columns by display name, exactly as typed, so a column called `Quantity_Checked` must be
written that way in the formula — capitals, underscore and all.

**Person column.** A column type that holds a real directory identity rather than typed text: you pick a
colleague and the system stores *who they are*, not what you typed. This matters because a typed name is not
attribution — two people can type the same name, and anyone can type anyone's. ⚠ Validation formulas **cannot
read person columns**, which is the single fact that shapes most of the Lists build.

**Lookup column.** A column that points at a row in another list. Also unreadable by validation formulas.

**Single line of text vs multiple lines of text.** Two different column types. Validation formulas can read
the first and **cannot read the second at all** — so any gate meaning "this narrative field is filled in" has
to use a single line of text, or be checked by a flow instead.

**Choice column.** A column with a fixed set of options. ⚠ Two settings matter: a **default value** means
every record is born with that answer already given, so a gate requiring "an answer" is satisfied by nobody
having decided anything; and **fill-in choices** let a user type a value outside the list, which can sidestep
a rule written against a specific word.

**Validation formula, and what "on the server" means.** A rule attached to the list itself, which SharePoint
evaluates **when the item is saved** — on Microsoft's servers, not in the screen you typed into. This is the
central mechanism of the whole kit. Because the check happens at the point the data is written rather than in
the form, it holds no matter which route the data came in by: the form, the grid, an import, another app, the
API. ⚠ A list holds exactly **one** formula and **one** message. Adding a second replaces the first.

**The grid / grid view.** The spreadsheet-style view of a list, where you type straight into cells without
opening a form. It matters twice over: it is the ordinary way people bulk-edit, and it is the honest way to
test whether your rule is really on the record — because it bypasses the form entirely. When the kit says
"test from the grid, not the form", this is why.

**Machinery column (shadow column, frozen copy, flow-maintained flag).** An extra hidden column that exists
only so a validation formula can reach something it otherwise cannot: a plain-text copy of a person column
(`Inspector_Email`), a copy of a linked record's key (`NC_Reference_Text`), or a yes/no a flow maintains to
stand in for a cross-list condition (`Closure_Ready`). ⚠ They are written by a flow **after** the save, so
they are briefly blank on every save — which is why `docs/01-data-model.md` insists no rule may be satisfied
by one of them being *absent*.

**Fails open / fails closed.** A rule **fails closed** when, if something has gone wrong or is missing, it
refuses — the safe direction. It **fails open** when the same conditions make it silently permit. A gate that
fails open is worse than no gate, because you believe you have a control and you do not.

**Patch.** The Power Apps instruction for writing to a record. It appears in this kit only because it lets a
canvas app write two columns in one operation, which collapses the two-save sequence the list route needs.

---

## Scale, permissions and limits

**Delegation, and the 500-row limit.** A canvas app does not read your whole list. It asks the list to do the
work where it can — that is *delegation* — and where it cannot, it pulls a page of rows and works on those.
That page is **500 rows by default and 2,000 at most**. ⚠ So a count or a total written the wrong way is
correct while you are testing with fifty rows and silently wrong once you pass five hundred: it counts the
page, not the list. Power Apps warns you with a blue underline while you are building, and the warning is easy
to dismiss. `docs/10-canvas-app-gates.md` §4 lists which operations delegate and what to do instead.

**Unique permission scope.** Normally every item on a list inherits the list's permissions, and SharePoint
stores that once. The moment you give one item its own permissions — which is exactly what locking a closed
record does — that item gets a **unique permission scope**, stored separately. Microsoft recommends no more
than **5,000** per list and hard-limits it at 50,000. So every record you lock spends one, permanently, and a
busy list needs an archive job before it runs out. Do the arithmetic in `docs/09-microsoft-lists-build.md` §5
against your own volumes.

**Requests, and the daily allowance.** Every action inside a flow counts against a per-user daily allowance —
**6,000 requests per user per 24 hours** on the licence most people have. ⚠ Background flows always draw on
the allowance of the flow's **owner**, not the person whose action started them. So all nine flows in this
build spend one person's budget however many inspectors are working.

**List view threshold.** SharePoint's 5,000-item limit on what a single view or query can handle at once. It
is why the kit keeps telling you to archive.

---

## The compliance words

**ISO 9001:2015.** The general quality-management standard most manufacturers are certified to.
`docs/04-compliance-mapping.md` maps its clauses to what this kit helps you evidence, and is explicit about
where the mapping stops.

**CMMC / NIST SP 800-171.** United States defence-supply-chain cybersecurity requirements. This kit touches a
small corner of them and `docs/04-compliance-mapping.md` says which corner. Do not read it as broader than it
is.

**Non-conformance (NC).** A record that something did not meet requirement. **Corrective action (CA).** A
record of what was done about it and whether that worked. **Disposition.** The decision about what happens to
the non-conforming material — use it, rework it, scrap it, send it back.

**Root cause / causal levels.** The chain of *why* behind a problem. This kit asks for three levels, each
answering why the one above it happened, and refuses a partial chain — that refusal is the single most
valuable condition in the kit.
