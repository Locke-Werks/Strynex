# EP-0000: Extension Profile Framework

| | |
| --- | --- |
| **Profile ID** | 0x0000 (reserved; this document defines the mechanism, it is not itself a profile) |
| **Status** | Proposed for O-CPS v0.3 |
| **Category** | Core, open |
| **License** | CC BY 4.0, as specification text |
| **Base** | O-CPS v0.2, sections 6.6, 7.2, 8, 11, 13.1, 16.1, 18.4 |
| **Replaces** | Nothing |
| **Conformance** | Optional to implement. Mandatory to tolerate. See section 7. |

---

## 1. Motivation

Section 6.6 of the base specification permits extension profiles and says
nothing about how one comes to exist:

> Other behaviors MAY be defined by extension profiles and MUST NOT conflict
> with the normative requirements of this document.

There is no identifier space, no allocation procedure, no registry, no
declaration mechanism, and no rule telling a receiver what to do with a profile
it has never heard of. Section 12.3 then defers Intersection Assist "to a
forthcoming extension profile", so the base specification already owes at least
one profile against a mechanism that does not exist.

The scarcity this document addresses is narrow and specific. Extension profiles
do not need new message classes: the five classes of section 8 carry every
behavior currently contemplated. What they consume is **enum code points**.
`IntentCode` (Appendix A.3) ends at 11. `TelemetryKind` (Appendix A.6) ends at 4.
Neither declares a reserved range, an extension range, or an allocation policy.

Section 19 freezes the wire format at v1.0. Code point ranges that are not
reserved before that freeze cannot be reserved after it without a new wire
version, and after the section 18.4 governance transfer the allocation decision
belongs to a committee rather than to the editor of record. Reserving the ranges
is a v0.3 action whose cost is a paragraph and whose window closes permanently.

## 2. Terminology

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
RECOMMENDED, MAY, and OPTIONAL are to be interpreted as in the base
specification, section 2.

**Extension profile.** A specification that defines behavior on top of the base
O-CPS message set, identified by a Profile ID, and constrained by section 6 of
this document.

**Base message.** An O-CPS message as defined by section 8 and Appendix A of the
base specification, ignoring any extension content.

**Extension content.** Any code point outside the v0.2-assigned range, and any
`ExtensionBlock`, carried in a base message.

**Safely ignorable.** The property, required by section 7, that a receiver which
discards all extension content from a message and processes only the base
message is left in a correct and safe state.

## 3. Profile identifiers

A Profile ID is an unsigned 16-bit integer. The space is partitioned as follows,
and this partition is normative.

| Range | Count | Category | Allocation |
| --- | --- | --- | --- |
| `0x0000` | 1 | Reserved | Never allocated. Denotes "no profile". |
| `0x0001` to `0x00FF` | 255 | **Core** | Open profiles published under the base specification's license, allocated by the registry maintainer. Intended for behavior that belongs to the commons: infrastructure interaction, vulnerable road user protection, safety-of-life signalling. |
| `0x0100` to `0x0FFF` | 3840 | **Registered** | Third-party profiles. Allocated on request to any requester, first come first served, no fee, no review of technical merit. The profile document may carry any license, including none. The registry records the identifier, the owner, and a contact. |
| `0x1000` to `0x7FFF` | 28672 | **Vendor** | Allocated in contiguous blocks of 256 to a single holder, who administers the block privately. A holder MUST possess an IANA Private Enterprise Number or an ISO/ITU object identifier arc and MUST record it in the registry. Profile documents need not be published. |
| `0x8000` to `0xFFFF` | 32768 | **Experimental** | Unallocated and unregistered. MUST NOT be transmitted by a participant holding production SCMS or C-ITS credentials. Intended for laboratory work, interoperability events, and closed-track testing. Collision is expected and unmanaged. |

The Core and Registered ranges exist to make the mechanism credible to a
standards body: anyone can obtain an identifier, and obtaining one carries no
obligation to any vendor. The Vendor range exists so that proprietary
application-layer behavior can be carried on a public wire without the owner
disclosing it, which is what makes the base specification adoptable by
competitors simultaneously.

## 4. Enum code point ranges

The following partitions apply to the enumerations of Appendix A. Values already
assigned by v0.2 are **frozen** and are reproduced here for completeness only.

### 4.1 IntentCode (Appendix A.3)

