# Strynex / O-CPS

**O-CPS** (Open Cooperative Perception System) is an open, manufacturer-agnostic
cooperative perception and control layer for road vehicles and roadside
infrastructure. **Strynex** is the reference implementation and vehicle-facing
brand for an O-CPS stack.

The two names are deliberately distinct. O-CPS is the open specification that any
automaker, supplier, or municipality may implement without royalty. Strynex is
one concrete implementation plus a set of optional application-layer overlays
(notably Slipstream Mode). A vehicle may advertise O-CPS conformance without
implementing any Strynex-branded overlay, and the protocol MUST continue to
interoperate correctly.

O-CPS reuses the existing V2X radio, network, and security standards (C-V2X PC5
/ NR-V2X sidelink, ITS-G5, SAE J2735, SAE J3161, ETSI TS 103 324, IEEE 1609.2,
ETSI TS 103 097) and adds the three primitives that ecosystem leaves
underspecified:

1. A **normalized perception profile** that lets vehicles with radically
   different sensor suites exchange a common world model.
2. A **fusion and trust framework** that produces calibrated confidence in
   multi-source object tracks.
3. A **C-ACC Group Protocol** that lets driver-authorized platoons form, persist,
   hand off leadership, and dissolve without cloud dependence.

It defines no new radio layers, no new cryptographic primitives, and no new
market structures.

## Positioning: a standards and influence play, not a product

Quoting the abstract directly:

> O-CPS is a standards and influence play, not a product. It is published in the
> open, with a permissive license posture, to give automakers, suppliers,
> municipalities, and fleet operators a shared target to converge on rather than
> a vendor-specific lock-in.

That framing drives everything in this repository. The deliverable is a
specification other people can implement, a schema bundle they can compile
against, and a conformance suite they can be measured by. Adoption is the win
condition. The v1.0 milestone explicitly ends in **transfer of governance to an
established standards body**. This is not a codebase being groomed for a moat.

## Current status

- **v0.2: published.** The full draft specification, dated April 2026, is in
  [`spec/`](spec/), converted verbatim from the source document in
  [`reference/`](reference/).
- **v0.3: open.** None of its three deliverables is complete: the machine-readable
  schemas, the Python test harness, and the adversarial PCAP corpus. Target
  Q3 2026. See [`ROADMAP.md`](ROADMAP.md) and the repository issues.

The protobuf under [`schemas/`](schemas/) is a first extraction of Appendix A into
compiling files. It is not yet the finished v0.3 schema deliverable. See the
issue for what remains.

Prerequisite work for v0.3 has landed and is not itself a deliverable:

- [`docs/errata-v0.2.md`](docs/errata-v0.2.md) records eight verified defects in
  v0.2, each with proposed replacement text. None is adopted. One is rated
  critical: the trust arithmetic of §11.3 and §11.4 currently prevents a VRU
  beacon track from raising any advisory, which defeats the §15.4 scenario.
- [`profiles/`](profiles/) supplies the extension mechanism §6.6 permits and
  never defines, and reserves enum code points for all planned behaviors. §19
  freezes the wire format at v1.0, so ranges not reserved before then cannot be
  reserved after.
- [`schemas/proposed/v0.3/`](schemas/proposed/v0.3/) holds the additive schema
  changes those two imply, including encodings for `Telemetry.payload`, which
  v0.2 leaves undefined for all four `TelemetryKind` values and which the PCAP
  corpus cannot be recorded without.

Two things follow that are not yet decided, both tracked as errata. §16.2 defines
the reference implementation as four components including a C reference codec
that appears in no milestone, and §16.3's first conformance category needs it.
And §14.3 applies the ISO 26262 lifecycle with no scoping by conformance class,
which puts ASIL C on the cheapest class in the specification.

## Repository layout

```
spec/                  The v0.2 specification, split per section
  README.md            Section index and conversion notes
  00-abstract.md ...   Per-section files, numbered in reading order
  OCPS_v0.2_full.md    Unsplit conversion, authoritative if the split disagrees
  LICENSE              CC BY 4.0, applies to spec/ and reference/
reference/             Source documents, verbatim
  OCPS_Whitepaper_v0.2.docx
  OCPS_Whitepaper_v0.2.pdf
schemas/proto/ocps/v2/ Normative protobuf from Appendix A, package ocps.v2
  common.proto         OcpsHeader, Pose, Dimensions          (A.1)
  perception.proto     PerceivedObject, PerceptionSummary    (A.2)
  intent.proto         IntentCode, Intent                    (A.3)
  group.proto          GroupMode, GroupState                 (A.4)
  attestation.proto    SensorHealth, Attestation             (A.5)
  telemetry.proto      TelemetryKind, Telemetry              (A.6)
schemas/proposed/v0.3/ Proposed v0.3 schema additions. NOT normative.
  README.md            What is proposed, the diffs against v2, and verification
  proto/ocps/v2/       ExtensionBlock, ConformanceDeclaration, Telemetry payloads
profiles/              Extension profile registry and profile specifications
  EP-0000-...md        The framework: ID ranges, code points, safe-ignore rules
  README.md            Registry, overlay map, and all code point allocations
  registry.yaml        Machine-readable allocations. Source of truth.
  EP-0001-...md        Intersection Assist, owed by §12.3 and roadmap v0.4
  EP-0002-...md        Merge Negotiation, closes a v0.2 gap around IntentCode MERGE
tests/                 Conformance suite scaffolding, stubs only, nothing passes
  README.md            What each of the seven suites must cover
docs/
  conversion-notes.md  How spec/ was produced, verified, and what was left uncorrected
  errata-v0.2.md       Eight verified v0.2 defects with proposed replacement text
  open-questions.md    Undecided questions. Not decisions.
ROADMAP.md             v0.3 through v1.0
LICENSE                Apache 2.0, applies to everything outside spec/ and reference/
```

## Licensing

The repository is deliberately split, matching §18 of the specification:

| Path | License | SPDX |
| --- | --- | --- |
| `spec/`, `reference/` | Creative Commons Attribution 4.0 International | `CC-BY-4.0` |
| Everything else (`schemas/`, `tests/`, reference implementation) | Apache License 2.0 | `Apache-2.0` |

The reasoning is the standard one for a specification that wants to be adopted.
The **document** is CC BY 4.0 so anyone can quote it, translate it, excerpt it
into their own standards work, or fold it into a national profile, as long as
they say where it came from. The **code and schemas** are Apache 2.0 because
implementers need an explicit patent grant and a warranty disclaimer that a
content license does not provide.

The full Apache 2.0 text is at [`LICENSE`](LICENSE); the full CC BY 4.0 text is
at [`spec/LICENSE`](spec/LICENSE).

Per §18.3, no patent claims are asserted against good-faith implementers.
"Strynex" is a trademark and is not licensed by either of the above; O-CPS
conformance claims do not require or imply permission to use it.

Copyright © 2026 Archon C. Locke, Specter Point Intelligence, LLC.
Colorado Springs, Colorado, USA.

## Working on this repository

The whitepaper in `reference/` is the source of truth. Two rules follow:

1. **Do not edit `spec/` to change the specification.** Those files are a
   mechanical conversion of the source document. Edit the source and re-convert,
   or the two drift apart silently.
2. **Where the spec is silent, mark a `TODO(spec)`.** Do not invent requirements,
   threat mitigations, tolerances, or conformance criteria to fill a gap. An
   explicit hole is a v0.3 agenda item; a quietly invented requirement is a bug
   that ships.
