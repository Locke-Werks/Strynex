# Proposed schema additions for O-CPS v0.3

**Nothing here is normative.** This directory holds the schema work proposed for
the v0.3 milestone. It is separate from [`../../proto/`](../../proto/) because
that tree is a verbatim extraction of Appendix A of the v0.2 specification, and
[`../../../docs/conversion-notes.md`](../../../docs/conversion-notes.md) records
that no field there was "added, removed, renamed, or renumbered." Adding
anything to those files would break that guarantee silently. So proposals live
here until adopted, and adoption means editing
`reference/OCPS_Whitepaper_v0.2.docx` and re-running the conversion.

Everything proposed is **additive and wire-compatible**. `OcpsHeader.version`
stays at 2. Protobuf field addition does not break an existing decoder, and
proto3 open-enum semantics mean an unrecognised enum value round-trips intact
rather than erroring, which is verified below.

## What is here

| File | Contents | Source |
| --- | --- | --- |
| [`proto/ocps/v2/extension.proto`](proto/ocps/v2/extension.proto) | `ExtensionBlock`, `ConformanceDeclaration` | EP-0000 section 5; erratum E-01 |
| [`proto/ocps/v2/telemetry_payloads.proto`](proto/ocps/v2/telemetry_payloads.proto) | `GridCell`, `RoadSurfaceFriction`, `WeatherEstimate`, `LocalCongestion`, `HazardFlag` and their enums | Appendix A.6 `Telemetry.payload`, currently undefined |
| [`proto/ocps/v2/ep0001.proto`](proto/ocps/v2/ep0001.proto) | `IntersectionApproach`, `StopIntent` | [EP-0001](../../../profiles/EP-0001-intersection-assist.md) |
| [`proto/ocps/v2/ep0002.proto`](proto/ocps/v2/ep0002.proto) | `MergeNegotiation`, `RefuseReason` | [EP-0002](../../../profiles/EP-0002-merge-negotiation.md) |

## Why the telemetry payloads matter more than they look

Appendix A.6 defines `Telemetry.payload` as `bytes` with the comment
`kind-specific encoding`. No section of v0.2 defines that encoding for any of
the four assigned `TelemetryKind` values. Three consequences follow, and the
third is the one that blocks the milestone.

1. Two conformant implementations transmitting `ROAD_SURFACE_FRICTION` share
   nothing but the kind number.
2. The automated privacy scan required by section 16.3 cannot inspect an opaque
   byte string, so `Telemetry` is the one message class where a prohibited field
   could ship undetected.
3. **The PCAP corpus cannot be recorded.** Section 16.2 requires recorded PCAPs
   covering the five deployment scenarios of section 15. The construction zone
   (15.3) and adversarial conditions (15.5) scenarios both carry telemetry, and
   there is nothing to encode.

Deliverable 3 of v0.3 is therefore blocked on a definition that is not itself a
listed deliverable.

## Amendments that cannot be expressed as new files

Protobuf has no mechanism to add a field to a message defined in another file.
The following are diffs against `../../proto/ocps/v2/`, to be applied when
adopted. They are not applied here, and the files in this directory do not
duplicate any message defined there.

### `perception.proto`

```diff
 message PerceptionSummary {
   OcpsHeader header     = 1;
   Pose       own_pose   = 2;
   Dimensions own_dims   = 3;
   repeated PerceivedObject objects = 4;
+  // E-01: required for Class A, which does not transmit Attestation and so has
+  // no other route to the declaration section 16.1 requires of it.
+  ConformanceDeclaration conformance = 5;
+  // EP-0000 section 5.
+  repeated ExtensionBlock extensions = 6;
-  bytes signature       = 15; // ECDSA-P256 over fields 1-4
+  bytes signature       = 15; // ECDSA-P256 over fields 1-6
 }
```

The signature scope change is not optional. A `ConformanceDeclaration` outside
the signed range can be stripped or forged in flight, which turns E-01's fix
into a trust-downgrade attack: an attacker rewrites a Class B peer's
declaration and moves it to `w_attest = 0.5`, or removes a Class A peer's
declaration and moves it to 0.2.

### `intent.proto`

```diff
 enum IntentCode {
   INTENT_UNKNOWN       = 0;
   ...
   PLATOON_LEAVE        = 11;
+  // 12-63    core reserved, base specification assignment
+  // 64-191   profile allocated in blocks of 8, see profiles/registry.yaml
+  // 192-254  vendor private
+  INTENT_PROFILE_EXTENSION = 255;  // meaning defined entirely by ExtensionBlock
 }

 message Intent {
   ...
   bytes      group_id       = 6;
+  repeated ExtensionBlock extensions = 7;
   bytes      signature      = 15;
 }
```

### `telemetry.proto`

