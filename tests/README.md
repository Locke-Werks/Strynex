# O-CPS Conformance Test Suite (scaffolding only)

**Status: not implemented.** Every directory below is an empty stub. Nothing in
this tree executes, asserts, or passes. The Python test harness is a v0.3
deliverable and has not been started.

This README records what each suite **must** cover, drawn from
[§16 Reference Implementation and Conformance](../spec/16-reference-implementation-and-conformance.md)
of the v0.2 specification. Where the specification does not state a threshold,
tolerance, or procedure, that gap is marked `TODO(spec)` rather than filled in.

---

## Conformance classes

The specification defines three conformance classes (§16.1). An implementation
MUST declare its class in its Attestation messages, or (for Class A, which does
not transmit Attestation) in its PerceptionSummary vendor-extension field.

| Class | Capability (per §16.1) | Use case |
| --- | --- | --- |
| **Class A (Baseline)** | Transmits and receives PerceptionSummary and Intent. Does not transmit Attestation. Cannot participate in C-ACC Groups as leader. | Aftermarket retrofit; low-cost OEM integrations; VRU beacons. |
| **Class B (Full)** | All of Class A, plus Attestation, GroupState, and C-ACC Group participation (leader or follower). Required for Slipstream Mode. | OEM integrations on vehicles with HSM and automotive Ethernet. |
| **Class I (Infrastructure)** | All of Class B, plus support for MAP, SPaT, and environment messages. Extended relevance horizons (up to 500 m). | Roadside units maintained by road authorities and operators. |

> **Naming note.** The classes are **A**, **B**, and **I**. There is no "Class C"
> in the v0.2 specification. Class I is the infrastructure class, and it is the
> only class permitted the 500 m relevance horizon (§7.4 additionally requires the
> RSU to substantiate that horizon in its conformance report).

Each suite below must be runnable per-class. A Class A implementation is not
expected to pass Attestation, GroupState, or Slipstream assertions; the harness
must skip rather than fail those, and must record the declared class in the
report.

---

## Test categories

§16.3 enumerates **seven** categories, not five. All seven are stubbed here.

### 1. `conformance/wire_format/`

Encoded messages MUST decode bit-identically on the reference decoder.

Must cover: all five message classes against the schemas in
[`schemas/proto/ocps/v2/`](../schemas/proto/ocps/v2/); canonical protobuf
deterministic serialization (protobuf-3.21 or later, per §8); round-trip
encode/decode identity; the `OcpsHeader` field set including `version` = 2;
object-taxonomy class IDs `0x00`–`0x0A`, the `0x0B`–`0xFE` RESERVED range, and
`0xFF` VENDOR_EXTENSION with a required vendor OID (§7.2); peers ignoring
unrecognized vendor OIDs; message-size envelopes from §8.2.

`TODO(spec)`: the whitepaper does not define the reference decoder's canonical
byte-ordering test vectors, nor a normative corpus of encoded messages. These
must be produced as part of the v0.3 schema deliverable.

### 2. `conformance/cryptographic/`

Signatures MUST validate against the reference verifier using SCMS or C-ITS test
PKI credentials.

Must cover: ECDSA over P-256 signature verification (MUST) and, where claimed,
Ed25519 (MAY) per IEEE 1609.2-2022 and ETSI TS 103 097 v2.1.1 (§3.2); signature
coverage over the correct field ranges (e.g. PerceptionSummary field 15 signs
fields 1–4, per Appendix A.2); pseudonym certificate structure, rotation, and
enrollment (§9.1.1–§9.1.3); rejection of malformed and expired credentials.

`TODO(spec)`: the whitepaper names SCMS and C-ITS test PKI but does not identify
which test PKI instance the reference verifier is provisioned against.

### 3. `conformance/timing/`

Rates, latencies, and lifetimes MUST be observed within the tolerances specified
in §8.2.

