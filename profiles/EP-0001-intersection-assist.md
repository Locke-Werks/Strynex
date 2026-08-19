# EP-0001: Intersection Assist

| | |
| --- | --- |
| **Profile ID** | `0x0001` |
| **Status** | Proposed for O-CPS v0.4 |
| **Category** | Core, open |
| **License** | CC BY 4.0 |
| **Framework** | [EP-0000](EP-0000-extension-profile-framework.md) |
| **Minimum conformance class** | A (vehicle side). Class I required of the infrastructure peer. |
| **Maximum ASIL introduced** | C, bounded by base 8.2 for `Intent` |
| **IntentCode block** | 88 to 95 |
| **Safe ignore** | Yes, non-CRITICAL. See section 8. |

---

## 1. Motivation

Base 12.3 names this profile and defers it:

> Intersection Assist consumes RSU-sourced MAP and SPaT messages (SAE J2735)
> together with local PerceptionSummary traffic to alert the driver to
> red-light runners, cross-traffic at blind intersections, and unprotected
> left-turn conflicts. It produces advisory output only and is always
> driver-mediated. Detailed specification is out of scope for v0.2 and is
> deferred to a forthcoming extension profile.

Roadmap v0.4 lists "Intersection Assist full specification" as a deliverable.
This document is that deliverable. It adds nothing to the base specification's
scope: the three hazards, the advisory-only constraint, and the driver mediation
are all taken from 12.3 as written.

Base 15.1 gives the deployment: an RSU on a signal head transmitting SPaT and
MAP per SAE J2735, acting as a Class I participant transmitting
`PerceptionSummary` derived from roadside radar and camera.

The gap this profile fills is that 12.3 says what to alert on and says nothing
about how an approaching vehicle expresses where it is going, which is what an
RSU needs in order to assess a conflict at all.

## 2. Applicability

This profile applies at a signalized or MAP-described intersection where at
least one Class I participant transmits SPaT and MAP.

It does **not** apply at uncontrolled intersections, which have no SPaT, and it
does not attempt to substitute for a signal. A vehicle MUST NOT act on this
profile in the absence of a valid MAP describing the intersection it is
approaching.

## 3. Code points

### 3.1 IntentCode

| Value | Name | Meaning |
| --- | --- | --- |
| 88 | `INTERSECTION_APPROACH` | Transmitter is approaching a MAP-described intersection and has selected a movement. |
| 89 | `INTERSECTION_STOPPING` | Transmitter intends to stop at the stop line. |
| 90 | `INTERSECTION_CLEARING` | Transmitter is within the intersection box and intends to clear it. |
| 91 | `UNPROTECTED_LEFT_PENDING` | Transmitter is queued for an unprotected left and has not committed. |
| 92 | `UNPROTECTED_LEFT_COMMITTED` | Transmitter has begun an unprotected left across opposing traffic. |
| 93 to 95 | Reserved | Reserved to this profile. MUST be treated as `INTENT_UNKNOWN`. |

These are transmitted in the base `Intent` message. `Intent.code` carries the
value; the `ExtensionBlock` of section 3.2 carries the qualifying detail.

Base 8.4 rates `Intent` as event-driven with a 2 Hz background heartbeat. On
intersection approach, transmission rate SHOULD rise to 5 Hz from 150 m or
5 seconds time-to-stop-line, whichever is reached first, and MUST NOT exceed the
10 Hz ceiling of base 8.2.

### 3.2 ExtensionBlock payload

```protobuf
// ExtensionBlock.profile_id = 0x0001, profile_version = 1
message IntersectionApproach {
  bytes  intersection_id   = 1;  // J2735 MAP IntersectionReferenceID, 4 bytes
  uint32 ingress_lane      = 2;  // J2735 MAP lane ID
  uint32 egress_lane       = 3;  // J2735 MAP lane ID; 0 if undecided
  uint32 signal_group      = 4;  // J2735 SPaT signal group for the movement
  uint32 time_to_stop_line_ms = 5;  // transmitter's own estimate
  uint32 distance_to_stop_line_cm = 6;
  StopIntent stop_intent   = 7;
}

enum StopIntent {
  STOP_INTENT_UNKNOWN = 0;
  WILL_STOP           = 1;  // transmitter has committed to stopping
  WILL_PROCEED        = 2;  // transmitter has committed to proceeding
  UNDECIDED           = 3;  // driver has not committed and prediction is low confidence
}
```

