# O-CPS v0.2 Errata

Defects in the v0.2 specification, each with proposed normative replacement
text, held for adoption into v0.3.

**This file does not amend anything.** Per the repository rule, `spec/` is a
mechanical conversion of `reference/OCPS_Whitepaper_v0.2.docx` and is not edited
to change the specification. Adopting any resolution below means editing the
source document and re-running the conversion. Until that happens, v0.2 stands
as written and this register records the gap.

Two errata registers now exist and they cover different things:

- [`conversion-notes.md`](conversion-notes.md) records defects **found during
  conversion**, principally the nine-instance cross-reference offset. Those are
  already known and are not repeated here.
- This file records defects found by **reading the specification against itself
  and against the schemas**. All are new. Each was verified directly against the
  file it cites.

## Index

| ID | Title | Severity | Class | Blocks |
| --- | --- | --- | --- | --- |
| [E-01](#e-01) | Class A conformance declaration has nowhere to go | Major | Schema + text | v0.3 schemas, Class A conformance |
| [E-02](#e-02) | `w_history` is inert by construction and its initial value is undefined | Major | Normative text | Fusion conformance, harness |
| [E-03](#e-03) | VRU beacon tracks are arithmetically barred from raising an advisory | **Critical** | Normative text | 15.4 scenario, VRU deployments |
| [E-04](#e-04) | No positional accuracy requirement exists anywhere | Major | Normative text | Implementer BOM, fusion validity |
| [E-05](#e-05) | 18.4 names the wrong ISO committee and omits ETSI TC ITS | Minor | Editorial | Governance transfer credibility |
| [E-06](#e-06) | Class A is forced to carry ASIL C for a message it may have no use for | Major | Normative text | Class A cost, VRU beacons |
| [E-07](#e-07) | The reference C codec is in 16.2 and in no milestone | Major | Process | v0.3 scope, wire-format conformance |
| [E-08](#e-08) | The trademark is scoped to an artifact 16.2 declares unshippable | Major | IP posture | Any shipped product |

Severity is defined as: **Critical**, a conforming implementation behaves less
safely than the specification's own prose promises; **Major**, a conforming
implementation cannot be built, tested, or sold without the implementer
inventing a requirement; **Minor**, the document is wrong in a way that costs
credibility but not correctness.

---

<a id="e-01"></a>

## E-01: Class A conformance declaration has nowhere to go

**Severity:** Major. **Affects:** 16.1, Appendix A.2,
`schemas/proto/ocps/v2/perception.proto`

### The defect

Section 16.1 states:

> A compliant implementation MUST declare its class in its Attestation messages
> (or, for Class A, in its PerceptionSummary vendor-extension field).

`PerceptionSummary` has no vendor-extension field. Appendix A.2 defines it as
`header`, `own_pose`, `own_dims`, `objects`, `signature`. The only vendor field
anywhere in the message set is `PerceivedObject.vendor_oid`, which A.2 marks
`optional; empty unless class_id=0xFF`.

### Consequence

Class A is the class that does not transmit Attestation, so 16.1 hands it the
only declaration route available and that route does not exist. The failure is
worst exactly where Class A matters most: a VRU beacon, named as a Class A use
case by 16.1 itself, transmits its own pose and **zero** perceived objects, so
it has not even a `PerceivedObject` to hang a vendor OID on. It cannot make a
mandatory declaration at all.

Downstream, 11.3 sets `w_attest = 0.5` for a source whose "attestation is absent
but cert is valid (Class A)". A receiver cannot apply that branch by inspection:
absence of Attestation within its 30-second 9.2 lifetime is indistinguishable
from a Class B participant whose Attestation was lost to congestion. The
declaration is what disambiguates, and it is unreachable.

### Proposed resolution

Add a field to `PerceptionSummary`. Protobuf field addition is wire-compatible,
so `OcpsHeader.version` stays at 2 and no existing decoder breaks.

```protobuf
message PerceptionSummary {
  OcpsHeader header     = 1;
  Pose       own_pose   = 2;
  Dimensions own_dims   = 3;
  repeated PerceivedObject objects = 4;
  ConformanceDeclaration conformance = 5;  // NEW, see below
  bytes signature       = 15;              // now signs fields 1-5
}

// Transmitted by every participant. REQUIRED for Class A, which does not
// transmit Attestation; OPTIONAL but RECOMMENDED for Class B and Class I,
// which declare class in Attestation per 16.1.
message ConformanceDeclaration {
  uint32 conformance_class = 1;  // 0x41='A', 0x42='B', 0x49='I' (ASCII)
  repeated uint32 profiles = 2;  // supported extension profile IDs, EP-0000
  string vendor_oid        = 3;  // OPTIONAL
}
```

Replace the 16.1 sentence with:

> A compliant implementation MUST declare its class. Class B and Class I
> participants declare it in the Attestation message. Class A participants,
> which do not transmit Attestation, MUST declare it in the
> `ConformanceDeclaration` field of every PerceptionSummary they transmit. A
> receiver that has observed no Attestation from a sender within the 9.2
> attestation lifetime and no `ConformanceDeclaration` MUST treat the sender as
> having stale attestation (`w_attest = 0.2`, 11.3) rather than as Class A.

The last sentence closes the disambiguation hole: silence is now penalised more
than an honest Class A declaration, which is the correct incentive.

**Signature scope note.** A.2 currently signs fields 1 through 4. Adding field 5
inside the signed range is required, otherwise a declaration can be stripped or
forged in flight, which converts this fix into a trust-downgrade attack.

---

<a id="e-02"></a>

## E-02: `w_history` is inert by construction and its initial value is undefined

**Severity:** Major. **Affects:** 11.3, 9.1.2, 10.5

### The defect

Section 11.3 defines one of the four multiplicative trust factors as:

> `w_history` = a rolling average in [0, 1] of the sender's agreement with
> consensus and with local sensors over the past 5 min (clamped >= 0.1 to allow
> recovery from transient faults)

Section 9.1.2 requires:

> Certificates MUST be rotated at least every 5 minutes during active operation

and adds that butterfly key expansion SHOULD be used "so that pseudonyms within
an epoch are not linkable". Section 10.1 makes the pseudonym certificate ID the
only identifier on the wire.

The history window and the maximum identifier lifetime are the same 5 minutes.
A receiver therefore can never accumulate a full window of history against any
sender, and after each rotation it is looking at what is, by design, a stranger.

### Consequence

Two problems, and the second is worse than the first.

1. `w = w_cert * w_attest * w_history * w_conf` reduces in practice to
   `w_cert * w_attest * w_conf`. A factor the specification says implementations
   "MUST NOT remove" does no work.
2. **The initial value is undefined.** v0.2 never states what `w_history` is for
   a sender not previously observed, which is every sender at first contact and
   every sender after every rotation. If an implementer picks the clamp floor of
   0.1, every conformant participant is de-rated to `w <= 0.05` and fails 11.4's
   0.5 threshold permanently. If an implementer picks 1.0, the factor is inert
   and harmless. Two conformant implementations can differ by a factor of ten on
   the same input, which defeats the point of a normalized profile.

There is also a trap: the obvious engineering fix, keeping history across
rotations by correlating senders, is precisely what 10.5 prohibits
("Construct long-term movement traces of individual vehicles"). An implementer
who reaches for the intuitive solution violates the privacy model.

### Proposed resolution

Append to 11.3:

> `w_history` MUST initialize to 1.0 for any sender with no observation history
> under the currently-presented `cert_id`. Implementations MAY accumulate
> `w_history` within a single certificate epoch and MUST discard the accumulated
> value on observing a new `cert_id` from a previously-tracked sender.
> Implementations MUST NOT attempt to carry `w_history` across a `cert_id`
> rotation by correlating pose, kinematics, or message timing, which would
> constitute a prohibited analysis under 10.5.
>
> Note that 9.1.2 bounds certificate lifetime at 5 minutes, which is the width
> of the `w_history` window. This factor is therefore effective only against a
> sender that misbehaves rapidly within one epoch. Sustained misbehavior across
> rotations is the responsibility of the misbehavior detectors of 9.3 and of the
> Misbehavior Authority, which holds the linkage keys that receivers
> deliberately do not.

The closing note is not decoration. It tells an implementer why the factor looks
weak and stops them from "fixing" it in a way that breaks section 10.

---

<a id="e-03"></a>

## E-03: VRU beacon tracks are arithmetically barred from raising an advisory

**Severity:** Critical. **Affects:** 15.4, 11.3, 11.4, 7.3, 16.1

### The defect

Four clauses interact, and the product of them contradicts the fifth.

1. Section 16.1 places VRU beacons in **Class A**.
2. Section 11.3 sets `w_attest = 0.5` for Class A ("attestation is absent but
   cert is valid").
3. Section 15.4 specifies that a VRU beacon transmits "at low rate (1-2 Hz) and
   **LOW-to-MEDIUM confidence**".
4. Section 7.3 caps MEDIUM at 127 of 255, so `w_conf <= 127/255 = 0.498`.

Therefore, for a VRU beacon track with no other contributing source:

```text
w = w_cert * w_attest * w_history * w_conf
  = 1.0    * 0.5      * 1.0       * 0.498
  = 0.249
```

Section 11.4 states:

> A fused track MAY influence driver-visible advisories or ADAS inputs only if
> its effective trust weight (maximum across contributing sources) is at least
> 0.5 AND at least one contribution is from a local sensor OR from an attested
> cooperative source at HIGH confidence or better. Tracks below this threshold
> MAY be displayed to the driver as informational but MUST NOT gate control
> decisions.

0.249 is below 0.5. A VRU beacon track fails the threshold on the first
condition, before the second condition is even reached.

Meanwhile 15.4 promises:

> Vehicles fuse these into the world model as flagged vulnerable tracks,
> weighted appropriately.

### Consequence

A pedestrian or cyclist beacon can be drawn on a display and can do nothing
else. It cannot raise a driver-visible advisory, because 11.4 gates advisories
at 0.5. It cannot influence ADAS input. The specification's own worked example
of protecting a vulnerable road user is defeated by the specification's own
arithmetic.

The failure is inverted with respect to risk. Where the vehicle's own sensors
already see the pedestrian, the local track satisfies 11.4 and the beacon adds
nothing. Where the pedestrian is occluded, which is the only case in which the
beacon carries information the vehicle does not already have, the beacon is the
sole contributor and is muted.

This defect alone removes the commercial basis for any VRU-first deployment.

### Proposed resolution

This needs a narrow, argued exception rather than a lower global threshold. The
0.5 threshold exists to stop an adversary inducing unsafe actuation. A track
that can only cause deceleration does not present that hazard. Suppressing VRU
warnings presents an asymmetric one.

Insert as 11.5:

> **11.5 Vulnerable Road User Exception**
>
> A fused track whose class is PEDESTRIAN (0x06), BICYCLE (0x05), or MOTORCYCLE
> (0x04) MAY influence driver-visible advisories at an effective trust weight of
> at least 0.2, notwithstanding the threshold of 11.4, subject to all of the
> following:
>
> - The resulting advisory MUST be presented as unverified, by a means distinct
>   from that used for advisories meeting the 11.4 threshold.
> - The track MUST NOT gate lateral control.
> - The track MUST NOT gate any increase in speed or any reduction in following
>   gap.
> - The track MUST NOT be the sole input to automated emergency braking.
> - Any automatic longitudinal response MUST be bounded at 2.0 m/s^2
>   deceleration and MUST be driver-overridable at any instant.
> - The track remains subject to the misbehavior detectors of 9.3. A sender
>   whose VRU-class reports repeatedly fail consensus comparison MUST be
>   de-rated and reported.
>
> Rationale, normative for interpretation: the threshold of 11.4 exists to
> prevent an adversary from inducing unsafe actuation. Within the constraints
> above, an adversary asserting fabricated VRU tracks can induce only bounded
> deceleration and a driver-visible warning. That residual hazard is smaller and
> better bounded than the hazard of suppressing every warning about an occluded
> pedestrian, which is the behavior 11.4 produces unamended.

**Alternative considered and rejected.** Raising `w_attest` for VRU beacons via
a lightweight attestation would preserve a single threshold, but it requires a
pedestrian's handset to hold an HSM meeting 13.3, which defeats the near-zero
BOM that makes beacons deployable. Recorded so the trade is not relitigated.

**Note on phantom braking.** The 2.0 m/s^2 bound and the sole-input prohibition
are load-bearing and should not be dropped when this text is edited. Without
them the exception becomes a remote braking primitive available to anyone
holding one valid enrollment, which 15.5 forbids.

---

<a id="e-04"></a>

## E-04: No positional accuracy requirement exists anywhere

**Severity:** Major. **Affects:** 7.1, 7.4, 8.3, 11.2

### The defect

Section 7.1 specifies the reference frame exhaustively: WGS-84, decimal degrees,
metres above the ellipsoid, heading clockwise from true north, velocity along
the heading axis, and "Deviation from WGS-84 is a conformance failure."

It states no accuracy requirement. A search of the full specification returns no
normative use of accuracy, precision, covariance, or any equivalent. Appendix
A.1 `Pose` carries no uncertainty field, and A.2 `PerceivedObject` carries no
covariance, which `conversion-notes.md` already records as a v0.2 fact.

### Consequence

Objects are transmitted as `range_m` and `bearing_deg` **relative to the
transmitter** (8.3, A.2). Every consumer must resolve them against the
transmitter's own pose. Transmitter pose error therefore propagates directly and
unbounded into every absolute object position a receiver derives.

Two implementations can both be fully conformant and disagree by half a road
width. Section 11.2 requires a receiver to reject tracks that are "physically
implausible given its prior track state" and to fire a misbehavior detector, so
an honest participant with a cheap receiver is indistinguishable from an
attacker, and will be reported as one.

This is also the single clause that sets aftermarket bill of materials. A
consumer GNSS module at 2 to 5 m and an RTK-corrected solution at sub-metre
differ by roughly two orders of magnitude in delivered cost once the correction
subscription is counted. v0.2 gives an implementer no basis to choose, which
means no two implementers will choose the same.

### Proposed resolution

Add as 7.1.1, reusing the confidence bands of 7.3 rather than inventing a second
scale:

> **7.1.1 Positional Accuracy**
>
> A participant MUST maintain an estimate of the horizontal accuracy of its own
> pose, expressed as a 68% confidence radius (1-sigma) in metres.
>
> | Own-pose horizontal accuracy | Permitted behavior |
> | --- | --- |
> | <= 1.5 m | Full participation. Object confidence up to VERY_HIGH permitted. |
> | > 1.5 m and <= 5.0 m | Object confidence MUST NOT exceed MEDIUM (7.3). Declared relevance horizon (7.4) MUST NOT exceed 100 m. |
> | > 5.0 m | MUST NOT transmit PerceptionSummary. MAY continue to receive. |
>
> The 1.5 m boundary is set at slightly under half a standard 3.6 m lane so that
> a conforming transmitter's lane assignment is unambiguous to a receiver.
>
> A participant whose accuracy estimate is unavailable MUST treat it as
> exceeding 5.0 m. A participant in a GNSS-denied environment MUST apply this
> rule to its dead-reckoned estimate, whose accuracy degrades with time since
> last fix.
>
> Receivers MUST NOT fire the physical-implausibility detector of 9.3 on a
> single positional disagreement within the accuracy envelope a sender's
> declared confidence band implies.

The final sentence prevents this fix from generating false misbehavior reports
against honest low-cost participants, which would otherwise be its most likely
side effect.

**Open question deliberately left open.** Whether `Pose` should carry an
explicit accuracy field on the wire, rather than being inferred from the
confidence band, is a genuine design decision with a message-size cost against
the 8.2 200 to 500 byte envelope. It is not resolved here. It belongs in the
v0.3 schema decision alongside the other constraint questions in
[`schemas/README.md`](../schemas/README.md).

---

<a id="e-05"></a>

## E-05: 18.4 names the wrong ISO committee and omits ETSI TC ITS

**Severity:** Minor. **Affects:** 18.4

### The defect

Section 18.4 states the intent to transfer governance to

> an established standards body (IEEE-SA, ISO TC 22 SC 31, or SAE)

ISO/TC 22/SC 31 is *Road vehicles: Data communication*, covering in-vehicle
networking, diagnostics and the extended vehicle. It is not the committee for
cooperative ITS. Cooperative ITS is **ISO/TC 204** (Intelligent transport
systems), working with CEN/TC 278 under the Vienna Agreement in Europe.

**ETSI TC ITS is absent from the list**, which is the more damaging omission.
Section 1.1 states O-CPS inherits the Collective Perception Message concept from
ETSI TS 103 324, and 17.2 positions O-CPS as extending it. The one body whose
work O-CPS most directly builds on is not offered the specification.

### Consequence

No technical consequence. The cost is entirely credibility, and it is incurred
with exactly the audience 18.4 is addressed to. A committee reader reaches 18.4
near the end of the document, and it is the paragraph that tells them whether
the author knows the landscape.

### Proposed resolution

Replace the parenthetical with:

> an established standards body (ISO/TC 204, ETSI TC ITS, IEEE-SA, or SAE, with
> the choice made in consultation with the active implementer community and
> having regard to the regional profile balance of the deployed population at
> the time of transfer)

---

<a id="e-06"></a>

## E-06: Class A is forced to carry ASIL C for a message it may have no use for

**Severity:** Major. **Affects:** 16.1, 8.2, 14.3, 15.4

### The defect

Section 16.1 defines Class A as: "Transmits and receives PerceptionSummary and
Intent." Transmission of Intent is not qualified as optional.

Section 8.2 allocates **ASIL C** to Intent, against ASIL B for
PerceptionSummary, on the stated ground that "an Intent message can trigger an
actuation-adjacent behavior ... whose misstatement has higher consequence."

Section 14.3 requires that "A conforming implementation SHALL be verified
against the ISO 26262 Part 4 and Part 6 lifecycle", with no scoping by
conformance class.

So the cheapest class in the specification carries the second-highest ASIL in
the specification.

### Consequence

Two problems.

The first is coherence. Section 16.1 lists "VRU beacons" as a Class A use case.
A pedestrian beacon is required to transmit Intent, and the 8.4 intent codes are
STRAIGHT, LANE_CHANGE_LEFT, LANE_CHANGE_RIGHT, TURN_LEFT, TURN_RIGHT, MERGE,
EXIT, STOP, YIELD, PLATOON_FORM, PLATOON_LEAVE. None has a meaning for a person
on foot. The specification requires a beacon to transmit a message class for
which it has no vocabulary.

The second is cost. Section 3.3 sets retrofittability as a design goal: Class A
"MUST be implementable on current C-V2X/ITS-G5 hardware without ECU
replacement." An ASIL C development lifecycle is the dominant non-recurring cost
in such a product and is what a retrofit vendor is least equipped to carry. The
goal in 3.3 and the obligation in 16.1 plus 8.2 plus 14.3 pull against each
other.

### Proposed resolution

Split the class along the seam that already exists in practice.

Amend the 16.1 table:

> | Class | Capability | Use case |
> | --- | --- | --- |
> | **Class A0 (Beacon)** | Transmits PerceptionSummary. Receives PerceptionSummary and Intent. Does not transmit Intent, Attestation or GroupState. Maximum ASIL allocation: **B**. | VRU beacons; fixed hazard beacons; low-cost telemetry-only participants. |
> | **Class A (Baseline)** | All of Class A0, plus transmits Intent. Does not transmit Attestation. Cannot participate in C-ACC Groups as leader. Maximum ASIL allocation: **C**. | Aftermarket retrofit; low-cost OEM integrations. |

and amend 14.3:

> A conforming implementation SHALL be verified against the ISO 26262 Part 4 and
> Part 6 lifecycle **at the highest ASIL allocated by 8.2 to any message class
> the implementation transmits**. A participant that receives but does not
> transmit a class does not inherit that class's allocation. The declared
> conformance class (16.1) bounds this allocation and MUST be stated in the
> vendor's O-CPS conformance report.

This makes the VRU beacon coherent, gives the retrofit market an ASIL B entry
point consistent with 3.3, and leaves the ASIL C obligation exactly where 8.2
argued it belonged, on participants that actually assert Intent.

---

<a id="e-07"></a>

## E-07: The reference C codec is in 16.2 and in no milestone

**Severity:** Major. **Affects:** 16.2, 16.3, `ROADMAP.md`, 19

### The defect

Section 16.2 defines the reference implementation as **four** components:

> a protobuf schema bundle, a C reference encoder/decoder, a Python test
> harness, and a set of recorded PCAPs covering the five deployment scenarios of
> 15

`ROADMAP.md`, following 19, scopes v0.3 as **three** deliverables: the
machine-readable schemas, the Python test harness, and the adversarial PCAP
corpus. The C reference encoder/decoder appears in no milestone from v0.3
through v1.0.

### Consequence

Section 16.3's first conformance category is:

> Wire-format conformance: encoded messages MUST decode bit-identically on the
> reference decoder.

There is no reference decoder and none is scheduled. Ship all three v0.3
deliverables exactly as scoped and the first and most basic conformance category
still cannot execute. Section 18.4 conditions the governance transfer on "three
independent conformant implementations", and conformance is currently
undemonstrable by construction.

Two compounding items, recorded here because they land in the same milestone:

- Section 8.2 states rate and size envelopes but not the **tolerance** around
  them that 16.3 requires. This is already marked `TODO(spec)` in
  [`tests/README.md`](../tests/README.md). Normative text gates harness code, so
  it must be resolved before the timing suite can be written, not after.
- Section 16.3 cryptographic conformance requires provisioning "using SCMS or
  C-ITS test PKI credentials", and v0.2 does not name which instance. Enrollment
  in a test PKI has a lead time measured in weeks. It is an application, not a
  sprint, and it should be started before it is needed.

### Proposed resolution

A roadmap change, not a specification change. One of two, chosen deliberately:

**Option A, preferred.** Add the C reference encoder/decoder to v0.3 as a fourth
deliverable, matching 16.2. Accept that v0.3 is a 2 to 4 engineer-quarter
milestone rather than a one-quarter one, and re-date it honestly.

**Option B.** Defer the codec to v0.4 explicitly, and record in
`tests/README.md` that wire-format conformance is **not testable at v0.3**,
naming the milestone at which it becomes testable. A stated deferral is a
schedule decision. Silence is a defect that surfaces at the first interop event.

Either way, `ROADMAP.md` and 19 should agree with 16.2 about how many components
the reference implementation has.

---

<a id="e-08"></a>

## E-08: The trademark is scoped to an artifact 16.2 declares unshippable

**Severity:** Major. **Affects:** 18.5, 16.2

### The defect

Section 18.5 states:

> "Strynex" is a trademark of Specter Point Intelligence, LLC, applied only to
> the reference implementation and branded overlay (Slipstream Mode).

Section 16.2 states of that same reference implementation:

> The reference implementation is licensed under Apache 2.0 and is intended for
> conformance testing, **not for production deployment**.

### Consequence

Sections 18.1 through 18.4 deliberately disable every capture mechanism in the
project: CC BY on the specification, Apache 2.0 on the implementation, no patent
assertion, governance transferred at v1.0. `NOTICE` states the resulting
position accurately, that the mark is the one asset not licensed.

Section 18.5 then binds that sole retained asset to an artifact the document
elsewhere says must not ship, and to one named overlay. Read together, the two
clauses leave no trademark basis for a product. Any shipped hardware, any
commercial software stack, and any of the other overlays fall outside the stated
scope of the mark.

This is a drafting artifact rather than an intention. It matters because it is
the specific clause that would be read in a licensing negotiation or a dispute,
and because trademark scope is established by use: a mark used outside its
stated scope for years, while the specification says otherwise, is a weaker
mark.

### Proposed resolution

Replace 18.5 with:

> **18.5 Trademark**
>
> "Strynex" and the Strynex marks are trademarks of Specter Point Intelligence,
> LLC. They are applied to the reference implementation, to branded
> application-layer overlays including Slipstream Mode, and to commercial
> products, services, and certification programmes offered under them. They are
> not licensed by 18.1 or 18.2 and are not conveyed by any conformance claim.
>
> Use of the term "O-CPS", and any claim of conformance to a conformance class
> of 16.1, is governed only by that conformance-class framework and is not
> subject to trademark gating. An implementer may build, ship, certify, and
> advertise a conformant O-CPS product without permission from, or reference to,
> Specter Point Intelligence, LLC.

This preserves the property that makes the specification adoptable, which is
that conformance never requires anyone's permission, while removing the
accidental restriction on the mark.

**Related, and deliberately not resolved here.** Section 16.2's "not for
production deployment" is correct for a reference implementation and should
stand. The consequence is that a shipping Strynex product is a distinct artifact
from the reference implementation and needs its own name in the document. That
is a product decision, not an erratum.

---

## Adoption

Nothing above is adopted. Each entry needs an owner decision, then an edit to
`reference/OCPS_Whitepaper_v0.2.docx`, then re-conversion per
[`conversion-notes.md`](conversion-notes.md).

Recommended order, cheapest and most blocking first:

1. **E-05** and **E-08**. Editorial, one paragraph each, no downstream effect.
2. **E-07**. A roadmap decision, and it sets the scope of everything else in
   v0.3.
3. **E-01** and **E-06**. Both change 16.1 and both change what the schema and
   harness must do. Decide them together.
4. **E-02** and **E-04**. Both change fusion behavior and both need the harness
   to exist before they can be shown to work.
5. **E-03**. Highest severity and the one that needs the most review, because it
   introduces an exception to a safety threshold. It should not be adopted on
   one person's reading. Route it through whichever functional-safety reviewer
   signs the section 14 hazard analysis.