```diff
 enum TelemetryKind {
   TELEMETRY_UNKNOWN     = 0;
   ...
   HAZARD_FLAG           = 4;
+  // 5-31     core reserved, base specification assignment
+  // 32-127   profile allocated in blocks of 8
+  // 128-254  vendor private
+  TELEMETRY_PROFILE_EXTENSION = 255;
 }

 message Telemetry {
   OcpsHeader    header  = 1;
   TelemetryKind kind    = 2;
-  bytes         payload = 3;   // kind-specific encoding
+  bytes         payload = 3;   // kind-specific encoding, see telemetry_payloads.proto
+  repeated ExtensionBlock extensions = 4;
   bytes         signature = 15;
 }
```

### `group.proto`

```diff
 message GroupState {
   ...
   uint64     epoch       = 8;
+  repeated ExtensionBlock extensions = 9;
   bytes      signature   = 15;
 }
```

Adding `extensions` to `GroupState` is listed as
[EP-0000 open question 1](../../../profiles/EP-0000-extension-profile-framework.md#11-open-questions)
and should not be adopted without deciding it. `GroupState` is ASIL C and
leader-election faults are a named hazard in section 14.1, which makes it the
most likely place for a profile to violate EP-0000 section 6 constraint 2 by
accident. No Core profile currently needs it.

`attestation.proto` is deliberately unamended. Attestation content feeds the
trust weighting of section 11.3, and permitting profile-defined material there
would let a profile influence its own trust.

## Verification

All four files compile cleanly under `libprotoc 35.1`, matching the toolchain
recorded in `conversion-notes.md`, in three configurations:

```sh
python -m pip install grpcio-tools

# 1. existing v2 tree alone, regression check
python -m grpc_tools.protoc -I schemas/proto \
  --python_out=/tmp/out schemas/proto/ocps/v2/*.proto

# 2. proposed tree alone
python -m grpc_tools.protoc -I schemas/proposed/v0.3/proto \
  --python_out=/tmp/out schemas/proposed/v0.3/proto/ocps/v2/*.proto

# 3. both together, which is what an implementer does
python -m grpc_tools.protoc -I schemas/proto -I schemas/proposed/v0.3/proto \
  --python_out=/tmp/out \
  schemas/proto/ocps/v2/*.proto schemas/proposed/v0.3/proto/ocps/v2/*.proto
```

No errors, no warnings, no name collisions. Profile payloads are sub-packaged
as `ocps.v2.ep0001` and `ocps.v2.ep0002` so that profiles cannot collide with
each other or with the base message set.

### Measured, not assumed

The safe-ignore rule of EP-0000 section 7.1 rests on a claim about proto3
enum behavior. It was tested rather than asserted:

| Check | Result |
| --- | --- |
| `Intent` with unallocated `code = 99` | Decodes without error. `code` reads back as 99. Round-trip byte-identical. `confidence` and header intact. |
| `Telemetry` with vendor-private `kind = 200` | Same. Payload preserved. |
| `ConformanceDeclaration`, Class A, two profiles | 6 bytes |
| `ExtensionBlock`, unknown profile, 6-byte payload | 14 bytes |
| `PerceptionSummary`, 8 objects, 64-byte signature | 309 bytes, inside the section 8.2 envelope of 200 to 500 bytes |

The first two rows confirm that proto3 open-enum semantics give safe-ignore for
free at the wire layer. Mapping an unrecognised value to `UNKNOWN` is therefore
an **application-layer obligation**, not something the decoder does. That
distinction belongs in the harness: a decoder that errors on code 99 is broken,
and an application that treats code 99 as meaningful is also broken, and only
the second is detectable from the wire.

The last row matters for adoption. A typical `PerceptionSummary` leaves roughly
190 bytes of headroom inside the envelope, so a `ConformanceDeclaration` at 6
bytes and a small `ExtensionBlock` fit without pushing the message over. A
profile that needs more than that headroom is at fault, per EP-0000 section 5.

These five rows are the first concrete conformance vectors this repository has
produced. They belong in `tests/corpora/vectors/` when the harness exists.

## Bearing on the open v0.3 constraint decisions

[`../../README.md`](../../README.md) lists constraints v0.2 states in prose that
the schema does not enforce, with `TODO(spec)` on whether they are schema-level
or harness-level obligations. This work bears on two of them:

- **`confidence` bounded 0 to 255 with banded semantics (7.3).** The telemetry
  payloads reuse those bands directly and add a normative constraint of their
  own: a `DERIVED_TEMPERATURE_MODEL` friction estimate MUST NOT exceed MEDIUM,
  because it is inference rather than observation. That is a per-field rule a
  schema cannot express, which is evidence for putting the whole class of
  confidence constraints in the harness.
- **Per-class message size envelopes (8.2).** Measured above. An envelope is a
  property of an encoded message, not of a schema, so this one can only be
  harness-level.

Neither is decided here. Both are v0.3 decisions and this is input to them.

## License

Apache 2.0, matching [`../../../LICENSE`](../../../LICENSE) and the rest of
`schemas/`.
