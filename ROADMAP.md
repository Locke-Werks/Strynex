# O-CPS Roadmap

Taken from [§19 Roadmap](spec/19-roadmap.md) of the v0.2 specification, which
states: "The following milestones define the path from the v0.2 draft to a stable
v1.0 release."

| Milestone | Deliverables | Target |
| --- | --- | --- |
| v0.3 (Interim) | Complete Appendix A schemas in machine-readable protobuf; reference Python test harness; adversarial PCAP corpus. | Q3 2026 |
| v0.4 (Interop) | First public interoperability event with at least two independent implementations; Intersection Assist full specification. | Q1 2027 |
| v0.5 (Infrastructure) | Class I (infrastructure) profile finalized; RSU reference deployment with two participating road authorities. | Q3 2027 |
| v0.6 (Safety) | ISO 26262 safety case template published; independent safety audit of the reference implementation. | Q1 2028 |
| v1.0 (Stable) | Frozen wire format; transfer of governance to an established standards body; public conformance registry. | Q4 2028 |

## Current milestone: v0.3 (Interim), Q3 2026

**v0.3 is the current open milestone. It is not started.** No deliverable is in
progress; nothing has been shipped against it. The three items below are tracked
as individual issues.

### 1. Complete Appendix A schemas in machine-readable protobuf

Partially advanced, not done. [`schemas/proto/ocps/v2/`](schemas/proto/ocps/v2/)
contains all six Appendix A schema groups extracted into files that compile under
`protoc`. What v0.3 still owes: canonical serialization test vectors, a decision
on whether the spec's implicit constraints (`cert_id` 8 bytes, `group_id` 16
bytes, `version` MUST be 2, `confidence` 0–255) are enforced in-schema or in the
harness, and a versioning and release process for the bundle.

### 2. Reference Python test harness

Not started. [`tests/`](tests/) is directory scaffolding and a README describing
what each of the seven §16.3 conformance categories must cover. No harness code
exists. Several tolerances the harness needs are undefined in v0.2 and are marked
`TODO(spec)` in [`tests/README.md`](tests/README.md); resolving them is part of
this milestone, not a separate one.

### 3. Adversarial PCAP corpus

Not started. `tests/corpora/pcap/` is empty. §16.3 requires that all four
misbehavior detectors of §9.3 be demonstrated against reference adversarial
PCAPs, and §16.2 says the reference implementation ships PCAPs covering the five
deployment scenarios of §15. Neither set exists.

## Note on later milestones

v0.4 through v1.0 are recorded here exactly as the whitepaper states them. They
have not been scoped beyond the one-line deliverable descriptions in the §19
table, and nothing in this repository elaborates on them. Do not treat the
absence of detail as a gap to be filled speculatively.
