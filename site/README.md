# site/

A single self-contained page presenting the Strynex overlay catalog:
`index.html`, no build step, no external requests, works from `file://`.

## GitHub Pages is deliberately not enabled

Do not enable it without deciding to publish this content to the open internet,
because that is what enabling it does.

**A GitHub Pages site does not inherit the visibility of its repository.** Pages
serves publicly regardless of whether the source repo is private. Access control,
meaning a Pages site restricted to people with repo read access, is available only
on GitHub Enterprise Cloud. It does not exist on Free, Pro, or Team. Publishing
Pages from a private repository at all requires Pro or higher; a Free account can
publish Pages only from public repositories.

This repository is private and owned by a personal account. Enabling Pages here
either fails, on Free, or succeeds and puts this page on the public internet, on
Pro. There is no configuration that produces a private site.

## Why that matters for this page specifically

The catalog references material that is not ready for publication:

- **Erratum E-03**, an unadopted safety defect in a specification positioned for
  external adoption. Publishing an unfixed safety defect is a coordinated
  disclosure decision, not a hosting decision. See
  [`../docs/errata-v0.2.md`](../docs/errata-v0.2.md) and repository issue #5.
- The commercial architecture: which layers are given away and which are retained.
- Extension profile code point allocations that are proposed and not adopted.

The page carries `<meta name="robots" content="noindex, nofollow">` as a second
line of defence. That is a request to well-behaved crawlers and nothing more. It
is not access control and it does not make publication safe.

## Why `site/` and not `docs/`

GitHub Pages can publish from three sources: the repository root, a `/docs`
directory, or a `gh-pages` branch. `/docs` in this repository holds
`errata-v0.2.md`, `open-questions.md`, and `conversion-notes.md`. Enabling Pages
from `/docs` would publish all of them, including the errata register.

`site/` is not a Pages source, so no Pages configuration reaches this directory by
accident. Serving it requires an explicit workflow that names it.

## If you decide to publish

Make the decision first, then pick a route:

- **Move the page to a separate public repository.** Cleanest. The private repo
  stays private, and only what you chose to publish is published.
- **Make this repository public.** Consistent with the standards posture stated in
  §18, but it publishes the errata register at the same time, so E-03 should be
  adopted first.
- **Enable Pages on a Pro account and accept a public site.** Same disclosure as
  above, with the added oddity of a public site over a private repo, which tends
  to surprise people later.

## Regenerating

`index.html` is generated from the Claude artifact source by wrapping the fragment
in a document scaffold and a minimal CSS reset. The artifact runtime supplies both;
a plain web server does not. Everything else, including all styling, is inline and
unchanged.
