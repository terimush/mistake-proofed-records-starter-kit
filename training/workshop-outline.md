# Workshop outline — half day, hands-on

A 3.5-hour workshop for a small-manufacturer audience: an MEP centre client group, a manufacturing
association, a professional-society section. Attendees leave having built a working gated record.

**State this plainly when the session is booked.** The workshop cannot be delivered credibly without
attendees having their own sandbox. A demonstration they watch is a talk, and there is already a talk in
`training/practitioner-talk.md`. If the host cannot get attendees into a tenant, deliver the talk instead and
say so honestly rather than running a hands-on session that nobody can follow along in.

---

## Audience and prerequisites

**Who it is for.** Quality managers, quality engineers, and owner-operators at small and mid-sized
manufacturers. Assume they understand their own process thoroughly and have never written code. Twelve to
twenty attendees is the workable range; past that, the exercise section stops being supportable by one
facilitator.

**What attendees must bring:**

- **A laptop**, not a tablet. The forms designer is not usable on a small screen.
- **A sandbox tenant or developer environment they can create in.** Not their production tenant. Confirm this
  with each attendee a week in advance, in writing, and confirm it again the day before.
- **Two test identities in that tenant.** This is the prerequisite that gets skipped and it is the one that
  breaks the session. The self-approval demonstration — the single most persuasive minute in the workshop —
  is impossible with one identity, because there is nobody for the gate to compare the approver against.
- **A rough description of one process they would gate**, written down before they arrive. Incoming
  inspection if they have nothing better.

**What the host must provide.** Reliable wifi that permits the platform's traffic, a projector, and a room
where people can turn to each other. Guest wifi that blocks the tenant has ended more sessions than any
technical problem in the material.

## Learning outcomes

By the end, attendees can:

- **State what a hard gate is** and explain to a colleague why enforcement on the data differs from a
  disabled button.
- **Build the three-table model** in a sandbox and connect a form to it.
- **Write a gate as an explicit condition** over their own process's fields, rather than as a policy sentence.
- **Demonstrate a gate refusing a transition**, and read the refusal message they wrote — and say why a list
  gives them one message for the whole list while the app gives them one per gate.
- **Say which of the four cheats their own build blocks**, and therefore describe their build accurately to
  an auditor.
- **Run the distribution check and the dwell check** against a system they already have, and interpret what
  comes back.
- **Identify the three mistakes that make a gate decorative** in a system they already run.

Note what is not on that list. Nobody leaves with a deployable quality system, and nobody leaves compliant
with anything. Say so at the start.

## Timed plan

**0:00 – 0:15 · Opening and the problem.** Small firms carry large-firm documentation obligations without
large-firm budgets. The failure mode is not missing records; it is retrospective records — assembled the week
before an audit, hard to defend, and produced by a process nobody was actually being guided by. Ask the room
directly how many have rebuilt records for an audit. Most hands go up. That is the whole motivation and it
takes fifteen minutes, not forty.

**0:15 – 0:35 · The idea, and where enforcement lives.** A record cannot advance to the next stage until the
evidence that stage requires actually exists. Then the part that matters: the three places you can enforce a
gate — hide the button, validate on save, enforce on the data at the transition — and why only the third
survives contact with an auditor. Draw the stage diagram on the board. This is `docs/02-hard-gate-pattern.md`
§1 and §2, delivered as talk.

**0:35 – 0:55 · Live demonstration.** See the sequence below. Do not let it run long; the room needs to be
building by the hour mark.

**0:55 – 1:05 · Break.**

**1:05 – 1:35 · Build the tables.** Everyone creates the three tables from `docs/01-data-model.md` in their
own sandbox. Walk the room. The predictable error is a text column where a person column belongs — catch it
now, because it quietly undermines two of the three gates and is tedious to unpick later.

**1:35 – 2:15 · Build gate 1 and the transition rule.** One rule, one gate — `Draft` → `Inspected`. Stamp the
timestamp on pass. Refuse with a message that names what is missing. Cover the trigger-condition point
explicitly before anyone saves a rule that writes to the record that triggered it — the infinite trigger loop
in `docs/03-implementation-guide.md` is a real problem in a shared tenant and it is easier to prevent than to
unwind.

**Say the one-formula rule out loud while they are typing it.** A SharePoint list holds a single validation
formula. Attendees who go home and add gate 2 underneath gate 1 will replace it, and the test they run
afterwards still refuses, so they will believe they have both. Ninety seconds now saves that.

**2:15 – 2:25 · Break.**

**2:25 – 3:00 · The exercise.** Attendees work on their own process rather than on incoming inspection. See
below.

