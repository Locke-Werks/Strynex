# O-CPS Extension Profile Registry

Extension profiles for O-CPS, and the allocation of the code points they
consume.

Section 6.6 of the base specification permits extension profiles without saying
how one comes to exist, and section 12.3 then defers Intersection Assist to "a
forthcoming extension profile". This directory is the mechanism that debt is
owed against.

**Start with [EP-0000](EP-0000-extension-profile-framework.md).** It defines
identifier ranges, code point partitions, the `ExtensionBlock` carrier, the
constraints every profile must satisfy, and the safe-ignore rule that keeps
profiles from fragmenting the population. Nothing else in this directory makes
sense without it.

Machine-readable allocations are in [`registry.yaml`](registry.yaml). That file
is the source of truth for code points. Where it and this README disagree, the
YAML wins.

---

## The layering, stated once

Two things are deliberately separate, and conflating them is the most likely way
to get this wrong.

**An extension profile is open, and it is a mechanism.** It defines code points,
a payload encoding, and required behavior, so that two implementations from
different vendors interoperate. Profiles in the Core range carry the base
specification's CC BY 4.0 license and are free to implement. They exist to be
adopted, including by competitors.

**A Strynex overlay is a product, and it is branded.** It is one implementation
of one or more profiles, plus the parts the specification deliberately leaves
implementation-defined (section 6.5 fusion weighting above all), plus a name
that is a trademark and is not licensed by CC BY 4.0 or Apache 2.0.

So Cadence is the Strynex product; EP-0003 Signal Timing Advisory is the open
mechanism it rides on. Anyone may implement EP-0003. Nobody else may call the
result Cadence. That split is the entire commercial architecture, and it is why
publishing the profiles openly costs nothing.

The map below therefore has two columns that look similar and are not: what an
overlay **requires** (open, allocated here) and what it **is** (branded, not
specified here).

---

## Profile index

Status values: **Written**, the profile document exists in this directory.
**Allocated**, code points are reserved and the document is not written.
**Withheld**, deliberately not allocated, with a reason.

| ID | Profile | Range | Status | Notes |
| --- | --- | --- | --- | --- |
| `0x0000` | [Extension Profile Framework](EP-0000-extension-profile-framework.md) | n/a | **Written** | The mechanism. Not itself a profile. |
| `0x0001` | [Intersection Assist](EP-0001-intersection-assist.md) | Core | **Written** | Owed by base 12.3. Required by roadmap v0.4. |
| `0x0002` | [Merge Negotiation](EP-0002-merge-negotiation.md) | Core | **Written** | `MERGE` exists in A.3 with no protocol. Base gap. |
| `0x0003` | Signal Timing Advisory | Core | Allocated | Vehicle and infrastructure sides of SPaT-paced approach. |
| `0x0004` | Cooperative Positioning | Core | Allocated | GNSS-denied. Base 13.4 requires the case and specifies nothing. |
| `0x0005` | Priority Vehicle Authorization | Core | Allocated | Security-critical. See warning below. |
| `0x0006` | Vulnerable Road User Extended | Core | Allocated | Blocked on errata E-03. See warning below. |
| `0x0007` | Work Zone Protection | Core | Allocated | Base 15.3 describes the scenario, not the messages. |
| `0x0008` | Grade Crossing | Core | Allocated | Rail. |
| `0x0009` | School Transport | Core | Allocated | Stop-arm and school-zone envelope. |
| `0x000A` | Responder Scene Protection | Core | Allocated | Moving cordon around crews on the shoulder. |
| `0x000B` | Wrong-Way Detection | Core | Allocated | Pose and heading against MAP topology. |
| `0x000C` | Traffic Flow Damping | Core | Allocated | Stop-and-go shockwave absorption. |
| `0x000D` | Network Flow Redistribution | Core | Allocated | Corridor-scale pre-diversion. |
| `0x000E` | Low-Speed Cooperative Maneuvering | Core | Allocated | Yards, depots, garages. |
| `0x000F` | Road Surface State Extended | Core | Allocated | Beyond the base `ROAD_SURFACE_FRICTION` kind. |
| `0x0010` | Degraded Sensing Continuity | Core | Allocated | Holding the world model when local sensors drop. |
| `0x0011` | Extended Hazard Reporting | Core | Allocated | Beyond the base `HAZARD_FLAG` kind. |
| `0x1000` to `0x10FF` | Specter Point Intelligence, LLC | Vendor | Allocated | 256-identifier block. Privately administered. |