Must cover: per-class rate ceilings and size envelopes from the §8.2 table
(PerceptionSummary 10–20 Hz / 200–500 B; Intent 2–10 Hz / 60–120 B; GroupState
1–5 Hz / 120–240 B; Attestation 0.1–0.5 Hz / 180–320 B; Telemetry ≤ 0.1 Hz /
≤ 1 KB); PC5 PPPP and ITS-G5 access-category assignment; `lifetime_ms` expiry
handling; the §3.3 design goal of 100 ms peer-visible latency at p99 under
nominal channel conditions.

`TODO(spec)`: §8.2 states the rate and size envelopes but does not state the
*tolerance* around them that §16.3 refers to. The permitted deviation is
undefined in v0.2.

### 4. `conformance/behavioral/`

State transitions for C-ACC Group formation, handoff, and dissolution MUST match
the specified state machine.

Must cover: the C-ACC Group state machine and its `GroupMode` values
(FORMING / STABLE / HANDOFF / DISSOLVING, per Appendix A.4); formation; driver
authorization gating; leader election and handoff; dissolution; safety envelopes.

> **Cross-reference discrepancy.** §16.3 cites "the state machine of §13.1.1",
> but in the v0.2 document the C-ACC state machine is at **§12.1.1**, while §13 is
> Vehicle Integration Interfaces. The reference is preserved as written in the
> spec text; the tests should target §12.1.1. See
> [`docs/conversion-notes.md`](../docs/conversion-notes.md).

### 5. `conformance/safety/`

Fail-safe transitions MUST occur under the specified trigger conditions and
complete within 2 seconds.

Must cover: the fail-safe state and its entry triggers (§14.2); the 2-second
completion bound stated in §16.3; graceful degradation, meaning loss of
cooperative messages MUST result in a return to local-sensor-only behavior, not unsafe
actuation (§3.3); ASIL allocation per message class as assigned in §8.2
(PerceptionSummary ASIL B, Intent ASIL C, GroupState ASIL C, Attestation QM,
Telemetry QM); the hazards and mitigations enumerated in §14.1.

### 6. `conformance/privacy/`

Emitted messages MUST NOT contain prohibited fields, as verified by automated
scan.

Must cover: the prohibited-field scan; data minimization on the wire; retention
limits per data type at the receiver; opt-in gating for Telemetry; prohibited
analyses (§10.1–§10.5).

> **Cross-reference discrepancy.** §16.3 cites "the prohibited fields of §11.1",
> but the Privacy Model is **§10** (§10.1 is Data Minimization on the Wire) and
> §11.1 is Fusion Inputs. Preserved as written; the tests should target §10.

### 7. `conformance/misbehavior/`

All four detectors MUST be demonstrated against the reference adversarial PCAPs.

Must cover: the four misbehavior detectors of §9.3; misbehavior reporting;
revocation handling including CRL processing (§9.4); the adversarial-conditions
deployment scenario (§15.5); consensus-comparison detection of systematic
confidence miscalibration (§7.3).

`TODO(spec)`: the reference adversarial PCAP corpus does not exist yet. It is a
v0.3 deliverable and is tracked as
[issue #3](https://github.com/Locke-Werks/Strynex/issues/3).

---

## Supporting directories

- **`corpora/pcap/`**: Recorded PCAPs. Per §16.2 the reference implementation is
  to ship PCAPs covering the five deployment scenarios of §15 (urban signalized
  intersection, limited-access highway platoon, construction zone, VRU proximity,
  adversarial conditions), plus the adversarial corpus required by the
  misbehavior suite. **Empty.**
- **`corpora/vectors/`**: Known-answer test vectors for wire-format and
  cryptographic conformance. **Empty.**
- **`harness/`**: The Python test harness named in §16.2 and scheduled for v0.3.
  **Empty; not started.**

## Ground rules for filling these in

1. Every assertion must cite the specification section it enforces.
2. Where v0.2 is silent, leave a `TODO(spec)` and raise it against v0.3. Do not
   invent thresholds, tolerances, or pass criteria.
3. The whitepaper in `reference/` is the source of truth. If a test and the spec
   disagree, the spec wins or the spec gets amended: the test is never quietly
   adjusted to match an implementation.
