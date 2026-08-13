# Postmortem

Mistakes that cost time, broke something that worked, or sent the work in
a wrong direction. Symptom, root cause, and what changed so it does not
recur.

## 2026-08-12 — Every design document stated a protocol without its order

**What happened.** Seven design documents were written and each was given
one Critic round before its step closed. All seven came back with at least
one high-severity finding, and five of them were the same defect: a
protocol described by what happens, with the order of steps and the
atomicity of each left unstated. Every one of the five produced silent
corruption rather than a crash.

- A context switch narrowed by a live-register mask skipped callee-saved
  registers holding an ancestor frame's values; another unit overwrote
  them and the ancestor resumed with foreign data.
- A waker validated a handle and then wrote, so a slot released between
  the two took the writes into a stranger's record.
- `Parked` was published by the unit rather than by the worker, announcing
  a machine context that was still being saved.
- Cancellation went through the ordinary wake path, whose AND branch
  decrements a counter and returns without waking, so a cancelled unit
  parked on an AND wait never resumed.
- The deadlock search read what units waited on and ignored what had
  already fired, and reported a set that was waiting only on the kernel.

**Root cause.** A design document reads as finished when every mechanism
is named, and none of these documents was missing a mechanism. What each
was missing was the sequence: which store happens before which, which pair
must be one atomic operation, and which thread performs each step. Prose
about a concurrent protocol hides that omission, because a reader supplies
a plausible order while reading and never notices they supplied it.

**What changed.** Every protocol in this repository is now written as an
ordered list with the actor of each step named, and each state transition
says whether it is a compare-and-swap and what it pairs with. The parking
protocol in `design/execution.md` is the model: five numbered steps, a
transition table with a "by" column, and a sentence for each ordering that
says what breaks if it is reversed. A protocol described any other way is
not finished, however complete its list of parts.

## 2026-08-13 — A fix that answers a Critic round is not safer than the draft it replaces

**What happened.** Three steps closed on the same shape. S5.3 took two
Critic rounds, and the second found a *critical* defect created by the
first round's own fixes: the winner field existed under OR alone, so a
cancel had nothing to claim, and a retired entry firing later drove an AND
counter to zero and enqueued the unit twice. S5.4 took two rounds and the
second found two criticals, both in the rewrite. S5.5 took two rounds and
the second found five defects, one of them introduced by the fix for
finding 4 of the first — a new edge that credited a store to the unit
where the whole point of the step it drew is that the worker performs it.

**Root cause.** A fix is written against one finding and read against
nothing. The draft it replaces was at least written against the whole
corpus; the patch is written against a paragraph, in a context already
crowded with the finding it answers, and the author's attention is on
having answered rather than on what the answer now claims. The three
defects above are all of one kind: the fix asserted something true of the
place it was patching and false one document over.

**What changed.** Two things, and the second is the one that saved time.

A step marked with the Critic role gets **two rounds by default**, and the
second round's brief is the fixes rather than the document. It is not a
formality: it found something every time it ran today.

And when the second round keeps hitting the same device, the device is the
defect. `dev/ARCHITECTURE.md` was organised three times as a strict
ordering of levels where a module may name only levels below it, and three
times a Critic showed the system is not layered that way — the detector's
arming rule runs on the park path, the detector reads a table the API
crate owns, the reactor publishes into the detector's channel. The fourth
attempt dropped the ordering instead of defending it and named three kinds
of edge instead of one: a call, an installed callable, and a read of a
root. Patching a device that has failed twice costs more than replacing
it, and the signal that it is the device rather than the row is that the
findings keep landing in different rows.