Two profiles carry standing warnings and MUST NOT be written up as ordinary
work items:

- **EP-0005 Priority Vehicle Authorization.** A priority claim any participant
  can assert is a corridor denial-of-service primitive. The authorization model
  has to bind to enrollment credentials, and section 10.5 restricts exactly that
  linkage. This is a security design problem before it is a profile.
- **EP-0006 Vulnerable Road User Extended.** Blocked on erratum
  [E-03](../docs/errata-v0.2.md#e-03). Until the base specification resolves the
  arithmetic that caps a VRU beacon track at trust weight 0.249 against an
  advisory threshold of 0.5, a VRU profile can define messages that no conformant
  receiver is permitted to act on.

---

## Strynex overlay map

All 26 overlays from the Strynex catalog, and what each requires. **"Base only"
means the overlay needs no new code points**: it is a product built on v0.2 as
written, or on parts of it the specification leaves implementation-defined.

That column is the useful one. Ten of the 26 need nothing from this registry at
all, which is a stronger statement about the base message set than any amount of
new specification would be.

### Strynex Drive

| Overlay | Requires | Why |
| --- | --- | --- |
| Slipstream | Base only | C-ACC Group Protocol, base 12.1, plus the 12.2 overlay. |
| Cadence | `0x0003` | Paces approach speed against SPaT phase. |
| Parallax | Base only | Trust-weighted fusion of peer `PerceptionSummary`, base 11. |
| Sightline | `0x0010` | Needs sensor-health-weighted fallback the base leaves undefined. |
| Interlace | `0x0002` | `MERGE` intent exists; the negotiation does not. |
| Breakwater | `0x000C` | Gap modulation to absorb a shockwave. |
| Wingman | `0x0006` | Two-wheeler conspicuity and left-turn conflict. Blocked, see E-03. |
| Cairn | `0x0004` | Position from peers and RSUs without GNSS. |
| Riptide | `0x000B` | Heading against MAP. |
| Flotsam | Base + payload | `HAZARD_FLAG` kind exists; its payload is undefined. Base v0.3 work, not a profile. Extended taxonomy is `0x0011`. |

### Strynex Civic

| Overlay | Requires | Why |
| --- | --- | --- |
| Metronome | `0x0003` | Infrastructure side of the same profile as Cadence. |
| Clearway | `0x0005` | Preemption and pre-diversion. Security-critical. |
| Cordon | `0x000A` | Moving protective envelope around responders. |
| Crossguard | `0x0009` | Stop-arm state and school-zone envelope. |
| Crossbuck | `0x0008` | Train presence and gate state. |
| Flagger | `0x0007` | Base 15.3 gives the scenario; the MAP delta and worker tracks need encoding. |
| Braid | `0x000D` | Corridor-scale flow splitting. |
| Groundtruth | Base + payload | `ROAD_SURFACE_FRICTION` and `WEATHER_ESTIMATE` kinds exist; payloads undefined. Extended states are `0x000F`. |

### Strynex Fleet

| Overlay | Requires | Why |
| --- | --- | --- |
| Slipstream Haul | Base only | Base 12.1 and the 15.2 scenario at freight scale. |
| Dispatch | Base only | Consumes `Attestation` and the V-DIAG records of 5.3. Off-wire product. |
| Marshal | `0x000E` | Low-speed maneuvering in yards and structures. |
| Affidavit | Base only | V-DIAG records, 5.3. Off-wire. Privacy review required before scoping. |

### Strynex Assurance

| Overlay | Requires | Why |
| --- | --- | --- |
| Assay | Base only | Base 6.5 states the weighting function is implementation-defined. That sentence is the product. Nothing to allocate. |
| Veil | Base only | Base 10 and the 16.3 privacy scan, implemented. Nothing to allocate. |
| Hallmark | Base only | A certification programme against 16.3, not a wire behavior. |
| Nightwatch | **Withheld** | Not allocatable. Base 10.3 caps retention at 60 seconds and 10.5 prohibits movement traces. No conformant version exists. Recorded so the question stays visible, not so it gets built. |

---

## Code point allocations

Partitions are defined in [EP-0000 section 4](EP-0000-extension-profile-framework.md#4-enum-code-point-ranges).
Blocks are permanent and are never reused, including after withdrawal.

### IntentCode

v0.2 assigns 0 to 11 (`INTENT_UNKNOWN` through `PLATOON_LEAVE`); 12 to 63 are
reserved for the base specification. Profile blocks:

| Block | Profile |
| --- | --- |
| 64 to 71 | `0x0002` Merge Negotiation |
| 72 to 79 | `0x0005` Priority Vehicle Authorization |
| 80 to 87 | `0x000E` Low-Speed Cooperative Maneuvering |
| 88 to 95 | `0x0001` Intersection Assist |
| 96 to 103 | `0x0009` School Transport |
| 104 to 111 | `0x000C` Traffic Flow Damping |
| 112 to 119 | `0x0004` Cooperative Positioning |
| 120 to 127 | `0x000A` Responder Scene Protection |
| 128 to 135 | `0x0003` Signal Timing Advisory |
| 136 to 191 | Unallocated (7 blocks) |

192 to 254 are vendor private. 255 is `INTENT_PROFILE_EXTENSION`.

### TelemetryKind

v0.2 assigns 0 to 4 (`TELEMETRY_UNKNOWN` through `HAZARD_FLAG`); 5 to 31 are
reserved for the base specification. Profile blocks:

| Block | Profile |
| --- | --- |
| 32 to 39 | `0x000F` Road Surface State Extended |
| 40 to 47 | `0x0011` Extended Hazard Reporting |
| 48 to 55 | `0x0008` Grade Crossing |
| 56 to 63 | `0x000D` Network Flow Redistribution |
| 64 to 71 | `0x0007` Work Zone Protection |
| 72 to 79 | `0x000A` Responder Scene Protection |
| 80 to 87 | `0x000B` Wrong-Way Detection |
| 88 to 127 | Unallocated (5 blocks) |

128 to 254 are vendor private. 255 is `TELEMETRY_PROFILE_EXTENSION`.

### Object class ID

Base 7.2 assigns `0x00` to `0x0A` and defines `0xFF` as `VENDOR_EXTENSION`.
`0x0B` to `0x3F` are reserved for the base specification. Profile blocks:

| Block | Profile |
| --- | --- |
| `0x40` to `0x47` | `0x0006` Vulnerable Road User Extended |
| `0x48` to `0x4F` | `0x0007` Work Zone Protection |
| `0x50` to `0x57` | `0x000A` Responder Scene Protection |
| `0x58` to `0xBF` | Unallocated (26 blocks) |

`0xC0` to `0xFE` are vendor private. `0xFF` is unchanged from base 7.2.

---

## Why the allocations exist before the profiles

Fifteen of the seventeen Core profiles above are allocated and unwritten. That
is deliberate and it is the point of this directory.

Roadmap v1.0 freezes the wire format and transfers governance under base 18.4.
Code point ranges not reserved before the freeze cannot be reserved afterward
without a new wire version, and after the transfer the allocation decision
belongs to a committee rather than to the editor of record. Reserving a range
costs a line in a file today. Recovering it later costs a standards cycle, or is
simply impossible.

Writing a profile is different work with a different deadline. A profile can be
written at any time, by anyone, including after the transfer. It does not need
to happen now.

So: allocate everything now, write profiles when they are needed. Two are
written because the base specification already owes them.

## Contributing a profile

See [EP-0000 section 8](EP-0000-extension-profile-framework.md#8-registry-operation)
for the allocation procedure and
[section 10](EP-0000-extension-profile-framework.md#10-profile-document-template)
for the document template.

Core and Registered allocations are free and are granted to anyone. Registered
allocations receive no technical review, which is intentional: the constraints
of EP-0000 section 6 are what protect the base specification, and they bind
whether or not anyone reviewed the profile.

## License

Profile documents in the Core range are specification text and are licensed
CC BY 4.0, matching [`../spec/LICENSE`](../spec/LICENSE). `registry.yaml` is
Apache 2.0, matching [`../LICENSE`](../LICENSE).

Overlay names appearing in the map above are trademarks of Specter Point
Intelligence, LLC and are not licensed by either. Implementing any profile
listed here requires no permission and conveys no right to any overlay name.