| Range | Category | Notes |
| --- | --- | --- |
| 0 to 11 | v0.2 assigned, frozen | `INTENT_UNKNOWN` through `PLATOON_LEAVE`. |
| 12 to 63 | Core reserved | Future assignment in the base specification itself. Not allocated to profiles. |
| 64 to 191 | Profile allocated | Allocated in blocks of 8 to a Core or Registered profile. |
| 192 to 254 | Vendor private | Meaningful only within a Vendor profile block. MUST be accompanied by an `ExtensionBlock` naming the profile. |
| 255 | `INTENT_PROFILE_EXTENSION` | The intent is defined entirely by an accompanying `ExtensionBlock`. |

### 4.2 TelemetryKind (Appendix A.6)

| Range | Category | Notes |
| --- | --- | --- |
| 0 to 4 | v0.2 assigned, frozen | `TELEMETRY_UNKNOWN` through `HAZARD_FLAG`. |
| 5 to 31 | Core reserved | Future assignment in the base specification itself. |
| 32 to 127 | Profile allocated | Allocated in blocks of 8. |
| 128 to 254 | Vendor private | As above. |
| 255 | `TELEMETRY_PROFILE_EXTENSION` | Kind defined entirely by an accompanying `ExtensionBlock`. |

### 4.3 Object class ID (section 7.2)

Section 7.2 already reserves `0x0B` to `0xFE` "for future assignment by the O-CPS
registry" and defines `0xFF` as `VENDOR_EXTENSION` requiring a vendor OID. That
scheme is not replaced. It is sub-allocated:

| Range | Category |
| --- | --- |
| `0x00` to `0x0A` | v0.2 assigned, frozen |
| `0x0B` to `0x3F` | Core reserved, base specification assignment |
| `0x40` to `0xBF` | Profile allocated, blocks of 8 |
| `0xC0` to `0xFE` | Vendor private |
| `0xFF` | `VENDOR_EXTENSION`, unchanged from 7.2 |

Section 7.2's existing requirement stands: a receiver that does not recognise a
vendor OID MUST ignore the object. Section 7 of this document generalises that
behavior to every extension point.

### 4.4 Block allocation

Blocks are allocated contiguously and are never reused. A profile that exhausts
its block requests a second block; it does not receive an extension of the
first. Retiring a profile does not return its block to the pool. This costs
address space, which is abundant, and buys the property that a code point
observed on the wire has exactly one meaning for the lifetime of the protocol,
which is not.

## 5. The ExtensionBlock

Extension content is carried in a single generalised container rather than in
per-message vendor fields.

```protobuf
// Proposed addition to package ocps.v2. Field addition is wire-compatible;
// OcpsHeader.version remains 2.
message ExtensionBlock {
  uint32 profile_id      = 1;  // 16-bit, section 3
  uint32 profile_version = 2;  // profile-defined, monotonic
  bytes  payload         = 3;  // profile-defined encoding
  uint32 flags           = 4;  // bit 0: CRITICAL (see section 7.2). Others reserved, MUST be 0.
}
```

`ExtensionBlock` is carried as a repeated field on `PerceptionSummary`, `Intent`,
`GroupState`, and `Telemetry`. It is **not** carried on `Attestation`: attestation
content is the input to the trust weighting of section 11.3, and permitting
profile-defined material there would let a profile influence its own trust.

Extension blocks MUST be inside the signature scope of the carrying message. A
message carrying an `ExtensionBlock` outside the signed range MUST be rejected.

The combined size of all extension content MUST NOT push the carrying message
beyond the per-class size envelope of section 8.2. Where a profile cannot fit,
the profile is at fault, not the envelope.

## 6. Constraints on profiles

These constraints are the reason this framework can be published openly without
weakening the base specification. A profile that violates any of them is not an
O-CPS extension profile, whatever it calls itself, and a conformant participant
MUST NOT implement it while claiming O-CPS conformance.

A profile MUST NOT:

1. **Define a new message class.** The five classes of section 8 are closed. A
   profile that needs a sixth is a change to the base specification and goes
   through section 18.4, not through this registry.
2. **Raise the ASIL allocation of any message class** above the allocation
   assigned by section 8.2. A profile may need less integrity than the class it
   rides on. It may never need more, because the receiving implementation was
   developed to the class allocation and does not know the profile exists.
