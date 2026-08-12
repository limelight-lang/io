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