`Intent.driver_committed` (Appendix A.3, field 5) distinguishes a
driver-committed intent from a system prediction and is load-bearing here.
`stop_intent = WILL_PROCEED` with `driver_committed = false` is a prediction and
MUST NOT alone raise a red-light-runner alert at another participant.

## 4. Behavior

Three assessments, matching the three hazards named in base 12.3. Each is
performed by the receiving participant, not by the RSU, so that an
infrastructure failure degrades to local behavior rather than to wrong behavior.

### 4.1 Red-light runner

A participant assesses an approaching peer as a probable red-light runner when
all of the following hold:

- a valid SPaT indicates the peer's `signal_group` is in, or will be in, a
  stop-required state at the peer's `time_to_stop_line_ms`;
- the peer's deceleration required to stop at the stop line exceeds 3.0 m/s^2,
  computed from its `Pose.speed_mps` and `distance_to_stop_line_cm`;
- the peer has not transmitted `INTERSECTION_STOPPING` or
  `stop_intent = WILL_STOP` within the last 500 ms.

The 3.0 m/s^2 figure is the threshold above which stopping is no longer routine
for a passenger vehicle on dry pavement. A profile implementation MAY use a
lower threshold when road-surface telemetry (base `ROAD_SURFACE_FRICTION`)
indicates reduced friction, and MUST NOT use a higher one.

A participant MAY also assess a peer as a probable runner on kinematics alone,
from `PerceptionSummary`, without any `Intent` from that peer. Vehicles that do
not implement this profile still get assessed. That is deliberate: the hazard is
posed by unequipped vehicles at least as much as by equipped ones.

### 4.2 Cross-traffic at a blind intersection

Base 15.1 already describes the mechanism: the RSU's `PerceptionSummary` covers
approach legs the vehicle cannot see, and the vehicle fuses them. This profile
adds only the conflict test.

A cross-traffic conflict exists when a track in the fused world model, whether
locally sensed or RSU-sourced, occupies a MAP-described lane whose movement
conflicts with the receiver's own `egress_lane`, and the projected paths
intersect within the receiver's time-to-stop-line plus 2 seconds.

Conflict assessment MUST use the fused track, subject to base 11.4 and to the
trust weighting of 11.3. An RSU-sourced track from an attested Class I
participant at HIGH confidence satisfies 11.4. An unattested track does not, and
MUST NOT raise an advisory on its own.

### 4.3 Unprotected left-turn conflict

A participant transmitting `UNPROTECTED_LEFT_PENDING` or
`UNPROTECTED_LEFT_COMMITTED` is assessed against opposing through movements in
the fused world model.

Two-wheeled classes deserve specific mention. `MOTORCYCLE` (0x04) and `BICYCLE`
(0x05) are the classes most often missed in this conflict and the ones for which
cooperative perception adds the most. A conforming implementation MUST NOT apply
a higher confidence threshold to a two-wheeled opposing track than it applies to
`PASSENGER_CAR`.

