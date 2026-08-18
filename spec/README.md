# O-CPS v0.2 Specification

Converted verbatim from `reference/OCPS_Whitepaper_v0.2.docx` (Draft v0.2, April 2026).
The unsplit conversion is preserved at [`OCPS_v0.2_full.md`](OCPS_v0.2_full.md);
the per-section files below are slices of that same text, split at level-1 headings.

## Licensing of this directory

    Open Cooperative Perception System (O-CPS), Draft v0.2
    Copyright (c) 2026 Archon C. Locke, Specter Point Intelligence, LLC
    SPDX-License-Identifier: CC-BY-4.0

The specification text in this `spec/` directory, and the source documents in
`../reference/`, are licensed under the Creative Commons Attribution 4.0
International License. The unmodified legalcode is at [`LICENSE`](LICENSE).

Everything outside `spec/` and `reference/`, including the protobuf schemas in
`../schemas/`, the conformance test scaffolding in `../tests/`, and any reference
implementation code, is licensed under Apache License 2.0. See
[`../LICENSE`](../LICENSE).

## Contents

- [Abstract](00-abstract.md)
- [1. Introduction](01-introduction.md)
- [2. Terminology and Conventions](02-terminology-and-conventions.md)
- [3. Scope and Goals](03-scope-and-goals.md)
- [4. Design Philosophy](04-design-philosophy.md)
- [5. System Architecture Overview](05-system-architecture-overview.md)
- [6. Layered Stack Detail](06-layered-stack-detail.md)
- [7. Normalized Perception Profile](07-normalized-perception-profile.md)
- [8. Message Layer: Message Types and Wire Format](08-message-layer.md)
- [9. Trust and Security Model](09-trust-and-security-model.md)
- [10. Privacy Model](10-privacy-model.md)
- [11. Fusion and World Model](11-fusion-and-world-model.md)
- [12. Cooperative Behaviors](12-cooperative-behaviors.md)
- [13. Vehicle Integration Interfaces](13-vehicle-integration-interfaces.md)
- [14. Safety Architecture](14-safety-architecture.md)
- [15. Deployment Scenarios](15-deployment-scenarios.md)
- [16. Reference Implementation and Conformance](16-reference-implementation-and-conformance.md)
- [17. Standards Alignment and Gap Analysis](17-standards-alignment-and-gap-analysis.md)
- [18. Governance, Intellectual Property, and Licensing](18-governance-ip-and-licensing.md)
- [19. Roadmap](19-roadmap.md)
- [20. Conclusion](20-conclusion.md)
- [Appendix A. Complete Message Schemas](21-appendix-a-message-schemas.md)
- [Appendix B. Glossary of Acronyms](22-appendix-b-glossary.md)
- [Appendix C. Normative References](23-appendix-c-normative-references.md)

## Conversion notes

- Conversion was performed with a `python-docx` reader that maps Word Heading 1/2/3 to
  Markdown `#`/`##`/`###`, Consolas runs to fenced code blocks (indentation preserved),
  and Word tables to GitHub-flavored Markdown tables.
- A token-level diff against the source document shows zero dropped words. All 12 tables,
  all 10 code blocks, and all RFC 2119 keyword occurrences (92 MUST, 8 SHOULD, 15 MAY,
  5 SHALL, 1 REQUIRED, 1 RECOMMENDED, 2 OPTIONAL) survive intact.
- The Word auto-generated Table of Contents field had no stored text and could not be
  extracted; the index above is regenerated from the actual headings.
- See [`../docs/conversion-notes.md`](../docs/conversion-notes.md) for discrepancies observed
  in the source document that were deliberately **not** corrected.
