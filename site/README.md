# site/

A single self-contained page presenting the Strynex overlay catalog:
`index.html`, no build step, no external requests, works from `file://`.

The repository README now carries the same content in markdown, which is what a
visitor reads on landing. This page is the designed version of it: the stack
diagram, the 26 overlay cards, and the code point tables laid out rather than
listed.

## GitHub Pages is not enabled

The repository is public, so enabling Pages would work and would put this page on
the open internet at a `github.io` address. That is a publishing decision rather
than a hosting detail, and nobody has made it yet.

What it would take: Pages publishes from the repository root, a `/docs`
directory, or a `gh-pages` branch. `site/` is none of those, so serving this page
requires a workflow that names it explicitly. Publishing from `/docs` instead
would serve `errata-v0.2.md`, `open-questions.md`, and `conversion-notes.md` as a
site, which is a different set of pages than the one intended here.

The earlier version of this file argued against Pages on the grounds that the
repository was private and a Pages site does not inherit repository visibility.
That was true and is now moot. Everything the page describes is in the public
tree: the errata register including E-03, the commercial architecture, and the
proposed code point allocations.

## What the page asserts

Status is stated on the page itself and repeated here because it is easy to read
a designed page as settled:

- Nothing on it is adopted specification text. `spec/` remains a verbatim
  conversion of the April 2026 whitepaper.
- Two extension profiles are written, fifteen are allocated and unwritten, one is
  deliberately withheld, and ten overlays need no allocation at all.
- E-03 is disclosed with its proposed resolution and is not fixed. See
  [`../docs/errata-v0.2.md`](../docs/errata-v0.2.md#e-03).

## Regenerating

`index.html` is generated from the Claude artifact source by wrapping the
fragment in a document scaffold and a minimal CSS reset. The artifact runtime
supplies both; a plain web server does not. Everything else, including all
styling, is inline and unchanged.

Two edits have been made to the generated file since, and a regeneration has to
repeat them:

1. The `noindex, nofollow` robots meta was removed. It existed to limit exposure
   while the repository was private.
2. The closing "Confidentiality" note, which said the page was not for public
   distribution ahead of a disclosure decision, was replaced by a "Disclosure"
   note recording that the decision was made.