**3:00 – 3:20 · Break your own gate.** Every attendee tries, against their own build, the two cheats their
build can actually answer — setting the status directly from the grid, and editing a field after its stage
has passed — and writes down what happened. They have built gate 1 only, so the causal-depth and
self-approval cheats from build step 7 have nothing to run against; demonstrate those two yourself on the
prepared sandbox instead, and have attendees predict the result before you run it. This is the most valuable
twenty minutes of the session and it should not be cut for time. Cut the exercise instead.

**3:20 – 3:30 · Close.** What to do on Monday, the two analytical checks, where the kit lives, and the
boundary — this is not a quality system and it is not certification.

## The live demonstration sequence

Run this on a prepared sandbox with the sample data already loaded. Do not build it live; build it before the
room arrives and rehearse the failure.

1. **Create a record and try to advance it empty.** The gate refuses and names what is missing. Read the
   refusal message aloud — it is the difference between a support call and no support call.
2. **Fill the fields properly and advance.** Point at the timestamp and say it was set by the platform, not
   typed. Then say what *has not* happened: on this tier the fields the gate required are not locked, and you
   will show that deliberately in step 5. Do not claim a lock you cannot demonstrate — the room will ask.
3. **Try to close with one causal level.** Refused. Three levels required.
4. **Try to approve your own corrective action.** Refused, because the approver matches the raiser. This is
   the moment the room leans in, and it is why the second test identity is a hard prerequisite.
5. **Then try to break it deliberately.** This is the most valuable moment in the workshop. Open the
   underlying list in grid view and set the stage straight to `Closed` on a half-empty record. Do it twice:
   once against a build whose rule is in the form, where it works, and once against a build whose rule is in
   list validation settings, where the server refuses it. Let the first one sit for a moment before you show
   the second.

The point of step 5 is not that the free platform is bad. It is that where you put the rule decides
everything, and that the difference is completely invisible from the form. Follow it with the ceiling stated
plainly: even with validation in place, Lists has no column-level permission, so a field can be altered after
its stage has passed until the record is locked at closure. That, and not the gate conditions, is what a paid
step up buys.

## The exercise

Attendees take the process they brought and write **one gate** as an explicit condition — a `REQUIRE` block
naming every field, what gets stamped on pass, what locks, and the refusal message a user would see.

Then, in pairs, each attendee tries to defeat the other's gate on paper. "What would your second shift do to
get past that?" This finds more real problems in ten minutes than the facilitator can find in an hour, and it
teaches the habit the kit actually wants: assume the gate will be routed around, and design so that routing
around it is harder than complying.

Ambitious attendees who finish early should build their gate in their sandbox. Most will not get that far,
and that is fine.

## Facilitator notes — what usually goes wrong

**Tenant permissions.** The most common blocker by a distance. Someone's IT has locked environment creation,
or the connector is blocked, or an admin consent prompt appears that nobody in the room can approve. Nothing
you can do on the day. Mitigate by confirming environment access in writing a week ahead, and have a paper
version of the exercise ready so blocked attendees still leave with a written gate.

**Attendees without a second identity.** Guaranteed to happen despite the confirmation. Pair them with
someone who has two and have them run the self-approval test together. Do not skip that test.

**Someone whose ERP already does this.** Give the honest answer: if your ERP already enforces the transitions
and stamps the timestamps, use the ERP. This kit exists for people whose ERP does not, or who cannot afford
the module that would. Then hand them the two analytical checks, because an ERP module that nominally enforces
gates and shows a hundred records created in one afternoon is worth knowing about. Do not compete with a tool
that is already working.

**Someone who wants to deploy it on Monday.** Redirect to a sandbox and one process, and say why. A gated
record rolled out across a shop in a week gets routed around within a fortnight.

**Time pressure.** The build sections will overrun. Protect the "break your own gate" slot at 3:00 by
shortening the exercise — a gate somebody has attacked is worth more than a second gate they have only
written.

## What to send attendees home with

- **`training/one-page-handout.md`**, printed, one per chair before anyone sits down.
- **The repository link**, `https://github.com/terimush/hard-gate-starter-kit`, on the last slide and on the handout.
- **Their own written gate**, on paper, from the exercise.
- **The two analytical checks**, framed as the one thing they can do on Monday without building anything.
- **APEX Accelerators and MEP centres** — free, publicly funded, and better first calls than any vendor if
  they are unsure what their contracts actually require.
- **The honest boundary**, said out loud at the end rather than left in a document: this is not validated
  software, it is not a quality management system, and it does not make anyone compliant with anything.