Note that errata [E-03](../docs/errata-v0.2.md#e-03) currently prevents a
Class A two-wheeler beacon from reaching the base 11.4 advisory threshold at
all. Until E-03 is resolved, this clause is effective only for two-wheelers
detected by a local sensor or by an attested Class I RSU.

## 5. Output constraints

This profile produces advisory output only, per base 12.3 and base 13.1.

A conforming implementation:

- MUST route all output through the vehicle's ADAS domain controller per
  base 13.1, which retains actuation authority;
- MUST NOT gate lateral control on any assessment in section 4;
- MUST NOT command an increase in speed;
- MAY command a bounded longitudinal reduction, not exceeding 3.5 m/s^2
  deceleration, and only for a red-light-runner or cross-traffic assessment that
  satisfies base 11.4;
- MUST make any automatic response driver-overridable at any instant;
- MUST NOT suppress or delay the vehicle's own automatic emergency braking.

## 6. Declarations required by EP-0000 section 6

| Item | Value |
| --- | --- |
| Minimum conformance class (item 8) | A on the vehicle side. The profile is inert without a Class I peer transmitting SPaT and MAP. |
| Maximum ASIL introduced (item 9) | C, matching the base 8.2 allocation for `Intent`. The profile introduces no behavior of higher integrity than the message class carrying it. |
| Safe-ignore behavior (item 10) | Section 8. |
| Privacy analysis (item 11) | Section 9. |

The profile satisfies the prohibitions of EP-0000 section 6: it defines no new
message class, raises no ASIL allocation, changes no threshold in base 11, 12 or
14, creates no control path around base 13.1, requires no cloud connectivity,
and is not a prerequisite for any base conformance class.

## 7. Interaction with the base specification

| Base clause | Interaction |
| --- | --- |
| 11.2 rule 1 | Local sensor tracks at HIGH or VERY_HIGH confidence dominate. An RSU assertion never overrides a confident local detection, including when the RSU is wrong. |
| 11.4 | Unchanged. Assessments in section 4 are gated by it, not exempted from it. |
| 12.1 | A vehicle in a C-ACC Group that receives an intersection advisory follows the group's dissolution rules; this profile does not add a dissolution trigger. |
| 14.2 | An intersection advisory is not a fail-safe trigger. Loss of RSU messages reverts to local-sensor-only assessment, per base 3.3 graceful degradation. |
| 15.1 | This profile is the specification of the behavior 15.1 describes. |

## 8. Safe-ignore statement

Required by EP-0000 section 7.1. The CRITICAL flag is **never** set by this
profile.

A receiver that discards all EP-0001 extension content:

- sees `Intent` messages with codes 88 to 95, which section 7.1 of EP-0000
  requires it to map to `INTENT_UNKNOWN`. An unknown intent contributes nothing
  to prediction, which is the same position as a peer transmitting no intent at
  all;
- continues to receive and fuse the RSU's `PerceptionSummary` normally, because
  that is base behavior and carries no extension content;
- loses the three assessments of section 4 and falls back to local sensing.

That fallback is the pre-O-CPS status quo at every intersection in the world. It
is safe by construction, which is why this profile is non-CRITICAL.

## 9. Security and privacy analysis

**Against base 10.1.** The payload of section 3.2 carries a J2735 intersection
reference, two lane identifiers, a signal group, and two kinematic estimates.
None is a persistent identifier. None survives certificate rotation. Lane
identity is intersection-scoped and meaningless elsewhere.

**Retention.** `Intent` retention is capped at 30 seconds by base 10.3 with no
exception, and extension content inherits the retention limit of its carrying
message class. Section 9 of EP-0000 requires this payload encoding to be
published so the 16.3 privacy scan can run against it, which this document does.

**Correlation risk, and it is real.** A participant declaring `ingress_lane` and
`egress_lane` at successive intersections discloses a route more precisely than
`PerceptionSummary` pose alone does, because a turn sequence is more
distinguishing than a position trace at the same sample rate. Pseudonym rotation
under base 9.1.2 is the mitigation, and base 9.1.2 already requires a fresh
certificate on entering a new geographic cell. Implementers SHOULD align
rotation to intersection departure where the SCMS geographic partition permits
it. This is a SHOULD rather than a MUST because forcing rotation on a fixed
geographic event is itself a correlation signal.

**Adversarial cases.**

| Attack | Effect | Mitigation |
| --- | --- | --- |
| Fabricated `UNPROTECTED_LEFT_COMMITTED` from a vehicle that is not there | Spurious conflict advisory to a real vehicle | Base 9.3 internal-inconsistency and consensus-deviation detectors; base 11.4 threshold; the fabricated vehicle has no corroborating `PerceptionSummary` from the RSU. |
| Fabricated `WILL_PROCEED` at speed, to trigger red-light-runner alerts | Nuisance advisories, possible bounded braking in receivers | Section 4.1 requires kinematic corroboration, not the declaration alone. A declaration inconsistent with the sender's own transmitted pose fires the base 9.3 internal-inconsistency detector. |
| Compromised RSU asserting phantom cross traffic | Bounded deceleration in approaching vehicles | Base 11.2 rule 1 keeps confident local detections dominant. Section 5 bounds the response at 3.5 m/s^2. Consensus deviation across multiple approaching vehicles produces misbehavior reports against the RSU. |
| SPaT replay | Advisory computed against a stale phase | Base 13.4 rejects any message more than 500 ms in the future or beyond its stated lifetime. J2735 SPaT carries its own timestamp and MUST be freshness-checked independently. |

**Residual risk, stated plainly.** A single compromised Class I enrollment can
induce nuisance braking of up to 3.5 m/s^2 in vehicles approaching one
intersection, until trust weights decay and misbehavior reports are filed. That
is within the bound base 15.5 sets for an adversary holding one compromised
enrollment, which is "momentary spurious advisories". It is not zero.

## 10. Conformance

Proposed assertions for the base 16.3 categories. Each is written to be
executable by the harness once it exists.

| Category | Assertion |
| --- | --- |
| Wire format | `IntersectionApproach` round-trips bit-identically inside an `ExtensionBlock` inside an `Intent`. |
| Wire format | An `Intent` carrying code 93, 94 or 95 decodes with `code` mapped to `INTENT_UNKNOWN` and does not error. |
| Timing | Approach transmission rate rises to at least 5 Hz within 150 m or 5 s of the stop line, and never exceeds 10 Hz. |
| Behavioral | A peer requiring more than 3.0 m/s^2 to stop, against a SPaT stop-required phase, and not asserting `WILL_STOP`, produces a red-light-runner assessment. |
| Behavioral | The same peer asserting `INTERSECTION_STOPPING` within the last 500 ms produces no assessment. |
| Behavioral | An RSU-sourced cross-traffic track from an unattested source produces no advisory. |
| Behavioral | A two-wheeled opposing track and a `PASSENGER_CAR` opposing track at identical confidence and geometry produce identical advisories. |
| Safety | No assessment in section 4 produces lateral control input. |
| Safety | Automatic longitudinal response never exceeds 3.5 m/s^2 and is overridable within one control cycle. |
| Safety | Loss of all RSU messages reverts to local-sensor-only assessment without a fail-safe transition. |
| Privacy | The prohibited-field scan runs against `IntersectionApproach` and finds no prohibited content. |
| Misbehavior | A fabricated `UNPROTECTED_LEFT_COMMITTED` with no corroborating `PerceptionSummary` fires the consensus-deviation detector. |
| Misbehavior | An `Intent` whose `distance_to_stop_line_cm` contradicts the sender's own `Pose` fires the internal-inconsistency detector. |

## 11. Open questions

1. **Whether `intersection_id` should be the J2735 `IntersectionReferenceID` or
   an O-CPS-local identifier.** J2735 is the obvious choice and creates a
   normative dependency on SAE licensing that the rest of the base specification
   avoids for message content. Base 1.1 already inherits from J2735
   conceptually; this would be the first place a J2735 field is carried
   verbatim.
2. **Whether unsignalized intersections belong in a second profile.** They have
   no SPaT and therefore no red-light-runner case, but the cross-traffic and
   unprotected-left cases apply unchanged. Splitting keeps each profile honest;
   merging avoids two profiles that share most of their text.
3. **Whether the 3.0 m/s^2 red-light threshold should vary by vehicle class.** A
   loaded commercial truck cannot stop at 3.0 m/s^2 from speed, so it is assessed
   as a runner more often than a passenger car in the same geometry. Base 7.2
   carries `COMMERCIAL_TRUCK` as a class, so the information is available. No
   position is taken here.
