# Open Questions

Unresolved questions about Strynex/O-CPS. Nothing in this file is a decision.
Entries stay here until the owner explicitly closes them, and closing an entry
means moving the resolution somewhere authoritative, not editing the question to
look answered.

---

## OQ-1: Is the general-purpose orchestration reframe a live parallel track, or an abandoned branch?

**Status:** Open. Undecided. No work has been done in either direction.
**Raised:** January 2026
**Owner:** Archon C. Locke

### What happened

In late January 2026, in a conversation about pitching and protecting the idea,
Strynex was described in terms that have nothing to do with vehicles. The framing
that came out of it generalizes Strynex to a secure, adaptive orchestration layer
sitting between complex real-world systems and human intent. The pitch language
recorded at the time:

> "Strynex is an orchestration layer for complex systems where full autonomy is
> dangerous and manual control doesn't scale. It sits between raw data and
> action, enforcing policy, trust, and intent, so systems act *appropriately*,
> not just automatically."

And the line meant to separate it from the agent-framework category:

> "Agents optimize. Strynex arbitrates."

The same conversation located the claimed moat in conceptual coherence rather
than in any specific implementation: trust-aware orchestration, policy-aligned
autonomy, human-legible system intent.

### Why this is a question and not a plan

The O-CPS whitepaper, dated three months later in April 2026, is unambiguously
vehicular. It defines Strynex narrowly, in §1.2, as "the reference implementation
and vehicle-facing brand for an O-CPS stack." It says nothing about
general-purpose orchestration, arbitration between systems and human intent, or
any non-automotive domain. The reframe is not mentioned anywhere in the
specification.

So there are at least three readings, and the record does not distinguish between
them:

1. **Abandoned branch.** The January framing was an exploration that was tried
   and dropped. The April whitepaper is the settled direction and the reframe is
   historical only.
2. **Live parallel track.** The vehicular O-CPS work is the first concrete
   instantiation of a broader thesis that still stands, and the general framing is
   deliberately being kept out of the specification to avoid diluting a standards
   play that needs to look narrow and implementable.
3. **Latent renaming problem.** Both are intended to continue, but they cannot
   share the name "Strynex" without confusing an audience, and no decision has
   been made about which one keeps it.

### What is at stake

The two framings pull the trademark, the positioning, and the IP posture in
different directions. The whitepaper commits Strynex to a royalty-free,
patent-non-asserting, governance-transferring standards play (§18, §19). The
January framing treats the concept itself as the defensible asset. Those are not
contradictory, but they are not the same strategy either, and reconciling them
after either one gains traction is more expensive than deciding now.

### What is explicitly NOT being done

Nothing in this repository implements, assumes, or prepares for the
general-purpose reframe. No abstraction has been added to the schemas, the tests,
or the spec to accommodate a non-vehicular domain. This entry exists to keep the
question visible, not to seed work.

### To close this

The owner decides. Not the repository, and not by inference from what gets built
next.
