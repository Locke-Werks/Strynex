# O-CPS Message Schemas

Normative protobuf for the O-CPS v2 wire format, package `ocps.v2`, extracted
from [Appendix A](../spec/21-appendix-a-message-schemas.md) of the v0.2
specification.

Per §8, all messages share a common header and are encoded with Canonical
Protocol Buffers, schema version 3, using the deterministic serialization rules
of protobuf-3.21 or later. Use of protobuf is not a commitment to Google's
implementation; any conformant encoder/decoder is acceptable.

## Files

| File | Contents | Appendix |
| --- | --- | --- |
| [`proto/ocps/v2/common.proto`](proto/ocps/v2/common.proto) | `OcpsHeader`, `Pose`, `Dimensions` | A.1 |
| [`proto/ocps/v2/perception.proto`](proto/ocps/v2/perception.proto) | `PerceivedObject`, `PerceptionSummary` | A.2 |
| [`proto/ocps/v2/intent.proto`](proto/ocps/v2/intent.proto) | `IntentCode`, `Intent` | A.3 |
| [`proto/ocps/v2/group.proto`](proto/ocps/v2/group.proto) | `GroupMode`, `GroupState` | A.4 |
| [`proto/ocps/v2/attestation.proto`](proto/ocps/v2/attestation.proto) | `SensorHealth`, `Attestation` | A.5 |
| [`proto/ocps/v2/telemetry.proto`](proto/ocps/v2/telemetry.proto) | `TelemetryKind`, `Telemetry` | A.6 |

## Compiling

```sh
mkdir -p schemas/gen
protoc -I schemas/proto --python_out=schemas/gen schemas/proto/ocps/v2/*.proto
```

Generated bindings are build artifacts and are gitignored. `schemas/proto` is the
source of truth.

**Verified:** all six files compile cleanly under `libprotoc 35.1` with no errors
or warnings.

## Fidelity

Message and field definitions are reproduced verbatim from Appendix A. The only
changes are mechanical, and each file says so in its header:

1. `syntax`, `package`, `option`, and `import` statements added or relocated so
   each file compiles independently. Appendix A declares `syntax` and `package`
   once, inside A.1.
2. The definitions split across six files, one per Appendix A subsection.
3. Unit and constraint comments added where the whitepaper states the unit in
   prose but not in the listing. Added comments are tagged `[spec §N]` citing the
   section that states them. Every untagged comment is verbatim from Appendix A.
   Where the whitepaper states no unit, none was invented.

No field was added, removed, renamed, or renumbered. See
[`docs/conversion-notes.md`](../docs/conversion-notes.md), including the note on
fields sometimes attributed to the object container that v0.2 does not actually
define.

## Known gaps for v0.3

These are constraints the specification states in prose that the schema does not
currently enforce, and no decision has been made about whether they belong in the
schema or in the conformance harness:

- `OcpsHeader.version` MUST be 2 (§8.1).
- `OcpsHeader.cert_id` is 8 bytes; `GroupState.group_id` and `Intent.group_id`
  are 16 bytes; `GroupState.members` entries are 8 bytes each.
- `confidence` is bounded 0-255 with banded semantics (§7.3).
- `class_id` values 0x0B-0xFE are RESERVED; 0xFF requires a vendor OID (§7.2).
- Per-class message size envelopes (§8.2).
- Canonical serialization test vectors do not exist yet.

`TODO(spec)`: v0.2 does not state whether these are schema-level or
harness-level obligations.

Licensed under Apache 2.0. See [`../LICENSE`](../LICENSE).