3. **Lower the advisory threshold of section 11.4**, or any other numeric safety
   bound in section 11, 12, or 14. Changing a safety threshold is an amendment to
   the base specification. E-03 in [`../docs/errata-v0.2.md`](../docs/errata-v0.2.md)
   is an example of how such a change is made: argued, bounded, and adopted into
   the base text, not asserted by a profile.
4. **Create a control path that bypasses section 13.1.** All advisory output
   still passes through the vehicle's existing ADAS domain controller, which
   retains actuation authority. A profile grants no new authority to anything.
5. **Extend retention beyond section 10.3**, introduce any persistent identifier
   prohibited by section 10.1, or enable any analysis prohibited by section 10.5.
6. **Require cloud connectivity in a safety-relevant path.** Section 3.2 makes
   cloud interfaces optional and forbids them in safety-critical loops. A profile
   inherits that prohibition.
7. **Make its own support mandatory for base conformance.** No profile is ever a
   prerequisite for a section 16.1 conformance class.

A profile MUST:

8. **Declare the minimum conformance class** (16.1) it requires of a
   participant that implements it.
9. **Declare the maximum ASIL** of any behavior it introduces, bounded by
   constraint 2.
10. **State its safe-ignore behavior explicitly**, satisfying section 7.
11. **State its privacy analysis** against section 10, naming what it puts on the
    wire and for how long a receiver may hold it.

## 7. Receiver obligations

### 7.1 Safe ignore

This is the central requirement of the framework.

Every extension profile MUST be **safely ignorable**. A receiver that discards
all extension content and processes only the base message MUST be left in a
correct and safe state. Concretely:

- A receiver encountering an unrecognised Profile ID MUST ignore that
  `ExtensionBlock` and MUST continue to process the rest of the message
  normally. It MUST NOT reject the message.
- A receiver encountering an unrecognised code point in an extension range of
  section 4 MUST treat it as the `UNKNOWN` value of that enumeration
  (`INTENT_UNKNOWN`, `TELEMETRY_UNKNOWN`, class `0x00`), and MUST NOT reject the
  message.
- Unrecognised extension content MUST NOT trigger any misbehavior detector of
  section 9.3. Extension content a receiver cannot parse is not evidence of
  anything.
- Unrecognised extension content MUST NOT influence the trust weighting of
  section 11.3.

The consequence is that base interoperability is unconditional. Two conformant
participants that share no profiles still exchange perception, intent, group
state and telemetry correctly. Profiles can never fragment the population.

### 7.2 The CRITICAL flag

Bit 0 of `ExtensionBlock.flags` marks a block whose omission changes the meaning
of the base message rather than merely adding to it.

A receiver that does not recognise the Profile ID of a block with CRITICAL set
MUST ignore the entire carrying message, and MUST NOT count that message toward
any rate, freshness, or consensus calculation.

CRITICAL exists because a small number of behaviors genuinely cannot degrade
safely, and it is better to name them than to have a profile author pretend
otherwise. It is expected to be rare. A Core profile setting CRITICAL SHOULD
justify it in its safe-ignore statement, and a profile that sets CRITICAL on
every block has almost certainly been designed wrong.

CRITICAL MUST NOT be set on a block carried in a `Telemetry` message. Telemetry
is QM per section 8.2 and nothing in it is permitted to be load-bearing.

### 7.3 Declaration

A participant declares the profiles it supports:

- **Class B and Class I**: in the `Attestation` message.
- **Class A**: in the `ConformanceDeclaration` field of `PerceptionSummary`
  proposed by E-01 in [`../docs/errata-v0.2.md`](../docs/errata-v0.2.md). EP-0000
  therefore depends on E-01 being adopted; without it, Class A participants have
  no way to declare profile support.

Declaration is advisory. A participant MUST NOT assume a peer supports a profile
merely because it declared it, and MUST NOT refuse base interoperation with a
peer that declares nothing.

There is no negotiation handshake. Profiles are announced, not agreed. Any
profile requiring bilateral agreement implements that agreement in its own
payload, using the Intent message class, exactly as the C-ACC Group Protocol of
section 12.1 does.

## 8. Registry operation

The registry is [`README.md`](README.md) in this directory, with the
machine-readable allocations in [`registry.yaml`](registry.yaml).

