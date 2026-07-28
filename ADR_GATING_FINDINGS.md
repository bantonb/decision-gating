# Decision-Gating: A Narrow, Tested Claim About Institutional Memory in Long-Lived AI-Assisted Work

## Why this exists

Most personal AI-automation projects accumulate an implicit problem: as work spans multiple
sessions, multiple projects, and enough time for context to be forgotten, an agent (or a human)
can silently redo, contradict, or drift from a decision that was already deliberately made. This
project — a personal automation layer built around one architectural principle — set out to test
whether a small, cheap mechanism could catch that specific failure mode, and to be honest about
what it did and didn't prove along the way.

The core principle behind it: **don't let a shared fact have two independent, silently-diverging
implementations.** Every architecturally significant decision gets one canonical record. New work
is checked against that record before it's built, not after.

This document reports what was actually tested, on a real codebase, with real numbers — including
the things that didn't work.

## What this is *not* claiming

Before the results: three things this project explicitly does not claim credit for, because
better-resourced, more mature work already covers this ground.

- **Not a memory system.** Products like Mem0, Zep, and Letta solve fuzzy, temporal, large-scale
  conversational memory — "what did this user tell me, and when." That's a different, harder
  problem than what's tested here, and this project makes no attempt to compete with it.
- **Not a bug-finding framework.** An early experiment compared a parallel, decomposed
  multi-agent review against a single continuous session reviewing the same code for the same
  bugs. They converged almost completely: 10 of 12 independently-verified claims matched on
  verdict label exactly, one of the two label mismatches was later resolved with a manual
  tie-breaker read, and a third case (same label on both sides) turned out, on closer reading, to
  describe a meaningfully different underlying mechanism and severity — worth flagging as a
  near-match rather than a clean one. At the scale tested, a strong single session already
  tracks cross-file interactions about as well as a decomposed one. That's reported here as a
  genuine, disclosed null result, not hidden.
- **Not general engineering discipline.** Good testing practice, TDD, and structured workflows
  (e.g. the Superpowers skill library) already solve that, well, and this project doesn't
  reinvent it.

## The one thing that was actually tested and held up

**Claim:** given a new code-change task, an agent instructed to check an existing decision log
before implementing will correctly recognize when the task conflicts with a previously-documented
decision — and do so before writing any code — while an otherwise-identical agent without that
instruction will not.

This was tested twice, on two structurally unrelated real bugs found in a production-adjacent
routing/clustering pipeline, with genuine session isolation (separate temp directories containing
only the relevant files; the decision log physically absent from the "no-check" condition's
working tree; every tool call logged).

**Methodology transparency, stated plainly rather than left implicit:** All tests used Claude
Sonnet 4.6. The decision log contained only the ADRs directly relevant to what was being tested
(two entries total, added one at a time as each scenario was designed) — this was not a test of
retrieval at scale, and a log with many more entries might behave differently; that's untested.
The same process that had already analyzed the underlying bugs also wrote the decision records
and later constructed the near-miss test cases against them. That's a real limitation: whoever
writes the test material knows what it's testing for, and this wasn't an independently-authored
or blind test set. It's disclosed here rather than presented as more objective than it is.

### Scenario 1 — day-indexing convention

Three different fields carry a day-assignment through the pipeline, with three different origins,
names, and indexing bases (0-indexed in one stage, 1-indexed by another, silently diverging by the
time a later stage reads it). This caused a real, previously-found bug: wrong appointment dates
whenever an intermediate reordering stage ran.

| | No-check condition | Gated condition |
|---|---|---|
| Caught the conflict before coding? | No | Yes — cited the decision record before reading any source |
| Tokens used | ~46,400 | ~37,600 |
| Outcome | Technically correct fix proposed, decision-blind | Correct fix, flagged as a documented-decision conflict, update requirement noted |

### Scenario 2 — dual path management

A separate, unrelated bug: two pipeline stages each maintain their own path expression for the
same handoff file, with nothing asserting they resolve to the same location — a bug class that
had already caused a real silent stale-data failure.

| | No-check condition | Gated condition |
|---|---|---|
| Caught the conflict before coding? | No | Yes — cited the decision record before reading any source |
| Tokens used | ~39,600 | ~30,300 (24% fewer, despite 20 tool calls vs. 7) |
| Outcome | Correct fix proposed, missed a related latent bug, no decision-log update noted | Correct fix, explicitly required an atomic multi-file update per the decision record's own procedure, flagged the update requirement |

The tool-call-vs-token direction here is counterintuitive and worth explaining rather than leaving
as a bare number: the gated agent made more calls but used fewer tokens because those calls were
targeted greps guided directly by the decision record's own description of where to look, versus
the ungated agent's broader, less-directed file reads. More calls, less wasted reading.

### What's consistent across both

- The gated condition caught the conflict every time; the ungated condition never did, in either
  run.
- The gated condition was **cheaper** both times, not just as effective — despite doing an extra
  check step, it used targeted lookups guided by the decision record instead of broad exploratory
  reads.

### Honest caveats — read this part too

- **This is n=2.** Two runs, two conventions, cleanly isolated methodology. That's enough to
  report as "shown to work in these specific cases," not enough to claim as generally proven.
