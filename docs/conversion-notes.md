# Conversion Notes

How `reference/OCPS_Whitepaper_v0.2.docx` became `spec/`, what was verified, and
what was deliberately left alone.

## Method

`pandoc` was tried first and produced correct tables and prose but did not map
Word's Heading 1/2/3 styles to Markdown headings, which made a clean per-section
split impossible. It also flattened the Consolas-formatted protobuf listings into
ordinary paragraphs with escaped underscores, which would have corrupted the
schema extraction.

The conversion was therefore done with a purpose-built `python-docx` reader that
walks the document body in order and maps:

- Word `Heading1` / `Heading2` / `Heading3` to `#` / `##` / `###`
- Consolas runs to fenced code blocks, with `<w:br/>` as line breaks and leading
  indentation preserved byte-for-byte
- Word tables to GitHub-flavored Markdown tables, first row as header, pipes
  escaped in cell content
- `ListParagraph` and numbered paragraphs to Markdown list items, indented by
  `w:ilvl`
- bold and italic runs to `**` and `*`

Prose was not touched. No paraphrasing, no reflowing, no normalization of
terminology or punctuation.

## Verification

A token-level diff was run between every `<w:t>` node in the source document and
the generated Markdown:

- **Source: 9,981 word tokens. Output: 10,001.** Zero tokens missing.
- The 20 extra tokens are entirely accounted for: the code-fence language tags
  added by the converter (`protobuf` x7, `text` x3) and the ten-word HTML comment
  standing in for the Word Table of Contents field.
- **All 12 tables** survived with exact row counts (11, 7, 14, 5, 6, 7, 6, 9, 4,
  12, 6, 39 rows respectively, matching the source).
- **All 10 code blocks** survived with indentation intact.
- **RFC 2119 keyword counts are identical**: MUST 92, SHOULD 8, MAY 15, SHALL 5,
  REQUIRED 1, RECOMMENDED 1, OPTIONAL 2.
- **All 112 headings** (25 H1, 76 H2, 11 H3) are present.

## What did not convert

**The Table of Contents.** Word stores an auto-generated TOC as a field with no
cached text in this document, so there was nothing to extract. `spec/README.md`
carries a regenerated index built from the actual headings. The
`# Table of Contents` heading is preserved in `spec/OCPS_v0.2_full.md` with an
HTML comment noting the substitution.

**Nothing else.** No content was lost.

## Discrepancies observed in the source, NOT corrected

These are errors or inconsistencies in the whitepaper itself. They were left
exactly as written, because silently correcting a specification is worse than
carrying a known defect. Each is a candidate erratum for v0.3.

1. **§16.3 cites "the state machine of §13.1.1"** for C-ACC Group formation,
   handoff, and dissolution. In v0.2 the C-ACC state machine is at **§12.1.1**;
   §13 is Vehicle Integration Interfaces. The cross-reference appears to be stale
   from an earlier section numbering.

2. **§16.3 cites "the prohibited fields of §11.1"** for privacy conformance. The
   Privacy Model is **§10**, and §10.1 is Data Minimization on the Wire. §11.1 is
   Fusion Inputs.

3. **§7.3 cites "consensus comparison (§10.3)"** for detecting systematic
   confidence miscalibration. §10.3 is Retention Limits, under the Privacy Model.
   The intended target is most likely §9.3 (Misbehavior Detection) or §11.x
   (Fusion), but the document does not make this recoverable.

4. **§1.2 forward-references Slipstream Mode as "§13"**, but Slipstream Mode is
   **§12.2**. Same off-by-one-section pattern as item 1.

5. **The same off-by-one pattern recurs in at least five more places**, all
   pointing one section too high, all left as written: §8.7 cites "the privacy
   model of §11" (Privacy Model is §10); §18.5 cites "the conformance-class
   framework of §17" (§16); §3.3 and §8.6 both cite "Class A, §17.1" (§16.1);
   §3.2 cites driver authorization at "§13.1.3" (§12.1.3); §9.3 cites
   "trust-weight de-rating (§12.3)" (§11.3).

The consistent +1 offset across all nine suggests a section was removed from an
earlier draft without the cross-references being renumbered. That is a single
erratum, not nine, and it should be fixed once at the source rather than patched
per-citation.

## Corrections to prior mischaracterizations

These are **not** defects in the whitepaper. The whitepaper is right; earlier
descriptions of it were wrong. Recorded so the wrong version does not get
reintroduced.

1. **§16.3 enumerates seven test categories, not five**: wire-format,
   cryptographic, timing, behavioral, safety, privacy, and misbehavior. `tests/`
   is scaffolded for all seven.

2. **Conformance classes are A, B, and I.** There is no Class C. Class I
   (Infrastructure) is the class carrying the extended horizon of up to 500 m.

## Notes on the protobuf extraction

The files under `schemas/proto/ocps/v2/` reproduce the Appendix A message and
field definitions verbatim. Four kinds of change were made, all mechanical and
all disclosed in each file header:

1. `syntax`, `package ocps.v2`, `option`, and `import` statements were added or
   relocated so the files compile independently. Appendix A declares `syntax` and
   `package` once, inside A.1.
2. Definitions were split across six files by message group, one per Appendix A
   subsection.
3. Unit and constraint comments were added where the whitepaper states the unit
   or constraint in prose but not in the schema listing. Every added comment is
   tagged `[spec §N]` citing the section that states it. Every untagged comment
   is reproduced verbatim from the Appendix A listing. Where the whitepaper does
   not state a unit at all (for example `pitch_deg` and `roll_deg`, which §7.1
   covers only for yaw), no comment was added rather than guessing from the field
   name.
4. Nothing else. No fields were added, removed, renamed, or renumbered.

**Verification:** all six files compile cleanly under `libprotoc 35.1` (invoked
via `grpcio-tools`), with `-I schemas/proto` and no errors or warnings.

### A field-set mischaracterization worth flagging

Some prior notes describe the object container as carrying `object_id`,
`class_probs`, position in **ENU meters**, velocity, orientation, and a **6x6
covariance** matrix. **The v0.2 whitepaper specifies none of those.** What
Appendix A.2 actually defines for `PerceivedObject` is:

`class_id`, `confidence` (0-255), `range_m`, `bearing_deg` (relative to
transmitter heading), `dims`, `rel_v_long_mps`, `rel_v_lat_mps`, `sensor_mask`,
and `vendor_oid`.

Objects are expressed in **range and bearing relative to the transmitter**, not
in an ENU frame. §7.1 mandates **WGS-84** for all pose and kinematic data and
states that deviation from WGS-84 is a conformance failure. There is no
per-object identifier, no class probability vector, no orientation field, and no
covariance of any dimension anywhere in the v0.2 message set.

Those fields were **not** added. If they are wanted, they are a v0.3 spec change
and need to be argued for in the specification first, not smuggled in through the
schema. Recorded here so the gap is visible rather than quietly resolved in
either direction.