**Allocation procedure.** Open an issue against the public repository naming the
requested category, a one-line description, an owner, and a contact. Core
allocations require the editor of record to agree the behavior belongs in the
commons. Registered and Vendor allocations are granted on request without
technical review.

**Permanence.** An allocation is permanent. Identifiers are never reassigned,
including after a profile is withdrawn, abandoned, or superseded. A withdrawn
profile is marked withdrawn in the registry and its identifier stays burned.

**No revocation.** The registry maintainer cannot revoke an allocation. The
maintainer may mark a profile as non-conformant with section 6, which is a
statement about the profile and not a reclamation of the code point.

**Survival across the governance transfer.** Section 18.4 contemplates
transferring governance to an established standards body once three independent
conformant implementations are in public deployment. The instrument of transfer
MUST carry the registry forward intact, and MUST preserve:

- every allocation made before the transfer, in every range;
- the partition of section 3, in particular the Vendor range, which exists so
  that holders can rely on privately administered blocks across a change of
  maintainer;
- the constraints of section 6, which are what make the Vendor range safe to
  grant.

This clause is the reason the framework belongs in v0.3 rather than later. After
the transfer, a Vendor block is a request to a committee. Before it, it is a
line in a file.

## 9. Conformance

EP-0000 is optional to implement and mandatory to tolerate.

A participant claiming conformance to any section 16.1 class MUST satisfy
section 7.1, whether or not it implements any profile. Safe ignore is not a
profile feature; it is a property of the base receiver, and it is what the rest
of this framework rests on.

Proposed additions to the section 16.3 conformance categories:

| Category | Assertion |
| --- | --- |
| Wire format | A message carrying an `ExtensionBlock` with an unallocated Profile ID decodes to the same base message as the identical message without it. |
| Wire format | A message carrying an out-of-range `IntentCode`, `TelemetryKind`, or `class_id` decodes with that field mapped to the corresponding `UNKNOWN` value, and does not error. |
| Behavioral | A receiver presented with an unrecognised non-CRITICAL block processes the carrying message. |
| Behavioral | A receiver presented with an unrecognised CRITICAL block discards the carrying message and does not count it toward rate or consensus. |
| Misbehavior | Unrecognised extension content, repeated at the section 8.2 rate ceiling, fires no detector of section 9.3. |
| Cryptographic | A message whose `ExtensionBlock` falls outside the signature scope is rejected. |
| Privacy | The prohibited-field scan of section 16.3 is applied to `ExtensionBlock.payload` as well as to base fields. |

The last row matters more than it looks. Without it, `ExtensionBlock.payload` is
an opaque byte string that defeats the automated privacy scan, and the extension
mechanism becomes the obvious place to smuggle a persistent identifier.
Profiles MUST therefore publish enough of their payload encoding for the scan to
run, including Vendor profiles, which may publish the encoding without publishing
the behavior.

## 10. Profile document template

A profile document carries, in this order: identifier and status table;
motivation; the code points it consumes; the payload encoding; behavior; the
declarations required by section 6 items 8 through 11; the safe-ignore
statement; conformance assertions; and its own security and privacy analysis.

[`EP-0001-intersection-assist.md`](EP-0001-intersection-assist.md) and
[`EP-0002-merge-negotiation.md`](EP-0002-merge-negotiation.md) are the worked
examples.

## 11. Open questions

Recorded, not answered. Consistent with `docs/open-questions.md`, nothing here
is a decision.

1. **Whether `ExtensionBlock` belongs on `GroupState`.** Group membership is
   ASIL C and leader election faults are a named hazard in section 14.1. A
   profile that extends group semantics is the most likely place for constraint
   2 of section 6 to be violated by accident. Arguments exist both ways and no
   Core profile currently needs it.
2. **Whether the Vendor range should require a published payload encoding at
   allocation time or only at first transmission on public roads.** Section 9
   requires publication for the privacy scan. Requiring it at allocation leaks a
   product roadmap; requiring it at deployment is harder to enforce.
3. **Whether Experimental-range transmission should be detectable** by
   conformant participants, so that an interoperability event can distinguish
   experimental traffic from misbehavior. A flag would help; it would also be
   trivially forged.
4. **How a profile is deprecated** when the base specification later assigns a
   Core code point covering the same behavior. Permanence forbids reclaiming the
   identifier, but nothing yet says which one an implementation should prefer.