- **One bonus finding in scenario 2 (a related latent bug the gated agent also caught) was likely
  primed by the decision record's own content**, which pointed at the exact file pair involved.
  It would not be honest to claim the gated agent found this "independently" — the ungated agent
  may well have found it too, given more exploration time. This is called out explicitly rather
  than folded into the headline result.
- An earlier version of this same experiment had a real methodological hole, identified before
  fully trusting the result: the "no-check" condition's session had passive read access to the
  entire repository, including the decision log, even though it wasn't instructed to look there.
  Roughly half its tool calls in that run weren't individually accounted for, so it could not be
  ruled out from the data that it had quietly found the log anyway — this wasn't a theoretical
  worry, it was an unresolved possibility in the actual run. The scenarios reported above are the
  re-run, with the decision log physically absent from the no-check condition's directory and
  every tool call logged, closing that gap before trusting the result.
- A separate, earlier attempt to show this same mechanism catches problems "a generally competent
  engineer wouldn't" did not succeed — an isolated baseline condition performed comparably via
  general reasoning alone. That result is not overwritten by the results above; it's a different,
  broader claim that remains unproven, and is reported as such. Put plainly: the value shown here
  isn't that the gate catches conflicts a careful single session would miss on its own — the
  earlier result suggests it might not. What the two scenarios above actually show is that the
  gate reaches the correct, conflict-aware answer at a lower token cost, and with an explicit,
  actionable flag raised before any code is written, rather than a fix quietly landing that
  happens to also be correct. That's a narrower and more modest claim than "finds more bugs," and
  it's the one this project can actually back with evidence.

### Precision check — does it also correctly say "no conflict"?

Everything above tests recall: does the gate catch a real conflict. It says nothing about
precision: does the gate stay quiet when a task merely looks similar to a documented decision but
doesn't actually violate it. An ungated agent can't produce a false positive by construction (it
never checks), so this was tested with the gated condition only, against two deliberately
constructed near-miss tasks:

- **Against the day-indexing decision:** adding a debug log statement that prints the current
  value of the affected field, with no logic change.
- **Against the path-management decision:** adding an opt-in, local-testing-only CLI flag that
  touches the same area of code without altering the existing production path logic.

Both were correctly resolved as "no conflict, proceed" — no false flags. The second case is the
more informative one: the agent explicitly noted the structural similarity to the documented
issue, named the specific condition that *would* turn it into a real conflict (the flag being
wired into production), and proceeded with an implementation that enforced that boundary. That's
a judgment call about intent, not a keyword match against the decision text.

One consistent secondary pattern across every test in this project: the gated agent resolved
straightforward cases from the one-line index summary alone, and only read a decision's full body
when a case was ambiguous enough to need it. The check overhead for a clear non-conflict was a
single index read (roughly a few hundred tokens); full detail was pulled only when the situation
actually warranted it.

## Where this sits relative to existing tools

This mechanism is not a novel technical capability. A markdown decision log plus an instruction
to check it before acting is a known, simple practice — closer to standard Architecture Decision
Record (ADR) discipline than to any new architecture. Several existing open-source tools already
build on exactly this idea, and at least one — [`adr-kit`](https://github.com/rvdbreemen/adr-kit)
— does it more robustly than what's tested here: it injects relevant decisions into an agent's
context automatically, uses a hook to nudge the agent the moment it touches a governed file, and
checks the actual diff against declarative rules at commit time and in CI. That's deterministic
enforcement; what's described in this document is an instruction, which a session could in
principle ignore.

What's being reported here isn't a claim to have invented the pattern. It's a measured, disclosed
experiment testing whether the *premise* these tools are built on — that checking decisions before
implementing meaningfully changes outcomes — actually holds, what it costs, and where its limits
are (including the precision check above). That kind of controlled, disclosed evidence is harder
to find published alongside these tools than the tools themselves.

That cost is worth stating plainly against the current landscape of large-scale agent memory
products, sourced rather than assumed: the paper introducing Mem0 reports its own memory footprint
at roughly 7,000 tokens per conversation, versus over 600,000 tokens for Zep's temporal-graph
approach on the same measure ([Mem0: Building Production-Ready AI Agents with Scalable Long-Term
Memory, arXiv:2504.19413](https://arxiv.org/pdf/2504.19413), Section 4.5). The mechanism tested in
this document — a flat decision-log check — cost on the order of a few hundred to about 1,400
tokens per check, depending on whether a case was resolved from the index alone or required
reading a decision's full body. These solve different problems at different scales, so this is not
a head-to-head benchmark claim. But for the specific, narrow job tested here — catching a silent
contradiction with an already-made decision, before code is written — it's a reasonable question
to ask whether a large memory-graph system is actually necessary, versus a much simpler pattern
that was measured, here, to work.

## Summary

One narrow, clearly-bounded claim, tested twice on real bugs in a real codebase, with disclosed
methodology fixes and disclosed limitations: a cheap decision-gating check caught a
documented-decision conflict that an otherwise-identical session without that check missed, in
both tested cases, at a lower token cost both times, and didn't false-flag on either of two
constructed non-conflict cases. Two tested cases is evidence, not a guarantee of reliability at
scale. Everything this project does not claim — general memory, general bug-finding, general
engineering discipline — is left to the tools that already do it well.
